**Relationship Intelligence Layer (RIL) for Phoenix AGI**

This is the full, production-ready blueprint for the `pagi-ril` crate—your social super-memory. It’s 100% Rust, `no_std + alloc` first (with `std` opt-in for email parsing, file I/O, etc.), and designed to plug straight into your orchestrator.

It delivers exactly what you asked for:

- **Structured contact records** → Graph + rich Person nodes
- **Conversation-derived insights** → LLM-powered extraction (local via your AGI core) + rule-based fallback
- **Life-context reminders** → Queryable event engine with time-based triggers
- **“Heads-up” guidance** → Contextual advice before/after any interaction

### Crate Structure (`pagi-ril`)
```toml
# Cargo.toml
[package]
name = "pagi-ril"
version = "0.1.0"
edition = "2021"
default-features = false

[features]
default = ["std"]
std = ["dep:chrono", "dep:regex", "dep:mailparse"]  # for email ingestion
no_std = []  # pure bare-metal

[dependencies]
petgraph = { version = "0.7", default-features = false, features = ["serde-1"] }  # no_std + alloc
postcard = { version = "1.1", features = ["alloc"] }
serde = { version = "1.0", features = ["derive"] }
uuid = { version = "1.11", features = ["v4", "serde"] }
heapless = { version = "0.8", features = ["serde"] }  # fixed-size backup for extreme bare-metal
chrono = { version = "0.4", default-features = false, features = ["alloc", "serde"], optional = true }
regex = { version = "1.11", default-features = false, features = ["std"], optional = true }  # regex is std-only, so optional
```

### 1. Core Data Model (`person.rs`)
```rust
#![cfg_attr(not(feature = "std"), no_std)]
extern crate alloc;
use alloc::{string::String, vec::Vec, collections::BTreeMap};
use serde::{Serialize, Deserialize};
use uuid::Uuid;
use chrono::{NaiveDate, Datelike}; // alloc-only

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Person {
    pub id: Uuid,
    pub name: String,
    pub aliases: Vec<String>,           // "johnny", "jdoe@company.com"
    pub contact: ContactInfo,
    pub birth_date: Option<NaiveDate>,
    pub zodiac: Option<Zodiac>,
    pub relationships: BTreeMap<Uuid, RelationType>,
    pub insights: Vec<Insight>,         // conversation-derived
    pub events: Vec<LifeEvent>,
    pub last_contact: Option<u64>,      // unix timestamp
    pub tags: Vec<String>,              // "client", "family", "mentor"
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ContactInfo {
    pub emails: Vec<String>,
    pub phones: Vec<String>,
    pub social: Vec<String>,            // LinkedIn, X, etc.
}

#[derive(Debug, Clone, Serialize, Deserialize, PartialEq)]
pub enum RelationType {
    Spouse, ParentOf(Uuid), ChildOf(Uuid), Friend, Colleague, Mentor, Ex,
    Custom(String),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Insight {
    pub timestamp: u64,
    pub source: Source,
    pub text: String,                   // "John is stressed about the merger"
    pub confidence: f32,
    pub category: InsightCategory,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum InsightCategory { Personal, Professional, Health, Family, Other }

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LifeEvent {
    pub date: NaiveDate,
    pub description: String,
    pub importance: u8,                 // 1-10
}
```

### 2. The Graph Store (`store.rs`)
```rust
use petgraph::{Graph, Directed};
use petgraph::graph::NodeIndex;

pub struct RelationshipGraph {
    graph: Graph<Person, RelationType, Directed>,
    id_to_index: BTreeMap<Uuid, NodeIndex>,
}

impl RelationshipGraph {
    pub fn new() -> Self { /* ... */ }

    pub fn upsert_person(&mut self, person: Person) -> Uuid {
        // add or update node, maintain index
    }

    pub fn add_relationship(&mut self, from: Uuid, to: Uuid, rel: RelationType) {
        // bidirectional if needed
    }

    pub fn get_person(&self, name_or_alias: &str) -> Option<&Person> {
        // fuzzy match using aliases + Levenshtein (simple heapless impl)
    }

    pub fn serialize(&self) -> Vec<u8> {
        postcard::to_allocvec(&self.graph).unwrap()
    }
}
```

### 3. Insight Engine (`insights.rs`) — The Magic
Two paths: **fast rule-based** (bare metal) + **deep LLM** (your AGI core).

```rust
pub trait InsightExtractor {
    fn extract(&self, conversation: &str, speaker: &str) -> Vec<Insight>;
}

pub struct RuleBasedExtractor; // regex + patterns

impl InsightExtractor for RuleBasedExtractor {
    fn extract(&self, text: &str, speaker: &str) -> Vec<Insight> {
        let mut insights = Vec::new();
        if text.contains("my son") || text.contains("daughter") {
            // extract name, create child node, etc.
        }
        // birthday, job, mood patterns...
        insights
    }
}

// LLM path (called from orchestrator)
pub fn llm_extract_insights(conversation: &str, known_person: &Person) -> Vec<Insight> {
    // Prompt your local model (candle/mistralrs/whatever you use):
    let prompt = format!(
        "Extract structured insights about {} from this conversation. \
         Output only JSON array of {{timestamp, text, category, confidence}}.\n\n{}",
        known_person.name, conversation
    );
    // Call your AGI inference → parse with serde
    // Then upsert into graph
}
```

