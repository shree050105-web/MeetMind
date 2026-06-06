# I Got Tired of Walking Into Conversations Without Context. So I Built an AI That Remembers.

*By Vineeta Choudhary*

---

Here is a scenario most professionals know well.

Someone follows up on a conversation from three weeks ago. They reference a budget figure, a deadline, a specific requirement. You remember the broad picture — but the details are gone. You spend the first few minutes of the conversation scrambling, scrolling, piecing together context that should have been instant.

This is not a rare edge case. It happens every day, across every industry. Sales managers track dozens of leads simultaneously. Consultants juggle requirements across multiple projects. Account managers handle relationship histories that span months or years.

The problem is not that people are disorganised. The problem is that there is no good system for storing and retrieving professional context over time. Notes get lost. CRM entries go unread. And most AI tools make it worse — every session starts completely fresh.

I built MeetMind to fix this.

---

## What MeetMind Is

MeetMind is an AI-powered meeting memory assistant. It stores everything you know about a professional contact and recalls it intelligently the moment you need it.

The interface is clean and focused — a cream and crimson design with a clear headline: *"Never walk into a meeting unprepared."*

The core workflow is two steps:

**Step 1 — Save notes after any interaction:**
Type the contact's name, write what was discussed, click Save. MeetMind stores it permanently in a semantic memory bank.

**Step 2 — Generate a briefing before any meeting:**
Type the contact's name, click Generate Briefing. MeetMind recalls every relevant memory and the AI produces a structured preparation guide in seconds.

**Without memory:**
```
User: "Prepare me for Rahul's meeting."
Agent: "I don't have any information about Rahul."
```

**After saving notes:**
```
User: "Prepare me for Rahul's meeting."
Agent: "Rahul needs a React website built with Tailwind CSS.
        Budget is ₹50,000. Follow-up is scheduled for Monday.
        He prefers morning meetings.
        Suggested opener: confirm whether his timeline has shifted."
```

That difference — between a blank slate and deeply specific context — is what persistent memory makes possible.

---

## The Architecture

The stack is simple and deliberate. Every tool solves exactly one problem.

```
User Browser (HTML / CSS / JavaScript)
            ↓
    Flask Backend (Python)
       ↓               ↓
Hindsight Memory     Groq AI
  (Vectorize)      (llama3-70b)
       ↓               ↓
  Persistent       Personalised
   Storage           Briefing
```

**Frontend:** Pure HTML, CSS, and JavaScript. No frameworks, no build tools. The UI communicates with Flask using the browser's native `fetch()` API.

**Backend:** Python Flask. Two routes handle everything — one for saving memories, one for generating briefings.

**Memory:** Hindsight by Vectorize. This is what makes MeetMind genuinely different from a standard chatbot — more on this below.

**AI:** Groq running llama3-70b for fast, structured inference.

**Deployment:** Render — live at https://meetmind-iatt.onrender.com

---

## Building the UI and Frontend

I led the UI design and frontend development for MeetMind.

The visual direction was intentional. Most AI tools default to dark themes and technical aesthetics. I went the opposite direction — a warm cream background, deep crimson navigation, serif typography, and a professional hero image. The goal was to make MeetMind feel like a tool that belongs in a real professional environment, not a developer side project.

The navbar is minimal: **Briefing** and **Save Notes** — exactly the two things users need, nothing else.

The hero copy lands immediately: *"Never walk into a meeting unprepared. MeetMind remembers every conversation, follow-up, and preference — so you always know exactly what to say."*

On the technical side, the frontend uses async JavaScript to communicate with Flask without any page reloads:

```javascript
async function generateBriefing() {
  const contact = document.getElementById("contact").value.trim();

  briefingBox.innerHTML = '<span class="loading">Generating briefing...</span>';

  const response = await fetch("/get_briefing", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ contact: contact })
  });

  const data = await response.json();
  briefingBox.textContent = data.briefing;
}
```

One async function. No dependencies. Instant feedback.

---

## Building the Flask Backend

I also contributed to the Flask backend. The backend is intentionally thin — its job is to connect memory and AI, not to do heavy lifting itself.

```python
@app.route('/get_briefing', methods=['POST'])
def get_briefing():
    contact = request.json.get('contact', '').strip()

    # Step 1: Recall relevant memories from Hindsight
    history = hindsight_db.recall(contact)

    # Step 2: Pass memories into Groq AI for briefing generation
    briefing_text = generate_meeting_briefing(contact, history)

    return jsonify({
        "briefing": briefing_text,
        "has_memory": len(history) > 0
    })

@app.route('/save_notes', methods=['POST'])
def save_notes():
    contact = request.json.get('contact')
    notes = request.json.get('notes')

    success = hindsight_db.retain(
        text=f"Meeting with {contact}: {notes}"
    )

    return jsonify({
        "status": f"Successfully remembered details for {contact}!"
    })
```

Receive request → recall memory → call AI → return response. That is the entire backend. The simplicity is the point.

---

## The Memory Layer: Why Hindsight Changes Everything

Standard databases do exact keyword matching. If you search for "Rahul" and the stored note says "the React website client from Monday", you get nothing back.

