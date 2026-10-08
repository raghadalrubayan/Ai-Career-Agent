# Career Research Agent (LangGraph)

A multi-agent pipeline that researches the Saudi tech job market, compares it with **your own CV**
(RAG), and writes a personalized **3-month action plan**.

Built for the SDAIA Academy "Deep Research Agent" project: `create_agent` agents chained with LangGraph's
`StateGraph`, with tracing, loop detection and a quality retry loop on top.

## Architecture

```
START -> researcher --(new research)--> analyst -> writer --(plan is specific)--> END
              ^   \--(research repeated)-------------> writer      |
              |                                                    |
              +------------ (plan not specific: retry) ------------+
```

| Stage | Tools | Notes |
|-------|-------|-------|
| researcher | pre-search (code runs 2 web searches first), `search_web` (Tavily, DuckDuckGo fallback), `read_webpage` | step budget: 10 tool calls |
| analyst | `search_my_cv` (RAG over your CV PDF) | checkpointer memory (`InMemorySaver`), step budget: 10 |
| writer | none | outputs `NEEDS_SPECIFIC_TOOLS` when the evidence is too thin, which triggers the retry loop (max 1 retry) |

### Why a pre-search?
Some models (especially small/free ones) skip tool calls and answer from memory. The pipeline therefore runs the
first web searches itself and passes the results to the researcher, so the plan is grounded in sources even when
the model does not call tools. The run summary reports whether the research was grounded.

### Reliability features
- **Conditional routing + quality loop:** the writer can send the task back to the researcher; if the retry
  research is almost identical to the first one (stagnation), re-analysis is skipped and the writer finishes
  with a visible warning.
- **LoopDetector, wired in:** repeated or near-identical tool calls are blocked *before* they run (the agent is
  told to change approach), and research output is checked for stagnation (Jaccard similarity).
- **Graceful failures:** search provider fallback, per-agent error isolation (`safe_run`), no pointless retry
  after an API error such as a rate limit, and a warning inside the plan when web search failed, an agent failed,
  or the output was cut off by `MAX_TOKENS`.
- **Observability:** trace tree with tokens, OpenRouter cost and tool-call count per agent run, and a run summary
  computed from the actual run (nothing is hard-coded).

## Example run (free model `nvidia/nemotron-3.5-lightning:free`)

```
Agent runs         : 3 (retries used: 0)
Tokens / cost      : 9951 / $0.0000      (free model)
Tool calls         : {'search_my_cv': 2, 'search_web(pre-search)': 2}
Web evidence       : 2 successful search(es) -> grounded in sources
Step budget        : researcher [2] / 10 per run, analyst 2 / 10 -> kept
Loop alerts        : 0
Output truncated   : no
```

## Requirements
- Python 3.11+ and [uv](https://docs.astral.sh/uv/) (for local runs), or just a Google account for Colab
- An OpenRouter API key (required) and a Tavily API key (recommended, free tier)
- Your CV as a text-based PDF (not a scanned image)
- The first run downloads a sentence-transformers embedding model; a full run took about 4 minutes in the example below

## Setup

### Google Colab (easiest)
1. Open `research_agent.ipynb` in Colab.
2. Add secrets (key icon, with *Notebook access* enabled):
   - `OPENROUTER_API_KEY` (required)
   - `TAVILY_API_KEY` (recommended, free at tavily.com: DuckDuckGo is often blocked in Colab)
3. Run all cells and upload your CV (PDF) when asked.

### Your own machine (uv)
```bash
uv sync
cp .env.example .env        # paste your keys
# put your CV next to the notebook as my_cv.pdf
uv run jupyter lab research_agent.ipynb
```

### Choosing a model
Set `MODEL_NAME` in the setup cell. The default is the free `nvidia/nemotron-3.5-lightning:free`
(on OpenRouter accounts without credits, free models are limited to 50 requests per day). Any tool-calling
model on OpenRouter works, for example `google/gemini-2.5-flash` with credits. A full run uses roughly 10 requests,
so run it once and avoid re-running cells needlessly.

## Troubleshooting
| Symptom | Fix |
|---------|-----|
| `SEARCH_UNAVAILABLE` / "Web search is NOT working" | Add `TAVILY_API_KEY` (with *Notebook access*), make sure `tavily-python` is installed, then re-run from the setup cell |
| `429 ... free-models-per-day` | The daily free limit is used up: wait for the reset, add credits, or change `MODEL_NAME` |
| Plan ends mid-way / "Output truncated: YES" | Raise `MAX_TOKENS` in the setup cell |
| `NameError` about a detector, tool or runner | A cell is out of date: use "Run all" from the first cell |
| "No text found in the CV" | Upload a text-based PDF, not a scan |

## Limitations
- The retry loop is implemented and routed, but it was not triggered in the example run (the first plan was already specific).
- Cost shows `$0.0000` with free models; token counts are always real.
- Search quality depends on Tavily/DuckDuckGo, and plans should be checked against real job postings before acting on them.

## Privacy
`my_cv.pdf`, `action_plan.md` and `.env` are in `.gitignore`. The generated plan contains details from your CV,
so clear that cell's output before sharing the notebook.

## Structure
```
.
├── research_agent.ipynb   # setup, observability, loop detector, tools, runner, RAG, prompts, agents, pipeline, checks
├── pyproject.toml         # dependencies (uv)
├── .env.example           # key template
├── .gitignore
├── uv.lock                # generated by `uv sync`
└── README.md
```

## How to submit
```bash
git add research_agent.ipynb README.md pyproject.toml .env.example .gitignore uv.lock
git commit -m "Finish research agent project"
git push
```

Submitted by: Raghad Alrubayan — academy: @SDAIAAcademy
