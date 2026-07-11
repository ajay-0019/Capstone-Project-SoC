# 🧭 Planner Agent

An autonomous AI agent that takes one big goal, breaks it into ordered steps, executes them one by one using live web search, and saves its entire state to disk — so if you shut it down halfway through and come back tomorrow, it picks up exactly where it stopped.

---

## 📌 The Problem

You set a big goal. Life happens. You lose track. You come back and have no idea where you left off. So you start over.

Normal AI chatbots forget everything the moment you close them. There is no structure, no memory of what was already figured out, no way to resume. For multi-step goals that take research and multiple sessions to complete, that is a serious problem.

---

## 💡 The Solution

The Planner Agent fixes this with one core idea: **the file is the brain, not the program.**

You give it a goal. It breaks that goal into 4–6 ordered steps and writes them to `plan.json`. Then it works through each step one at a time — searching the web for real information, saving the result after every step, and marking it done before moving to the next. Kill the program mid-run and restart it tomorrow: it reads the file, finds the first unfinished step, and continues from exactly there.

---

## 🧠 Architecture

```
                    ┌──────────────────────────────────┐
                    │           plan.json (state)       │
                    │  goal, status, steps, results     │
                    └──────────────────────────────────┘
                          ▲                    │
                   write result           read next
                   mark done              pending step
                          │                    ▼
      ┌───────────────────────────────────────────────┐
      │              THE LOOP (stateless)              │
      │                                               │
      │  1. load plan.json                            │
      │  2. no plan? → make_plan(goal) → save         │
      │  3. find first step with status != done       │
      │  4. all done? → print summary, exit           │
      │  5. execute that ONE step (tools allowed)     │
      │  6. save result, mark done, advance step      │
      │  7. save plan.json → go to 1                  │
      └───────────────────────────────────────────────┘
```

Because state is flushed to disk after every step, a crash, a rate-limit, or Ctrl-C costs you at most one step of progress.

---

## 🏗️ The Four Organs

This project is the capstone of a structured AI/ML program. Each week added one capability:

| Week | Organ | What it gave the agent |
|------|-------|------------------------|
| 1 | Voice | A persona, structured output, JSON extraction |
| 2 | Hands | Tool calling, browser automation |
| 3 | Brain | ReAct reasoning loop, live web search |
| 4 | Self | Persistent memory, goal tracking across sessions |
| Capstone | Purpose | Task decomposition + state-driven orchestration |

---

## 📁 Project Structure

```
capstone/
├── capstone.py       # Entry point and agent persona
├── planner.py        # make_plan, execute_step, run_loop, Gemini calls
├── tools.py          # All tool functions and schemas
├── app.py            # Streamlit web interface
├── plan.json         # Agent state — starts as {}, grows during execution
├── memory.json       # Persistent facts — starts as []
├── goals.json        # Quest log — starts as []
├── requirements.txt  # Dependencies
└── .env              # API keys (never committed)
```

---

## ⚙️ How It Works

### Step 1 — Planning
`make_plan(goal)` sends the user's goal to Gemini and asks it to return a clean JSON list of 4–6 steps. Each step is a single, self-contained research or content task. The steps are written into `plan.json` with status `pending`.

### Step 2 — Execution Loop
`run_loop()` finds the first pending step and calls `execute_step()`. Inside each step execution, the agent can call tools — primarily `search_the_web` — to fetch real information. After the step is complete, the result is written back to `plan.json` and the step is marked `done`.

### Step 3 — Resumption
Every run starts by loading `plan.json`. If a plan is already `in_progress`, the loop skips completed steps and picks up from the first pending one. The Python process is completely disposable — all progress lives on disk.

### Step 4 — Persistent Memory
Facts the agent learns during execution are saved to `memory.json` via the `remember()` tool. Goals are tracked in `goals.json`. Both files are loaded at the start of every run.

---

## 🛠️ Tech Stack

| Component | Technology |
|-----------|-----------|
| LLM | Gemini 2.0 Flash (`google-generativeai`) |
| Web Search | DuckDuckGo Lite via `requests` + `BeautifulSoup` |
| State | Local JSON files (`plan.json`, `memory.json`, `goals.json`) |
| Frontend | Streamlit |
| Rate Limiting | Exponential backoff (2s → 4s → 8s → 16s) |