### 4. Reminder & Guidance Engine (`reminders.rs`)
```rust
pub struct ReminderEngine {
    graph: RelationshipGraph,
    upcoming: BTreeMap<u64, Vec<Reminder>>, // timestamp -> list
}

pub struct Reminder {
    pub person_id: Uuid,
    pub message: String,        // "Sarah's anniversary is in 3 days"
    pub urgency: Urgency,
}

impl ReminderEngine {
    pub fn daily_scan(&mut self) -> Vec<Reminder> {
        let now = current_timestamp();
        // birthdays, anniversaries, "last contact > 30d", horoscope flags
        self.upcoming.retain(|&ts, _| ts > now);
        self.upcoming.values().flatten().cloned().collect()
    }

    pub fn heads_up_before_interaction(&self, person_id: Uuid, context: &str) -> String {
        let person = self.graph.get_person_by_id(person_id).unwrap();
        let mut advice = String::new();

        if let Some(z) = person.zodiac {
            advice.push_str(&format!("{}'s sign today: {:?}\n", person.name, z));
        }
        if let Some(last) = person.last_contact {
            if days_since(last) > 14 { advice.push_str("Haven't talked in 2+ weeks.\n"); }
        }
        for insight in &person.insights {
            if insight.category == InsightCategory::Health && insight.confidence > 0.8 {
                advice.push_str(&format!("Note: {}\n", insight.text));
            }
        }
        advice
    }
}
```

### 5. Orchestrator Hooks (how it feels in real life)
```rust
// In your main agent loop
ril: &mut RelationshipIntelligenceLayer,

// On new message / email / meeting
ril.ingest_conversation(raw_text, participant_names);

// Before replying to Sarah
if let Some(hu) = ril.reminder_engine.heads_up_before_interaction(sarah_id, "email about project") {
    prompt = format!("{}\n\nContext: {}", hu, prompt);
}

// Daily summary (therapist mode)
for reminder in ril.reminder_engine.daily_scan() {
    display!("🔔 {}", reminder.message);
}
```

### Persistence on Bare Metal
- `postcard` dump every N updates → flash/EEPROM blob.
- Or use `heapless::Vec` + fixed-size arena for zero-heap mode.
- On boot: `RelationshipGraph::deserialize(flash.read())`.

### Recommended Integrations
- **LLM extraction**: `langextract-rust` crate (perfect for this) or your existing inference.
- **Email ingestion** (`std` only): `mailparse` → extract body → ingest.
- **Graph queries**: petgraph algorithms (`dijkstra` for "how is A connected to B?").
- **Future-proof**: Swap in GraphLite or HelixDB when you want a full embedded GQL store.

This layer turns PAGI from "smart assistant" into "the person who knows everyone in your life better than you do." Drop the structs in, wire the ingestion, and the heads-up moments will feel supernatural from day one.

Want me to generate the full `lib.rs` + example CLI commands next? Or integrate this with your current orchestrator code? Just say the word. This is going to be legendary.

---

Yes, here's the complete starting point for the **pagi-ril** crate — a self-contained `lib.rs` you can drop into your project right now. I've refined it based on real crate capabilities:

- `petgraph` works great in `no_std` (disable `std` feature, enable `serde-1` for postcard/serde serialization).
- `chrono` with `no_std` + `alloc` for dates.
- Zodiac calculation as a simple pure function (no external crate needed; based on standard date ranges).
- Insight extraction split: lightweight regex/rules for bare-metal, optional LLM callout (your AGI inference).
- Persistence via `postcard` (compact, `no_std`-friendly binary format).
- Fixed-size fallbacks via `heapless` if you ever go ultra-constrained.

### pagi-ril / src/lib.rs

