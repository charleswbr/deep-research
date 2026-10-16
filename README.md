# Deep Research

An attempt to reproduce OpenAI's Deep Research when it launched (February 2025).
I built it during Entrepreneur First's Builders Retreat: you give it a research
question in the terminal, it asks you clarifying questions, plans a report,
searches the web, reads the relevant pages and writes a long, cited Markdown
report. It is built on the OpenAI Agents SDK (`openai-agents`), with several
agents handing work to each other.

Status: v1 prototype. It works end to end, with known improvements still to
make, and it is not maintained.

## How it works

```
your query
    |
    v
Enhancer Agent (o3-mini) <--> qa tool: asks you clarifying questions
    |  research directive
    v
Brain Agent (o3-mini), the orchestrator
    |-- planner_agent (gpt-4o-mini): report outline + search queries
    |-- retriever_agent (gpt-4o-mini), once per query
    |       |-- web_research: DuckDuckGo results (title, URL, snippet)
    |       '-- fetch_tool: Playwright page fetch -> Markdown
    |                       -> Gemini 2.0 Flash extracts the relevant part
    |
    '-- handoff --> Writer Agent (gpt-4o): final Markdown report
```

1. **Enhancer Agent** (`agents_list/enhancer_agent.py`, `o3-mini`) analyses your
   query. If anything is ambiguous it calls the `qa` tool
   (`tools/q_and_a_tool.py`), which prints its questions in the terminal and
   reads your answers. It can loop until the query is clear, then outputs a
   detailed research directive.
2. **Brain Agent** (`agents_list/brain_agent.py`, `o3-mini`) runs the research
   loop. It uses the Planner and Retriever agents as tools and hands off to the
   Writer when every search is done.
3. **Planner Agent** (`agents_list/planner_agent.py`, `gpt-4o-mini`) returns
   JSON with a report skeleton (headings only) and 5 to 10 search queries. The
   Brain calls it again to revise the outline when new findings change the
   picture; searches already run are marked as done.
4. **Retriever Agent** (`agents_list/retriever_agent.py`, `gpt-4o-mini`) takes
   one query, gets results with `web_research`, judges each snippet, and calls
   `fetch_tool` on the relevant URLs. It returns JSON mapping each URL to the
   content found there.
   - `web_research` (`tools/web_research.py`) scrapes DuckDuckGo's HTML results
     page with `requests` and BeautifulSoup (1 to 10 results).
   - `fetch_tool` (`tools/fetch_tool.py`) loads the page in headless Chromium
     with Playwright, converts it to Markdown, cleans it
     (`utils/markdown_helpers.py`), and asks Google's `gemini-2.0-flash` to keep
     only the content relevant to the query.
5. **Writer Agent** (`agents_list/writer_agent.py`, `gpt-4o`) writes a formal,
   academic report in Markdown from the outline and the retrieved content
   only, citing sources inline as `<CIT url='...'>...</CIT>`.

All system prompts are in `system_prompts.py`. The final report is printed to
the terminal; it is not saved to a file.

Other files: `tools/google_tool.py` is a Playwright Google search scraper that
the agents don't currently use (DuckDuckGo replaced it). `test.py` is a scratch
script that runs the Retriever Agent alone on a query.

## Setup

The repo doesn't pin a Python version; the OpenAI Agents SDK needs Python 3.9
or later.

```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install google-genai      # used by tools/fetch_tool.py, missing from requirements.txt
playwright install chromium   # browser used by fetch_tool
```

Create a `.env` file in the repo root (it is git-ignored) with:

| Variable         | Used by                                                  |
| ---------------- | -------------------------------------------------------- |
| `OPENAI_API_KEY` | The OpenAI Agents SDK, for every agent (read by the SDK) |
| `GOOGLE_API_KEY` | `fetch_tool`, for the Gemini API                         |

`.env` is loaded with `python-dotenv` when `tools/fetch_tool.py` is imported.

## Run

```bash
python main.py
```

Type your research query (several lines are fine), then press Enter on an empty
line to submit. Answer the Enhancer's questions the same way, ending with an
empty line. Type `exit` at the first prompt to quit. A full run makes many model
calls and page fetches, so it takes a while.