Hindsight stores meaning as semantic embeddings — mathematical representations of what a note means, not just what it literally says. When you search for "Rahul", it retrieves everything conceptually related to Rahul, ranked by semantic similarity.

```python
def retain(self, text):
    """Store a memory permanently in Hindsight."""
    response = requests.post(
        f"{HINDSIGHT_URL}/retain",
        headers={"Authorization": f"Bearer {HINDSIGHT_API_KEY}"},
        json={"bank_id": BANK_ID, "text": text}
    )
    return response.status_code == 200

def recall(self, query_name):
    """Retrieve the most relevant memories for a contact."""
    response = requests.post(
        f"{HINDSIGHT_URL}/recall",
        headers={"Authorization": f"Bearer {HINDSIGHT_API_KEY}"},
        json={
            "bank_id": BANK_ID,
            "query": query_name,
            "top_k": 5
        }
    )
    results = response.json().get("results", [])
    return [r.get("text", "") for r in results]
```

Every note saved accumulates permanently. The memory bank never resets. The agent becomes measurably more useful with every interaction logged — not because the AI model changes, but because the context it receives gets richer over time.

---

## Generating the Briefing with Groq

Once Hindsight returns the relevant memories, Groq's llama3-70b model turns raw notes into a clean, structured briefing:

```python
def generate_meeting_briefing(contact_name, past_history_list):

    formatted_history = "\n".join([f"- {note}" for note in past_history_list])

    response = client.chat.completions.create(
        model="llama3-70b-8192",
        messages=[
            {
                "role": "system",
                "content": "You are an elite executive assistant AI. Prepare concise, actionable meeting briefings."
            },
            {
                "role": "user",
                "content": f"""
                    Prepare a briefing for my meeting with: {contact_name}

                    Historical context:
                    {formatted_history}

                    Structure:
                    1. SUMMARY OF PAST INTERACTIONS
                    2. KEY REMINDERS
                    3. SUGGESTED CONVERSATION OPENERS
                """
            }
        ]
    )
    return response.choices[0].message.content
```

The structured output format is enforced through the prompt. Every briefing comes back as three scannable sections — no walls of text, no generic filler.

---

## Deployment on Render

I handled deployment. Render picks up Python projects automatically from GitHub — connect the repo and it installs everything from `requirements.txt`.

The critical step that is easy to miss: environment variables from your local `.env` file do not transfer to Render automatically. Every key needs to be added manually in the Render dashboard under **Environment → Add Environment Variable**:

```
GROQ_API_KEY
HINDSIHT_API_KEY
HINDSIHT_BANK_ID
```

Skip this and the app deploys silently with no API access. Add them before the first deploy.

Live at: **https://meetmind-iatt.onrender.com**

---

## The Moment Memory Changes Everything

The most compelling demonstration of MeetMind is watching the briefing quality shift after the first saved note.

**First interaction — empty memory:**
The agent returns generic guidance. "This appears to be your first meeting with this person. Approach with an open mind." Correct but useless.

**After one saved note:**
```
"Rahul needs a React website. Budget ₹50,000. Follow up Monday."
```

**Second interaction — memory active:**
The agent references the website, the budget, the deadline. It suggests specific conversation openers based on where things were left. It reads like briefing notes from an assistant who was there.

The AI model did not change. The prompt did not change. Only the memory changed — and that made everything different.

---

## What Went Wrong (And How We Fixed It)

**Model deprecation.** The Groq model we originally hardcoded was decommissioned while we were building. The error was cryptic. Fix: always check the provider's current supported models list. We switched to `llama3-70b-8192` and it worked immediately.

**Memory disappearing on restart.** Early versions stored notes in Python dictionaries — everything was lost every time the server restarted. Switching to Hindsight's persistent memory bank fixed this permanently.

**Silent API failures on Render.** Missing environment variables caused the deployed app to fail with no useful error message. Fix: add all API keys in the Render dashboard before first deploy, then check server logs if anything behaves unexpectedly.

---

## Lessons That Stuck

**The memory layer is the product.** The LLM is interchangeable. What makes MeetMind valuable is accumulated context over time. Without persistent memory, it is just another chatbot wrapper.

**Simplicity ships faster.** Plain Flask, plain HTML, one API per concern. Every framework you skip is a failure point you avoid.

**The demo story is as important as the code.** The before/after moment — no context versus full context — communicates the product's value in seconds. Build your demo around that moment.

---

## What Comes Next

MeetMind handles the recall problem well. The natural extensions from here:

- Automatic note extraction from meeting transcripts
- Calendar integration to generate briefings the night before scheduled calls
- Team-shared memory banks for entire sales or account management teams
- Memory recency scoring to show how fresh each recalled detail is

---

## Try It

**Live app:** https://meetmind-iatt.onrender.com

**Source code:** https://github.com/sickme78/MeetMind

**Hindsight memory layer:** https://hindsight.vectorize.io

**Vectorize agent memory:** https://vectorize.io/what-is-agent-memory

If you have ever wished you could walk into a conversation already knowing everything relevant — this is the tool that makes that possible.

---

*Vineeta Choudhary is a developer who builds practical, memory-powered AI tools for real-world problems.*