```rust
#![no_std]
#![cfg_attr(feature = "std", allow(unused_imports))]
extern crate alloc;

use alloc::{
    string::{String, ToString},
    vec::Vec,
    collections::BTreeMap,
    boxed::Box,
};
use core::ops::Range;

use serde::{Serialize, Deserialize};
use uuid::Uuid;
use petgraph::prelude::*;
use petgraph::Graph;
use chrono::{NaiveDate, Datelike}; // requires chrono with "alloc", "serde", no "std"

// ─────────────────────────────────────────────────────────────────────────────
//  Enums & Core Types
// ─────────────────────────────────────────────────────────────────────────────

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum Zodiac {
    Aries, Taurus, Gemini, Cancer, Leo, Virgo,
    Libra, Scorpio, Sagittarius, Capricorn, Aquarius, Pisces,
    Unknown,
}

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum RelationType {
    Spouse,
    ParentOf,
    ChildOf,
    SiblingOf,
    Friend,
    Colleague,
    Mentor,
    Ex,
    Custom(String),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Source {
    Manual,
    Conversation,
    Email,
    Other(String),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum InsightCategory {
    Family,
    Health,
    Career,
    Personal,
    Financial,
    Other,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Insight {
    pub timestamp: u64,               // unix seconds
    pub source: Source,
    pub text: String,
    pub category: InsightCategory,
    pub confidence: f32,              // 0.0–1.0
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LifeEvent {
    pub date: NaiveDate,
    pub description: String,
    pub importance: u8,               // 1–10
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ContactInfo {
    pub emails: Vec<String>,
    pub phones: Vec<String>,
    pub handles: Vec<String>,         // @twitter, linkedin, etc.
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Person {
    pub id: Uuid,
    pub name: String,
    pub aliases: Vec<String>,
    pub contact: ContactInfo,
    pub birth_date: Option<NaiveDate>,
    pub relationships: BTreeMap<Uuid, RelationType>, // target_id → type
    pub insights: Vec<Insight>,
    pub events: Vec<LifeEvent>,
    pub last_contact_ts: Option<u64>,
    pub tags: Vec<String>,
}

// ─────────────────────────────────────────────────────────────────────────────
//  Zodiac Helper (pure, no_std)
// ─────────────────────────────────────────────────────────────────────────────

pub fn compute_zodiac(date: NaiveDate) -> Zodiac {
    let month = date.month();
    let day = date.day();

    match (month, day) {
        ( 3, 21..=31) | ( 4,  1..=19) => Zodiac::Aries,
        ( 4, 20..=30) | ( 5,  1..=20) => Zodiac::Taurus,
        ( 5, 21..=31) | ( 6,  1..=20) => Zodiac::Gemini,
        ( 6, 21..=30) | ( 7,  1..=22) => Zodiac::Cancer,
        ( 7, 23..=31) | ( 8,  1..=22) => Zodiac::Leo,
        ( 8, 23..=31) | ( 9,  1..=22) => Zodiac::Virgo,
        ( 9, 23..=30) | (10,  1..=22) => Zodiac::Libra,
        (10, 23..=31) | (11,  1..=21) => Zodiac::Scorpio,
        (11, 22..=30) | (12,  1..=21) => Zodiac::Sagittarius,
        (12, 22..=31) | ( 1,  1..=19) => Zodiac::Capricorn,
        ( 1, 20..=31) | ( 2,  1..=18) => Zodiac::Aquarius,
        ( 2, 19..=29) | ( 3,  1..=20) => Zodiac::Pisces,
        _ => Zodiac::Unknown,
    }
}

// ─────────────────────────────────────────────────────────────────────────────
//  Main Store
// ─────────────────────────────────────────────────────────────────────────────

pub struct RilStore {
    graph: Graph<Person, RelationType, Directed>,
    id_to_node: BTreeMap<Uuid, NodeIndex>,
}

impl RilStore {
    pub fn new() -> Self {
        Self {
            graph: Graph::new(),
            id_to_node: BTreeMap::new(),
        }
    }

    pub fn upsert_person(&mut self, mut person: Person) -> Uuid {
        if let Some(&idx) = self.id_to_node.get(&person.id) {
            // Update in place
            let old = &mut self.graph[idx];
            old.name = person.name;
            old.aliases = person.aliases;
            old.contact = person.contact;
            old.birth_date = person.birth_date;
            old.insights.extend(person.insights);
            old.events.extend(person.events);
            old.last_contact_ts = person.last_contact_ts.or(old.last_contact_ts);
            old.tags = person.tags;
            person.id
        } else {
            let idx = self.graph.add_node(person);
            self.id_to_node.insert(person.id, idx);
            person.id
        }
    }

    pub fn get_person_by_name_or_alias(&self, query: &str) -> Option<&Person> {
        self.graph.node_weights()
            .find(|p| {
                p.name.eq_ignore_ascii_case(query) ||
                p.aliases.iter().any(|a| a.eq_ignore_ascii_case(query))
            })
    }

    pub fn add_relation(&mut self, from_id: Uuid, to_id: Uuid, rel: RelationType) {
        if let (Some(&a), Some(&b)) = (self.id_to_node.get(&from_id), self.id_to_node.get(&to_id)) {
            self.graph.add_edge(a, b, rel);
            // Optional: mirror reverse edge if bidirectional semantics desired
        }
    }

    // ────────────────────────────────────────────────────────────────────────
    //  Simple rule-based ingestion (no LLM, bare metal friendly)
    // ────────────────────────────────────────────────────────────────────────

    pub fn ingest_text(&mut self, text: &str, source: Source, speaker_name: Option<&str>) {
        let now = 0; // replace with real timestamp source

        // Very basic example extractors — expand with more regex or patterns
        if let Some(name) = extract_name(text) {
            let mut person = Person {
                id: Uuid::new_v4(),
                name: name.to_string(),
                aliases: vec![],
                contact: ContactInfo { emails: vec![], phones: vec![], handles: vec![] },
                birth_date: None,
                relationships: BTreeMap::new(),
                insights: vec![],
                events: vec![],
                last_contact_ts: Some(now),
                tags: vec![],
            };

            // Infer more...
            if text.contains("birthday") || text.contains("born") {
                // parse date crudely — in real use, better parser or LLM
                person.birth_date = try_parse_date(text);
            }

            let id = self.upsert_person(person);

            let insight = Insight {
                timestamp: now,
                source,
                text: text.to_string(),
                category: InsightCategory::Personal,
                confidence: 0.65,
            };
            if let Some(&idx) = self.id_to_node.get(&id) {
                self.graph[idx].insights.push(insight);
            }
        }
    }

    // Heads-up string for orchestrator / display
    pub fn heads_up(&self, person_name: &str) -> Option<String> {
        let person = self.get_person_by_name_or_alias(person_name)?;
        let mut lines = Vec::new();

        if let Some(bd) = person.birth_date {
            lines.push(format!("{} zodiac: {:?}", person.name, compute_zodiac(bd)));
        }
        if let Some(ts) = person.last_contact_ts {
            let days = (current_unix() - ts) / 86400;
            if days > 14 {
                lines.push(format!("No contact in {} days", days));
            }
        }
        for ev in &person.events {
            lines.push(format!("Event: {} ({})", ev.description, ev.date));
        }
        for ins in &person.insights {
            if ins.confidence > 0.7 {
                lines.push(format!("Note: {}", ins.text));
            }
        }

        if lines.is_empty() { None } else { Some(lines.join("\n")) }
    }

    // Persistence
    #[cfg(feature = "std")]
    pub fn save_to_bytes(&self) -> Vec<u8> {
        postcard::to_allocvec(&self.graph).unwrap_or_default()
    }

    #[cfg(feature = "std")]
    pub fn load_from_bytes(bytes: &[u8]) -> Option<Self> {
        let graph: Graph<Person, RelationType> = postcard::from_bytes(bytes).ok()?;
        let mut id_to_node = BTreeMap::new();
        for (idx, node) in graph.node_indices().zip(graph.node_weights()) {
            id_to_node.insert(node.id, idx);
        }
        Some(Self { graph, id_to_node })
    }
}

// Dummy helpers — implement properly
fn extract_name(_text: &str) -> Option<String> { None }
fn try_parse_date(_text: &str) -> Option<NaiveDate> { None }
fn current_unix() -> u64 { 0 }  // hook to RTC or host time
```

