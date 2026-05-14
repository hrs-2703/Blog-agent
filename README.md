# Blog Writing Agent

An AI-powered technical blog generator built with **LangGraph**, **Google Gemini**, and **Streamlit**. Give it a topic, and it autonomously plans, researches (if needed), writes every section in parallel, optionally generates diagrams, and delivers a finished Markdown blog post — all inside a live web UI.

---

## How It Works

The backend is a **LangGraph state machine** with the following pipeline:

```
Topic
  └─► Router ──────────────────────────────────────────────────┐
        │ needs_research?                                        │ no
       yes                                                       │
        │                                                        │
        ▼                                                        │
     Research (Tavily web search + Gemini synthesis)            │
        │                                                        │
        └──────────────────────────────────────────────────────►│
                                                                 ▼
                                                          Orchestrator
                                                     (creates structured Plan
                                                       with 5–9 tasks)
                                                                 │
                                                    ┌────────────┼────────────┐
                                                    ▼            ▼            ▼
                                                 Worker       Worker  ...  Worker
                                              (parallel section writing per task)
                                                    │            │            │
                                                    └────────────┴────────────┘
                                                                 │
                                                                 ▼
                                                     Reducer Subgraph
                                                   ┌─────────────────────┐
                                                   │  merge_content      │
                                                   │  decide_images      │
                                                   │  generate_and_place │
                                                   └─────────────────────┘
                                                                 │
                                                                 ▼
                                                    Final Markdown (.md file)
```

### Research Modes

| Mode | Triggered when | Recency window |
|---|---|---|
| `closed_book` | Evergreen concepts (e.g., "how TCP works") | No web search |
| `hybrid` | Mix of evergreen + recent tools/models | Last 45 days |
| `open_book` | News, "latest", weekly roundups | Last 7 days |

### AI Models Used

| Role | Model | SDK |
|---|---|---|
| All LLM calls (routing, planning, writing) | `gemini-2.5-flash` | `langchain-google-genai` |
| Image / diagram generation | `gemini-2.5-flash-image` | `google-genai` |
| Web search | Tavily Search API | `langchain-community` |

---

## Project Structure

```
Blog_ai1/
├── bwa_backend.py     # LangGraph graph: all nodes, schemas, image logic
├── bwa_frontend.py    # Streamlit UI
├── .env               # API keys (never commit this)
├── images/            # Auto-created; stores generated diagrams
└── *.md               # Auto-created; one file per generated blog
```

---

## Prerequisites

- **Python 3.9 or higher**
- A **Google AI Studio API key** — get one free at [aistudio.google.com](https://aistudio.google.com/apikey)
- *(Optional)* A **Tavily API key** for web research — [tavily.com](https://tavily.com)

---

## Setup — Step by Step

### 1. Clone / download the project

```bash
cd "path\to\Blog_ai1"
```

### 2. Create a virtual environment

```bash
python -m venv .venv
```

Activate it:

```bash
# Windows (PowerShell)
.venv\Scripts\Activate.ps1

# Windows (Command Prompt)
.venv\Scripts\activate.bat

# macOS / Linux
source .venv/bin/activate
```

### 3. Install dependencies

```bash
pip install streamlit langgraph langchain-core langchain-google-genai ^
            google-genai langchain-community python-dotenv pydantic pandas
```

Or install them all at once (copy as one line):

```bash
pip install streamlit langgraph langchain-core langchain-google-genai google-genai langchain-community python-dotenv pydantic pandas
```

### 4. Configure API keys

Create a `.env` file in the project root with the following content:

```env
GOOGLE_API_KEY=your_google_ai_studio_key_here
TAVILY_API_KEY=your_tavily_key_here   # optional — remove line if not using
```

> **Where to get keys**
> - `GOOGLE_API_KEY` → [https://aistudio.google.com/apikey](https://aistudio.google.com/apikey) (free tier available)
> - `TAVILY_API_KEY` → [https://tavily.com](https://tavily.com) (free tier: 1 000 searches/month)

> **Security note**: Never commit your `.env` file. Add it to `.gitignore` if using Git.

### 5. Run the app

```bash
streamlit run bwa_frontend.py
```

The app opens automatically at `http://localhost:8501`.

---

## Using the App

### Generate a blog

1. Type your topic in the **sidebar** text area. Examples:
   - `"How attention mechanisms work in transformers"`
   - `"Latest news in AI agents this week"`
   - `"Comparison: Redis vs Memcached for caching"`
2. Set the **As-of date** (defaults to today; affects news recency filtering).
3. Click **Generate Blog**.
4. Watch real-time progress in the status bar — each graph node is shown as it executes.

### Read the output

The UI has five tabs:

| Tab | Contents |
|---|---|
| **Plan** | Structured outline: blog title, audience, tone, kind, and a table of all tasks with word targets |
| **Evidence** | Web sources found and cited (only populated in hybrid / open_book mode) |
| **Markdown Preview** | Rendered blog post with inline images |
| **Images** | Generated diagrams/illustrations with download option |
| **Logs** | Raw LangGraph event stream for debugging |

### Download your blog

From the **Markdown Preview** tab:
- **Download Markdown** — just the `.md` file
- **Download Bundle (MD + images)** — a `.zip` containing the markdown and all generated images

### Load a past blog

Previously generated blogs are saved as `.md` files in the project folder. They appear in the **Past blogs** list in the sidebar. Click a title then **Load selected blog** to view it again without re-running the agent.

---

## Configuration Reference

### `.env` file

| Variable | Required | Description |
|---|---|---|
| `GOOGLE_API_KEY` | **Yes** | Used for Gemini 2.5 Flash (LLM) and Gemini 2.5 Flash Image (image generation) |
| `TAVILY_API_KEY` | No | Enables web research. Without it, the agent always runs in `closed_book` mode |

### Changing the LLM model

In `bwa_backend.py`, line 116:

```python
llm = ChatGoogleGenerativeAI(model="gemini-2.5-flash")
```

Replace `"gemini-2.5-flash"` with any model that supports function calling, e.g., `"gemini-2.5-pro"`.

### Changing the image model

In `bwa_backend.py`, inside `_gemini_generate_image_bytes()`:

```python
model="gemini-2.5-flash-image"
```

### Tuning blog length

Each task's `target_words` is set by the Orchestrator (120–550 words per section). To influence this, add explicit instructions in your topic string, e.g.:

> `"How diffusion models work — keep it concise, under 1000 words total"`

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| `GOOGLE_API_KEY` not found | `.env` missing or not in project root | Create `.env` with your key |
| No web research happens | `TAVILY_API_KEY` not set | Add key to `.env` or accept closed_book mode |
| Image generation fails | Google API quota / safety filter | The blog still saves; images show a fallback error block |
| `ModuleNotFoundError` | Missing package | Run the `pip install` command in step 3 again |
| Port 8501 already in use | Another Streamlit instance running | Run `streamlit run bwa_frontend.py --server.port 8502` |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Orchestration | [LangGraph](https://github.com/langchain-ai/langgraph) |
| LLM | [Google Gemini 2.5 Flash](https://ai.google.dev) via `langchain-google-genai` |
| Image generation | [Gemini 2.5 Flash Image](https://ai.google.dev) via `google-genai` |
| Web search | [Tavily](https://tavily.com) via `langchain-community` |
| UI | [Streamlit](https://streamlit.io) |
| Schema validation | [Pydantic v2](https://docs.pydantic.dev) |

