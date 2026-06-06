# How I Built MeetMind: A Contact Memory System That Briefs You Before You Walk Into the Room

I've sat across from someone I'd met twice before and completely blanked on what we discussed last time — their budget, their hesitation about timelines, the thing I promised to follow up on. It's an embarrassing failure mode, and I kept waiting for a tool that solved it properly. Eventually I built one.

## What the System Does and How It Fits Together

MeetMind is a pre-meeting intelligence tool. The core loop is two steps: after a meeting, you save notes about the person — budget, preferences, commitments, decisions. Before your next meeting with them, you type their name and get a structured briefing: a summary of past interactions, key things to remember, and suggested conversation openers. Powered by Groq's Llama 3.3 model, the whole round trip is fast enough that the briefing feels instant.

The stack is deliberately minimal. The entire repository fits in a single directory:

```
templates/index.html   <- the entire frontend
app.py                 <- Flask routes + request handling
ai_brain.py            <- Groq API integration + prompt logic
memory_vault.py        <- persistence layer (JSON-backed memory store)
memory_store.json      <- the actual stored contact memories
requirements.txt
.gitignore
```

No database server. No ORM. No background workers. The whole thing starts with `python app.py`.

## My Role: Leading the Backend

I led the backend architecture and was personally responsible for writing all three Python files: `app.py`, `ai_brain.py`, and `memory_vault.py`. These three files are where every meaningful decision in the system lives. My teammates handled the frontend design; I was accountable for everything that makes the system actually work — the memory model, the AI integration, the API routing, and keeping all of it wired together cleanly enough that a teammate could pick up any piece without asking me what it did.

That division of responsibility sounds clean in retrospect. In practice, it meant I was the single point of failure for the backend during a compressed build cycle, debugging API key errors at 1am while also reviewing pull requests on the HTML side. Leading a small team under time pressure means the technical decisions and the people decisions happen simultaneously — you can't separate them.

## The Three Files: How They Divide the Problem

### `memory_vault.py` — The Memory Layer

This is the persistence layer. It reads from and writes to `memory_store.json`, keyed by lowercase contact name. The design constraint I set for myself was strict: this file knows nothing about AI. It stores strings, retrieves lists of strings, and writes to disk. Full stop.

```python
MEMORY_FILE = "memory_store.json"

class MockHindsightClient:
    def __init__(self):
        self.storage = self._load()

    def _load(self):
        if os.path.exists(MEMORY_FILE):
            with open(MEMORY_FILE, "r") as f:
                return json.load(f)
        return {}

    def _save(self):
        with open(MEMORY_FILE, "w") as f:
            json.dump(self.storage, f, indent=2)

    def retain(self, text):
        if "Meeting with " in text:
            try:
                content_split = text.split("Meeting with ")[1]
                parts = content_split.split(": ")
                if len(parts) >= 2:
                    contact_name = parts[0].strip().lower()
                    notes = ": ".join(parts[1:]).strip()
                    if contact_name in self.storage:
                        self.storage[contact_name].append(notes)
                    else:
                        self.storage[contact_name] = [notes]
                    self._save()  # saves to file permanently
                    return True
            except Exception:
                return False
        return False

    def recall(self, query_name):
        contact_name = query_name.strip().lower()
        return self.storage.get(contact_name, [])

hindsight_db = MockHindsightClient()
```