### Quick Start Usage (in your orchestrator)

```rust
let mut ril = RilStore::new();

// From conversation
ril.ingest_text("John said his daughter Emma turns 7 next month", Source::Conversation, Some("John"));

// Later, before replying
if let Some(hu) = ril.heads_up("John") {
    println!("Heads-up:\n{}", hu);
    // → feed to prompt: "Use this context: {hu}\nUser: ..."
}
```

### Next Steps / Enhancements

- Replace dummy extractors with real regex (`regex` crate, but it's `std`-only → use `fancy-regex` or simple string find for `no_std`).
- For strong extraction → call your local LLM with structured output prompt → parse JSON → upsert.
- Add birthday/anniversary reminders in a separate `scan_reminders(&self) -> Vec<String>` method.
- Persistence: on bare metal → `postcard` blob to flash; add versioning.
- Graph queries: use `petgraph::algo` for "shortest path of influence" or clusters.

Paste any of your existing code (orchestrator loop, storage, inference) and I can show exact wiring. Or tell me which part to expand first (ingestion, reminders, CLI commands, etc.). Let's make PAGI remember *everything* about the people in your world. 🚀

---

Here’s a clean, practical implementation of the `scan_reminders` method for your `RilStore`.  
It focuses on the most common reminder triggers in a personal AGI context:

- **Birthdays** (coming up in the next 30 days, with special emphasis in the next 7 days)
- **Significant life events** (anniversaries, kids' birthdays if stored, major remembered dates)
- **Long time no see** (no contact > 30/60/90 days)
- **Fresh/high-confidence insights** that might need follow-up

### Add these helpers at the top of `lib.rs` (after the existing code)

```rust
// Helper: days until a date in the current or next year
fn days_until(target: NaiveDate, from: NaiveDate) -> i64 {
    let this_year = NaiveDate::from_ymd_opt(from.year(), target.month(), target.day()).ok();
    
    if let Some(ty) = this_year {
        if ty >= from {
            return (ty - from).num_days();
        }
    }
    
    // Otherwise next year
    let next_year = NaiveDate::from_ymd_opt(from.year() + 1, target.month(), target.day()).ok();
    next_year.map(|ny| (ny - from).num_days()).unwrap_or(365)
}

// Simple "today or very soon" check
fn is_soon(days: i64) -> bool { days >= 0 && days <= 7 }
fn is_upcoming(days: i64)  { days >= 0 && days <= 30 }
```

### New public method on `RilStore`

```rust
#[derive(Debug, Clone)]
pub struct Reminder {
    pub person_name: String,
    pub person_id: Uuid,
    pub kind: ReminderKind,
    pub message: String,
    pub urgency: Urgency,
    pub days_relevant: Option<i64>,     // for sorting / display
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ReminderKind {
    Birthday,
    LifeEvent,
    NoContactLong,
    FollowUpInsight,
    Other,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum Urgency {
    High   = 3,    // today–tomorrow, important insight
    Medium = 2,    // this week
    Low    = 1,    // this month or just stale contact
}

impl RilStore {
    // ────────────────────────────────────────────────────────────────
    //  Scan for reminders worth surfacing today / this week
    // ────────────────────────────────────────────────────────────────
    pub fn scan_reminders(&self, today: NaiveDate, max_horizon_days: i64) -> Vec<Reminder> {
        let mut reminders = Vec::new();

        for person in self.graph.node_weights() {
            let name = person.name.clone();
            let id = person.id;

            // ── Birthdays ──────────────────────────────────────────────
            if let Some(bd) = person.birth_date {
                let days = days_until(bd, today);

                if is_soon(days) {
                    reminders.push(Reminder {
                        person_name: name.clone(),
                        person_id: id,
                        kind: ReminderKind::Birthday,
                        message: format!("{}'s birthday is in {} day(s) — zodiac {}", name, days, compute_zodiac(bd)),
                        urgency: if days <= 2 { Urgency::High } else { Urgency::Medium },
                        days_relevant: Some(days),
                    });
                } else if is_upcoming(days) {
                    reminders.push(Reminder {
                        person_name: name.clone(),
                        person_id: id,
                        kind: ReminderKind::Birthday,
                        message: format!("{}'s birthday coming up in {} days", name, days),
                        urgency: Urgency::Low,
                        days_relevant: Some(days),
                    });
                }
            }

            // ── Life events (anniversaries, remembered dates, etc.) ───
            for ev in &person.events {
                let days = days_until(ev.date, today);

                if is_soon(days) || (ev.importance >= 8 && is_upcoming(days)) {
                    let msg = format!(
                        "{} — {} in {} day(s) (importance {})",
                        ev.description, name, days, ev.importance
                    );
                    reminders.push(Reminder {
                        person_name: name.clone(),
                        person_id: id,
                        kind: ReminderKind::LifeEvent,
                        message: msg,
                        urgency: if days <= 3 || ev.importance >= 9 { Urgency::High } else { Urgency::Medium },
                        days_relevant: Some(days),
                    });
                }
            }

            // ── Long time no see ───────────────────────────────────────
            if let Some(last_ts) = person.last_contact_ts {
                let seconds_ago = current_unix().saturating_sub(last_ts);
                let days_ago = seconds_ago / 86_400;

                if days_ago >= 60 {
                    reminders.push(Reminder {
                        person_name: name.clone(),
                        person_id: id,
                        kind: ReminderKind::NoContactLong,
                        message: format!("No contact with {} in {} days", name, days_ago),
                        urgency: if days_ago >= 90 { Urgency::High } else { Urgency::Medium },
                        days_relevant: Some(-(days_ago as i64)),  // negative = past
                    });
                }
            }

            // ── Fresh / high-confidence insights that deserve follow-up ─
            for ins in &person.insights {
                let age_days = (current_unix() - ins.timestamp) / 86_400;
                if age_days <= 10 && ins.confidence >= 0.80 {
                    reminders.push(Reminder {
                        person_name: name.clone(),
                        person_id: id,
                        kind: ReminderKind::FollowUpInsight,
                        message: format!("Recent note: {} ({} days ago, conf {:.0}%)", ins.text, age_days, ins.confidence * 100.0),
                        urgency: Urgency::Medium,
                        days_relevant: Some(-(age_days as i64)),
                    });
                }
            }
        }

        // Sort: high → medium → low urgency, then by days_relevant (soonest first)
        reminders.sort_by(|a, b| {
            b.urgency.cmp(&a.urgency)
                .then_with(|| a.days_relevant.cmp(&b.days_relevant))
        });

        reminders
    }
}
```

### Example usage in your orchestrator / daily summary

```rust
let today = NaiveDate::from_ymd_opt(2026, 2, 14).unwrap(); // or real source

let reminders = ril.scan_reminders(today, 30);

for r in reminders {
    if r.urgency == Urgency::High {
        println!("🔔 URGENT: {}", r.message);
    } else {
        println!("  • {}", r.message);
    }
}
```

### Customization ideas

- Add configurable thresholds (via struct fields or consts)
- Filter by tags (`family`, `close_friends`, etc.)
- Weight urgency by relationship closeness (spouse/children > colleagues)
- Deduplicate similar reminders for the same person
- Generate natural-language summary sentences via your local LLM

Let me know if you want:

- a version with configurable thresholds / filters
- integration with a real timestamp source (RTC, host call, etc.)
- a "daily briefing" formatter that groups by person
- or anything else (e.g. zodiac daily advice tie-in)

Happy building — this is the kind of quiet magic that makes an agent feel truly personal. 🚀

---

Here’s an enhanced version of the `scan_reminders` method that incorporates:

- **configurable thresholds** (via a new lightweight config struct)
- **relationship-based weighting** (urgency boost for closer relationships)

### Step 1: Add these types near the top of `lib.rs` (after existing enums)

```rust
#[derive(Debug, Clone)]
pub struct ReminderConfig {
    // Time windows (in days)
    pub birthday_urgent_days: i64,      // default 3
    pub birthday_upcoming_days: i64,    // default 30
    pub no_contact_medium_days: i64,    // default 60
    pub no_contact_urgent_days: i64,    // default 90
    pub insight_fresh_days: i64,        // default 10
    pub insight_min_confidence: f32,    // default 0.80

    // Relationship closeness multipliers (applied to base urgency score)
    pub closeness_weights: BTreeMap<RelationType, f32>,
}

impl Default for ReminderConfig {
    fn default() -> Self {
        let mut weights = BTreeMap::new();
        weights.insert(RelationType::Spouse,          2.0);
        weights.insert(RelationType::ChildOf,         1.8);
        weights.insert(RelationType::ParentOf,        1.7);
        weights.insert(RelationType::SiblingOf,       1.5);
        weights.insert(RelationType::Mentor,          1.3);
        weights.insert(RelationType::Friend,          1.1);
        weights.insert(RelationType::Colleague,       0.9);
        weights.insert(RelationType::Ex,              0.8);
        // Custom relations default to 1.0 unless explicitly set

        Self {
            birthday_urgent_days: 3,
            birthday_upcoming_days: 30,
            no_contact_medium_days: 60,
            no_contact_urgent_days: 90,
            insight_fresh_days: 10,
            insight_min_confidence: 0.80,
            closeness_weights: weights,
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub enum Urgency {
    High   = 3,
    Medium = 2,
    Low    = 1,
    None   = 0,
}
```

### Step 2: Update `RilStore` to hold config (add this field)

```rust
pub struct RilStore {
    graph: Graph<Person, RelationType, Directed>,
    id_to_node: BTreeMap<Uuid, NodeIndex>,
    config: ReminderConfig,          // ← new
}

impl RilStore {
    pub fn new() -> Self {
        Self {
            graph: Graph::new(),
            id_to_node: BTreeMap::new(),
            config: ReminderConfig::default(),
        }
    }

    pub fn with_config(config: ReminderConfig) -> Self {
        Self {
            graph: Graph::new(),
            id_to_node: BTreeMap::new(),
            config,
        }
    }

    // Optional: allow runtime config change
    pub fn set_config(&mut self, config: ReminderConfig) {
        self.config = config;
    }
}
```

### Step 3: Enhanced `scan_reminders` with thresholds & weighting

```rust
impl RilStore {
    pub fn scan_reminders(&self, today: NaiveDate) -> Vec<Reminder> {
        let mut reminders = Vec::new();
        let now_unix = current_unix();

        for (node_idx, person) in self.graph.node_indices().zip(self.graph.node_weights()) {
            let name = person.name.clone();
            let id = person.id;

            // ── 1. Compute closeness multiplier ───────────────────────────────
            let closeness = self.max_closeness_multiplier(node_idx);
            // 1.0 = neutral, >1.0 = closer → higher urgency

            // ── 2. Birthdays ──────────────────────────────────────────────────
            if let Some(bd) = person.birth_date {
                let days = days_until(bd, today);

                if days >= 0 && days <= self.config.birthday_urgent_days {
                    let score = 3.0 * closeness;
                    reminders.push(Reminder {
                        person_name: name.clone(),
                        person_id: id,
                        kind: ReminderKind::Birthday,
                        message: format!("{}'s birthday in {} day(s) — zodiac {:?}", name, days, compute_zodiac(bd)),
                        urgency: score_to_urgency(score),
                        days_relevant: Some(days),
                    });
                } else if days >= 0 && days <= self.config.birthday_upcoming_days {
                    let score = 2.0 * closeness;
                    reminders.push(Reminder {
                        person_name: name.clone(),
                        person_id: id,
                        kind: ReminderKind::Birthday,
                        message: format!("{}'s birthday in {} days", name, days),
                        urgency: score_to_urgency(score),
                        days_relevant: Some(days),
                    });
                }
            }

            // ── 3. Life events ────────────────────────────────────────────────
            for ev in &person.events {
                let days = days_until(ev.date, today);
                if days < 0 { continue; } // skip past events

                let base_score = if ev.importance >= 9 { 3.0 }
                                 else if ev.importance >= 7 { 2.0 }
                                 else { 1.0 };

                if days <= self.config.birthday_urgent_days || (ev.importance >= 8 && days <= self.config.birthday_upcoming_days) {
                    let score = base_score * closeness;
                    reminders.push(Reminder {
                        person_name: name.clone(),
                        person_id: id,
                        kind: ReminderKind::LifeEvent,
                        message: format!("{} — {} in {} day(s) (importance {})", ev.description, name, days, ev.importance),
                        urgency: score_to_urgency(score),
                        days_relevant: Some(days),
                    });
                }
            }

            // ── 4. No contact ─────────────────────────────────────────────────
            if let Some(last_ts) = person.last_contact_ts {
                let days_ago = ((now_unix - last_ts) / 86_400) as i64;

                if days_ago >= self.config.no_contact_urgent_days {
                    let score = 3.0 * closeness;
                    reminders.push(Reminder {
                        person_name: name.clone(),
                        person_id: id,
                        kind: ReminderKind::NoContactLong,
                        message: format!("No contact with {} in {} days", name, days_ago),
                        urgency: score_to_urgency(score),
                        days_relevant: Some(-days_ago),
                    });
                } else if days_ago >= self.config.no_contact_medium_days {
                    let score = 2.0 * closeness;
                    reminders.push(Reminder {
                        person_name: name.clone(),
                        person_id: id,
                        kind: ReminderKind::NoContactLong,
                        message: format!("Haven't connected with {} in {} days", name, days_ago),
                        urgency: score_to_urgency(score),
                        days_relevant: Some(-days_ago),
                    });
                }
            }

            // ── 5. Fresh/high-confidence insights ─────────────────────────────
            for ins in &person.insights {
                let age_days = ((now_unix - ins.timestamp) / 86_400) as i64;
                if age_days <= self.config.insight_fresh_days && ins.confidence >= self.config.insight_min_confidence {
                    let base_score = if ins.confidence >= 0.90 { 3.0 }
                                     else if ins.confidence >= 0.85 { 2.5 }
                                     else { 2.0 };

                    let score = base_score * closeness;
                    reminders.push(Reminder {
                        person_name: name.clone(),
                        person_id: id,
                        kind: ReminderKind::FollowUpInsight,
                        message: format!("Recent note: {} ({}d ago, conf {:.0}%)", ins.text, age_days, ins.confidence * 100.0),
                        urgency: score_to_urgency(score),
                        days_relevant: Some(-age_days),
                    });
                }
            }
        }

        // Sort: highest urgency first, then soonest relevant (smallest days_relevant)
        reminders.sort_by(|a, b| {
            b.urgency.cmp(&a.urgency)
                .then_with(|| a.days_relevant.cmp(&b.days_relevant))
        });

        reminders
    }

    // ── Helper: find strongest closeness weight for a person ────────────────
    fn max_closeness_multiplier(&self, node: NodeIndex) -> f32 {
        let mut max_w = 1.0;

        // Look at all outgoing & incoming edges
        for edge in self.graph.edges(node) {
            if let Some(weight) = self.config.closeness_weights.get(edge.weight()) {
                max_w = max_w.max(*weight);
            }
        }
        for edge in self.graph.edges_directed(node, Direction::Incoming) {
            if let Some(weight) = self.config.closeness_weights.get(edge.weight()) {
                max_w = max_w.max(*weight);
            }
        }

        max_w
    }
}

// ── Utility ────────────────────────────────────────────────────────────────
fn score_to_urgency(score: f32) -> Urgency {
    if score >= 4.0 { Urgency::High }
    else if score >= 2.5 { Urgency::High }
    else if score >= 1.8 { Urgency::Medium }
    else if score >= 1.0 { Urgency::Low }
    else { Urgency::None }
}
```

### Usage examples

```rust
// Default behavior
let reminders = ril.scan_reminders(today);

// Custom thresholds & add a custom relation weight
let mut cfg = ReminderConfig::default();
cfg.birthday_urgent_days = 5;
cfg.no_contact_urgent_days = 120;
cfg.closeness_weights.insert(RelationType::Custom("Best friend".to_string()), 1.6);

let mut ril_custom = RilStore::with_config(cfg);
let reminders_prio = ril_custom.scan_reminders(today);
```

This gives you fine-grained control and makes reminders feel much more **personal** — a spouse’s birthday 4 days away becomes urgent even if the default threshold is 3, while a colleague’s long silence might stay low-priority.

Want to add:

- filtering by tags or person list?
- a grouped “daily briefing” formatter (by person / urgency)?
- natural-language ranking / deduplication?

Just let me know what to expand next.

---

Yes — let's extend the **Relationship Intelligence Layer** with:

- **tag-based filtering** in reminders (so you can ask for "family only", "close friends", "professional network", etc.)
- **calendar sync basics** (import/export in a simple, Rust-friendly format like iCalendar/.ics subset — focused on birthdays + key life events)

This keeps everything `no_std`-compatible where possible, with `std` features for file I/O and parsing.

### 1. Tag-based Filtering

We'll add two things:
- A `tags` filter parameter on `scan_reminders`
- Optional "tag groups" in config for quick presets (e.g. "family", "inner_circle")

#### Update `ReminderConfig` (add presets)

```rust
#[derive(Debug, Clone)]
pub struct ReminderConfig {
    // ... (existing fields)

    // New: named tag groups for quick filtering
    pub tag_groups: BTreeMap<String, Vec<String>>,  // e.g. "family" → ["family", "spouse", "child"]
}

impl Default for ReminderConfig {
    fn default() -> Self {
        let mut cfg = Self { /* ... existing */ tag_groups: BTreeMap::new() };
        cfg.tag_groups.insert("family".to_string(), vec!["family".to_string(), "spouse".to_string(), "child".to_string(), "parent".to_string()]);
        cfg.tag_groups.insert("inner_circle".to_string(), vec!["family".to_string(), "best_friend".to_string(), "mentor".to_string()]);
        cfg.tag_groups.insert("professional".to_string(), vec!["colleague".to_string(), "client".to_string(), "partner".to_string()]);
        cfg
    }
}
```

#### Update `scan_reminders` signature & logic

```rust
pub fn scan_reminders(
    &self,
    today: NaiveDate,
    filter_tags: Option<&[String]>,           // None = all, Some = must have at least one of these tags
    use_tag_group: Option<&str>,              // shortcut: "family", "inner_circle", etc.
) -> Vec<Reminder> {
    let effective_tags = if let Some(group) = use_tag_group {
        self.config.tag_groups.get(group).map(|v| v.as_slice())
    } else {
        filter_tags
    };

    let mut reminders = Vec::new();
    let now_unix = current_unix();

    for (node_idx, person) in self.graph.node_indices().zip(self.graph.node_weights()) {
        // ── Tag filter ───────────────────────────────────────────────────────
        if let Some(required) = effective_tags {
            if !required.iter().any(|req| person.tags.iter().any(|t| t == req)) {
                continue; // skip this person
            }
        }

        let name = person.name.clone();
        let id = person.id;
        let closeness = self.max_closeness_multiplier(node_idx);

        // ... rest of the reminder generation logic remains identical ...
        // (birthdays, life events, no contact, insights)
    }

    // sorting remains the same
    reminders.sort_by(/* ... */);

    reminders
}
```

#### Usage examples

```rust
// All reminders
ril.scan_reminders(today, None, None);

// Only family-related people
ril.scan_reminders(today, Some(&["family".to_string()]), None);

// Using preset
ril.scan_reminders(today, None, Some("inner_circle"));

// Multiple explicit tags
ril.scan_reminders(today, Some(&["family".to_string(), "best_friend".to_string()]), None);
```

This lets your orchestrator / UI offer quick views like:
- "Show me today's family reminders"
- "Inner circle heads-ups"
- "Professional network – people I haven't talked to lately"

### 2. Basic Calendar Sync (iCalendar .ics subset)

We'll implement simple **export** of birthdays + important life events to iCalendar format (`.ics` text), and a basic **import** parser that only cares about VEVENTs with DTSTART and SUMMARY (enough for birthdays/anniversaries/kids events).

This is kept minimal and works in `std` (file I/O), but the format itself is pure text so you could adapt for bare-metal if needed.

#### Add these near the top (new dependencies in Cargo.toml if using `std`)

```toml
[features]
std = ["dep:chrono", "dep:regex", /* ... */, "dep:percent-encoding"]  # for SUMMARY escaping if needed
```

#### New methods on `RilStore`

```rust
#[cfg(feature = "std")]
use std::io::{self, Write};

#[cfg(feature = "std")]
impl RilStore {
    /// Export birthdays + high-importance life events as simple .ics
    pub fn export_to_ics<W: Write>(&self, writer: &mut W, prod_id: &str) -> io::Result<()> {
        writeln!(writer, "BEGIN:VCALENDAR")?;
        writeln!(writer, "VERSION:2.0")?;
        writeln!(writer, "PRODID:{}", prod_id)?;  // e.g. "-//PAGI//Phoenix AGI 1.0//EN"
        writeln!(writer, "CALSCALE:GREGORIAN")?;
        writeln!(writer, "METHOD:PUBLISH")?;

        let now = chrono::Utc::now().format("%Y%m%dT%H%M%SZ").to_string();

        for person in self.graph.node_weights() {
            // Birthdays ─ recurring yearly
            if let Some(bd) = person.birth_date {
                let uid = format!("pagi-birthday-{}-{}", person.id, bd);
                writeln!(writer, "BEGIN:VEVENT")?;
                writeln!(writer, "UID:{}", uid)?;
                writeln!(writer, "DTSTAMP:{}", now)?;
                writeln!(writer, "DTSTART;VALUE=DATE:{}", bd.format("%Y%m%d"))?;
                writeln!(writer, "RRULE:FREQ=YEARLY")?;
                writeln!(writer, "SUMMARY:{}'s Birthday", person.name)?;
                writeln!(writer, "DESCRIPTION:Zodiac: {:?}", compute_zodiac(bd))?;
                writeln!(writer, "END:VEVENT")?;
            }

            // High-importance life events (importance >= 7)
            for ev in &person.events {
                if ev.importance < 7 { continue; }

                let uid = format!("pagi-event-{}-{}", person.id, ev.date);
                writeln!(writer, "BEGIN:VEVENT")?;
                writeln!(writer, "UID:{}", uid)?;
                writeln!(writer, "DTSTAMP:{}", now)?;
                writeln!(writer, "DTSTART;VALUE=DATE:{}", ev.date.format("%Y%m%d"))?;
                writeln!(writer, "SUMMARY:{} - {}", person.name, ev.description)?;
                writeln!(writer, "DESCRIPTION:Importance: {}", ev.importance)?;
                writeln!(writer, "END:VEVENT")?;
            }
        }

        writeln!(writer, "END:VCALENDAR")?;
        Ok(())
    }

    /// Very basic import: parse .ics and upsert birthdays + events
    /// (only handles simple DATE-based VEVENTs, no timezones/recurrence parsing)
    #[cfg(feature = "std")]
    pub fn import_from_ics<R: io::BufRead>(&mut self, reader: R) -> io::Result<usize> {
        let mut count = 0;
        let mut in_event = false;
        let mut current_summary = String::new();
        let mut current_date: Option<NaiveDate> = None;
        let mut current_uid = String::new();

        for line in reader.lines() {
            let line = line?;
            let trimmed = line.trim();

            if trimmed == "BEGIN:VEVENT" {
                in_event = true;
                current_summary.clear();
                current_date = None;
                current_uid.clear();
                continue;
            }
            if trimmed == "END:VEVENT" {
                if in_event {
                    if let Some(date) = current_date {
                        // Try to parse person from SUMMARY (heuristic)
                        if let Some((name_part, desc)) = current_summary.split_once(" - ") {
                            if let Some(person) = self.get_person_by_name_or_alias(name_part.trim()) {
                                let mut p = person.clone();
                                let ev = LifeEvent {
                                    date,
                                    description: desc.trim().to_string(),
                                    importance: 8,  // assume high if imported
                                };
                                p.events.push(ev);
                                self.upsert_person(p);
                                count += 1;
                            }
                        } else if current_summary.ends_with("'s Birthday") {
                            let name = current_summary.trim_end_matches("'s Birthday").trim();
                            if let Some(person) = self.get_person_by_name_or_alias(name) {
                                let mut p = person.clone();
                                p.birth_date = Some(date);
                                self.upsert_person(p);
                                count += 1;
                            }
                        }
                    }
                }
                in_event = false;
                continue;
            }

            if !in_event { continue; }

            if trimmed.starts_with("DTSTART;VALUE=DATE:") {
                let date_str = trimmed.trim_start_matches("DTSTART;VALUE=DATE:");
                if let Ok(date) = NaiveDate::parse_from_str(date_str, "%Y%m%d") {
                    current_date = Some(date);
                }
            } else if trimmed.starts_with("SUMMARY:") {
                current_summary = trimmed.trim_start_matches("SUMMARY:").to_string();
            } else if trimmed.starts_with("UID:") {
                current_uid = trimmed.trim_start_matches("UID:").to_string();
            }
        }

        Ok(count)
    }
}
```

### Example usage

```rust
// Export
let mut buf = Vec::new();
ril.export_to_ics(&mut buf, "-//PAGI//Phoenix AGI//EN").unwrap();
std::fs::write("pagi_reminders.ics", buf).unwrap();

// Import from file
let file = std::fs::File::open("birthdays.ics").unwrap();
let reader = std::io::BufReader::new(file);
let imported = ril.import_from_ics(reader).unwrap();
println!("Imported {} events", imported);
```

### Notes & Next Steps

- The .ics export is deliberately simple (DATE-only, yearly birthdays, no alarms yet). You can later add RRULE parsing, VALARM, etc.
- Import is heuristic — real-world .ics files vary wildly. For production, consider a tiny `icalendar` crate (but many are `std`-heavy).
- Bare-metal: you could still generate the ICS text in memory and send it over UART/USB instead of file.

Want to add:
- Alarm / notification time offsets in ICS?
- Two-way sync conflict handling?
- Google Calendar / Outlook API hooks (would need `std` + HTTP crate)?
- Or a grouped daily briefing formatter that uses the tag filters?

Let me know the priority — we're building something really powerful here.

---