---

## 🚀 Setup and Running

### 1. Clone the repo
```bash
git clone https://github.com/ajay-019/Agentic-AI-and-GenAi
cd capstone
```

### 2. Install dependencies
```bash
pip install -r requirements.txt
```

### 3. Add your API key
Create a `.env` file in the project root:
```
GEMINI_API_KEY=your_key_here
```
Get a free key from [aistudio.google.com](https://aistudio.google.com)

### 4. Initialize state files
```bash
echo "{}" > plan.json
echo "[]" > memory.json
echo "[]" > goals.json
```

### 5. Run

**Terminal (CLI mode):**
```bash
python capstone.py
```

**Web interface:**
```bash
streamlit run app.py
```

---

## 🖥️ Example Run

```
==================================================
  Planner Agent
==================================================

Enter your goal: Create a 5-day travel plan for Mumbai

[Planning...]
[Plan created with 5 steps]

[Step 1] Research the top tourist attractions in Mumbai
  [Tool: search_the_web]
  [Result]: Mumbai's top attractions include the Gateway of India...

[Step 2] Create a Day 1 and Day 2 itinerary for South Mumbai
  [Result]: Day 1 - Morning: Gateway of India (9am)...

[Step 3] Create a Day 3 and Day 4 itinerary for Bandra and Juhu
  [Result]: Day 3 - Bandra Fort, Bandstand Promenade...

^C   ← killed here

==================================================
  Planner Agent
==================================================

[Resuming plan: Create a 5-day travel plan for Mumbai]
[Progress: 3/5 steps done]

[Step 4] Research the best local foods and where to find them
  [Tool: search_the_web]
  [Result]: Must-try foods include Vada Pav at Ashok Vada Pav...

[Step 5] Compile accommodation recommendations by area and budget
  [Result]: Budget: Colaba hostels from ₹800/night...

All steps complete.
==================================================
```

---

## 📊 plan.json Schema

```json
{
  "goal": "Create a 5-day travel plan for Mumbai",
  "status": "in_progress",
  "current_step": 3,
  "steps": [
    {
      "id": 1,
      "task": "Research top tourist attractions in Mumbai",
      "status": "done",
      "result": "Gateway of India, Marine Drive, Elephanta Caves..."
    },
    {
      "id": 2,
      "task": "Create Day 1 and Day 2 itinerary",
      "status": "done",
      "result": "Day 1 Morning: Gateway of India..."
    },
    {
      "id": 3,
      "task": "Create Day 3 and Day 4 itinerary",
      "status": "pending",
      "result": null
    }
  ]
}
```

---

## ✅ Features

- **Autonomous task decomposition** — one goal becomes an ordered step list automatically
- **Live web search** — steps fetch real information from the web, not cached model knowledge
- **Fault-tolerant resumption** — Ctrl-C, crash, or restart costs at most one step
- **Persistent memory** — facts saved to `memory.json` survive across sessions
- **Goal tracking** — quest log in `goals.json` tracks objectives across runs
- **Rate limit handling** — exponential backoff retries on API quota errors
- **Web UI** — Streamlit interface with progress bar, step viewer, and agent log
- **Bounded token usage** — each step only sends the current task and previous result, not the full plan

---

## 🔒 Security

- API key lives in `.env` only — never hardcoded
- `.env` is in `.gitignore` — never committed
- Run `git status` and confirm `.env` is not listed before every push

---

## 📈 What I Learned

The most valuable insight from building this: the best agents are not the ones with the biggest models. They are the ones that manage state well. When the agent's memory lives on disk instead of in RAM, it stops being a demo and starts being something you can actually rely on.

---

## 🔭 Future Extensions

- Email or WhatsApp notification when a long plan completes
- Parallel step execution for independent tasks
- A step dependency graph so steps can wait for each other
- Multi-user support with separate plan files per session
- Integration with Google Calendar to schedule planned tasks automatically