The class is named `MockHindsightClient` deliberately. It mirrors the interface of [Hindsight](https://github.com/vectorize-io/hindsight), the open-source [agent memory system](https://vectorize.io/what-is-agent-memory) from Vectorize. The `retain()` and `recall()` method names map directly onto Hindsight's production API. The intent was to write the simplest possible implementation that satisfies the interface, so swapping in the real Hindsight client later requires changing one import and one instantiation — nothing else in `app.py` needs to change.

One thing that caused me real pain here: the text parsing in `retain()`. The format `"Meeting with {contact}: {notes}"` is a convention I established so the entire system has one canonical way of encoding a memory. But that convention has to be known by `app.py` when it formats the string before calling `retain()`, and any mismatch is silent — the method just returns `False` and nothing gets saved. I spent more time than I'd like debugging this because the failure doesn't surface loudly. It's the kind of implicit contract that bites you when you're working fast with other people.

### `ai_brain.py` — The Prompt Layer

This file owns one function: take a contact name and a list of historical notes, call Groq, and return a structured briefing. The entire file is about 40 lines.

```python
import os
from groq import Groq

# Initialize the Groq Brain using the key inside the hidden .env file
client = Groq(api_key=os.getenv("GROQ_API_KEY"))

def generate_meeting_briefing(contact_name, past_history_list):
    if past_history_list:
        formatted_history = "\n".join([f"- {note}" for note in past_history_list])
    else:
        formatted_history = "No previous history found. This is your very first meeting with them."

    system_instruction = (
        "You are an elite executive assistant AI. Your goal is to prepare a brief, "
        "high-impact, bulleted meeting preparation guide for the user."
    )

    user_prompt = f"""
Please prepare a briefing for my upcoming meeting with: {contact_name}

Here is the historical context of my past interactions with this person:
{formatted_history}

Provide your output exactly in this structure:
1. SUMMARY OF PAST INTERACTION (Keep it short)
2. KEY REMINDERS (Promises made or things to look out for)
3. SUGGESTED CONVERSATION OPENERS
"""
    response = client.chat.completions.create(
        model="llama-3.3-70b-versatile",
        messages=[
            {"role": "system", "content": system_instruction},
            {"role": "user", "content": user_prompt}
        ],
        temperature=0.6
    )
    return response.choices[0].message.content
```

`temperature=0.6` is a deliberate product decision, not a default I left in. Lower temperature keeps the model anchored to the stored facts — you don't want it inventing follow-ups that never happened. The three-section output structure is enforced in the prompt text rather than through JSON mode, because the briefing is meant to be read by a human directly. Keeping it readable prose rather than structured data simplified both the prompt and the frontend rendering.

### `app.py` — The Routing Layer

I wrote three routes. The most important is `/get_briefing`, where the memory layer and the AI layer meet:

```python
from flask import Flask, render_template, request, jsonify
from dotenv import load_dotenv
from memory_vault import hindsight_db
from ai_brain import generate_meeting_briefing

load_dotenv()
app = Flask(__name__)

@app.route('/get_briefing', methods=['POST'])
def get_briefing():
    contact = request.json.get('contact', '').strip()
    if not contact:
        return jsonify({"error": "Please enter a valid name."}), 400

    # Step 1: Look up the person in our memory vault
    history = hindsight_db.recall(contact)

    # Step 2: Pass that memory into our Groq AI Engine
    briefing_text = generate_meeting_briefing(contact, history)

    return jsonify({
        "briefing": briefing_text,
        "has_memory": len(history) > 0
    })

@app.route('/save_notes', methods=['POST'])
def save_notes():
    data = request.json
    contact = data.get('contact', '').strip()
    notes = data.get('notes', '').strip()
    if not contact or not notes:
        return jsonify({"error": "Both name and notes fields are required!"}), 400

    success = hindsight_db.retain(text=f"Meeting with {contact}: {notes}")

    if success:
        return jsonify({"status": f"Successfully remembered details for {contact}!"})
    else:
        return jsonify({"error": "Failed to log memory structure."}), 500

@app.route('/get_stats')
def get_stats():
    contacts = len(hindsight_db.storage)
    notes = sum(len(v) for v in hindsight_db.storage.values())
    return jsonify({"contacts": contacts, "notes": notes})
```

The `has_memory` flag in the `/get_briefing` response lets the frontend differentiate between a briefing built from real history and a first-meeting briefing. That distinction matters for user trust — you want to signal clearly when the system is drawing on actual stored context versus generating something generic. The `/get_stats` route feeds the dashboard stats banner: contacts remembered, notes saved — live counters that give users a concrete sense that memory is accumulating.

## The Hardest Part: API Keys and Coordination Across a Team

The technical problems were solvable. The most painful operational problem was managing the Groq API key across a team working in different local environments.

The system uses `load_dotenv()` at startup and expects `GROQ_API_KEY` in a `.env` file that isn't committed to the repo. That's standard practice, but when you're leading a team through an integration under time pressure, "standard practice" still produces a non-trivial number of `KeyError` exceptions, `401` responses from Groq, and confused teammates who can't tell whether they have a code bug or a config issue.

A few things I'd do differently: fail loudly at startup rather than at request time. A check like this at the top of `app.py` would have saved several rounds of debugging:

```python
if not os.getenv("GROQ_API_KEY"):
    raise RuntimeError("GROQ_API_KEY is not set. Check your .env file.")
```

Silent failures in environment configuration are a team productivity tax. If the server starts, people assume it works. The error only surfaces when someone makes a request and gets a 500, at which point the debugging starts from scratch rather than from a clear signal. As team lead I should have built that check in from day one instead of documenting the `.env` setup in a README and assuming everyone would follow it.

The other coordination challenge was keeping `app.py`, `ai_brain.py`, and `memory_vault.py` in sync during parallel development. Because I owned all three files, that was mostly fine — but the moment I handed off any piece for a teammate to extend, the implicit contracts (the text encoding format, the expected return types from `recall()`) had to be made explicit. A typed interface or even a simple docstring would have prevented several "why isn't this saving" conversations.

## How a User Actually Experiences MeetMind

The UI has two modes, corresponding directly to the two main routes.

**Before the meeting:** The user types a contact name — Rahul, Priya, whoever they're about to meet — and clicks "Generate my briefing." The frontend POSTs to `/get_briefing`. The server calls `hindsight_db.recall("rahul")`, gets back the stored notes list, passes it to `generate_meeting_briefing()`, and returns the briefing text. If `has_memory` is true, the UI renders a memory-backed briefing. If false — first meeting — it renders preparation advice framed for a cold introduction. The whole thing runs on Groq's hosted inference, so it comes back in under two seconds.

**After the meeting:** The user fills in what happened — budget discussed, preferences mentioned, decisions made — and clicks "Save to memory." The frontend POSTs to `/save_notes`. The server formats it as `"Meeting with Rahul: Budget Rs.50,000. Prefers mornings."` and passes it to `hindsight_db.retain()`, which appends the note to that contact's list and writes to disk immediately. The next briefing for that contact will include this note.

The real-world example from the dashboard is illustrative: "Memory Found — Last met 3 days ago. Budget: Rs.50,000. Prefers mornings." That surface-level simplicity is the whole point. You don't need a dashboard full of charts to walk into a meeting prepared; you need those three facts, in that form, thirty seconds before you join the call.

## Why the Hindsight Interface Was Worth Mirroring

The `MockHindsightClient` naming was a deliberate architectural commitment, not a placeholder I planned to rename later. The [Hindsight docs](https://hindsight.vectorize.io/) describe an agent memory system built around retain, recall, and reflect operations, with four retrieval strategies running in parallel: semantic similarity, entity-aware lookup, keyword matching, and graph traversal. Results come back sized by token budget, not a fixed top-k cutoff.

For a small contact list, a JSON file keyed by lowercase name works fine. The predictable failure mode is scale: two contacts named "Rahul" become ambiguous; notes from three months ago are weighted equally to notes from yesterday; there's no semantic search, so a query for "what did we discuss about pricing" returns nothing useful unless someone wrote "pricing" verbatim in the notes. The retain/recall abstraction means the upgrade to a production [Hindsight](https://github.com/vectorize-io/hindsight) backend is a drop-in:

```python
# Production upgrade: swap the mock for the real Hindsight client
from hindsight_client import Hindsight

client = Hindsight(base_url="http://localhost:8888")
client.retain(bank_id="user-contacts", content=f"Meeting with {contact}: {notes}")
results = client.recall(bank_id="user-contacts", query=contact_name)
```

The `bank_id` parameter also adds per-user memory isolation — critical the moment more than one person is using the system. One import change, one instantiation change. Every route in `app.py` continues to work unchanged because the interface contract was respected from the beginning.

## Lessons Learned

**1. Naming your mock after the production interface is not over-engineering.** Calling the class `MockHindsightClient` and matching its method signatures to Hindsight's actual API meant the upgrade path was designed before the first line of real code was written. That decision cost nothing and will save real refactoring time later.

**2. Silent environment failures are a team productivity tax.** Every minute spent debugging a missing API key is a minute not spent on the actual problem. Fail at startup, loudly and specifically. `raise RuntimeError("GROQ_API_KEY is not set")` is one line that pays for itself the first time a teammate tries to run the server.

**3. Leading a backend under time pressure means owning the implicit contracts.** The string format `"Meeting with {contact}: {notes}"` is load-bearing — `retain()` parses it and `save_notes()` produces it. When you establish a convention as team lead, you own documenting it immediately, not after someone silently breaks it.

**4. Temperature is a product decision.** `temperature=0.6` produces briefings that stay close to stored facts. Higher values generate more creative conversation openers but risk hallucinated follow-ups you never promised. For a feature people will rely on before an important meeting, accuracy beats creativity.

**5. The zero-history case is a feature, not an edge case.** When `has_memory` is false, the system still generates a useful first-meeting preparation guide rather than an empty state or an error. Handling that path as a distinct, useful mode means the tool is valuable from the first use, not only after data accumulates.

---

The code is on GitHub. If you're building anything that involves per-contact or per-entity memory across sessions, I'd look at [Hindsight](https://github.com/vectorize-io/hindsight) seriously before building a custom retrieval layer — the benchmark results on LongMemEval are independently reproduced, and the retain/recall API is clean enough to build against from day one.

## References
- Hindsight GitHub: https://github.com/vectorize-io/hindsight
- Hindsight docs: https://hindsight.vectorize.io/
- Vectorize agent memory: https://vectorize.io/what-is-agent-memory

---

Rajshree Singh is a systems-focused backend engineer dedicated to architecting stable, low-latency server architecture and seamless data pipelines for intelligent automation systems.
