<div align="center">
  <img src="logo.svg" width="72" alt="OpalZero" />
  <h1>OpalZero</h1>
</div>

**A self-hosted multi-agent intelligence kernel.** Give it a plain-English intent; it plans, dispatches specialist agents, validates the output, and streams structured results back to your application in real time.

**[→ Official website & docs](https://albertobarnabo.com/opal-zero/)**

---

## What it does

You send one sentence. OpalZero takes it from there:

1. **Planner** breaks the intent into a dependency-ordered task graph
2. **Dispatcher** assigns each task to a specialist agent (WebSearcher, Analyst, Coder, …)
3. **Governor** scores every result across five quality criteria — and injects new tasks if the bar isn't met
4. **ContextBus** aggregates all outputs into a typed, structured `MissionState`
5. Everything streams back to your app over SSE, event by event

```
intent  ──▶  Planner  ──▶  Dispatcher  ──▶  Agents
                                               │
                                           Governor  ◀── quality gate
                                               │
                                           ContextBus
                                               │
              your app  ◀── SSE stream ◀───────┘
```

The result is not a chat response. It is a **structured data payload** — a map of named values (metrics, tables, timelines, images) that your frontend can render directly, export as CSV/Markdown/HTML, or feed into the next operation.

---

## Five minutes to first result

**1. Start the server:**

With OpenAI:

```bash
docker run \
  -e OPENAI_API_KEY=sk-... \
  -e TAVILY_API_KEY=tvly-... \
  -p 8000:8000 \
  ghcr.io/albertobarnabo/opalzero-server:latest
```

Or fully local with Ollama (no API key required):

```bash
ollama pull llama3.1:8b   # any model that supports tool calling

docker run \
  -e AXION_PROVIDER=ollama \
  -e OPALZERO_MODEL=llama3.1:8b \
  -p 8000:8000 \
  ghcr.io/albertobarnabo/opalzero-server:latest
```

**2. Install the SDK:**

```bash
npm install opal-zero
```

**3. Run a mission:**

```ts
import { OpalZeroClient } from "opal-zero";

const client = new OpalZeroClient({ baseUrl: "http://localhost:8000" });

for await (const event of client.execute("Compare the top 3 EVs under $60k")) {
  if (event.type === "task_completed")  console.log(`✅ ${event.slug}:`, event.result);
  if (event.type === "mission_complete") console.log("Result:", event.mission_state);
}
```

**Or with React:**

```tsx
import { useOpalZero } from "opal-zero/react";

const { run, status, cards, activeAgent } = useOpalZero({ client });

// cards is BentoCard[] — typed, ready to render, no parsing needed
```

---

## Key capabilities

| Capability | Description |
|---|---|
| **Autonomous planning** | The Planner determines the task graph from your intent — no templates, no predefined flows |
| **Specialist agents** | WebSearcher, Analyst, Coder, and custom agents implemented as WASM modules |
| **Live quality gate** | The Governor validates every agent result and expands the plan mid-run if coverage is insufficient |
| **Structured output** | Results come back as typed data (`MissionState`), not unstructured text |
| **Mission refinement** | Deepen any completed mission with a follow-up intent; new data merges into the existing result |
| **Human-in-the-loop** | Agents can pause and ask for clarification; your app answers and execution resumes |
| **File context** | Upload CSV, JSON, PDF, or images; agents can reference them during execution |
| **Export** | Any mission can be exported as Markdown, CSV, or HTML |
| **Multi-provider** | Run on OpenAI, Anthropic Claude, Ollama local models, or any OpenAI-compatible endpoint (Groq, Mistral, Together…) |
| **Self-hosted** | Runs as a single Docker container; your data never leaves your infrastructure |

---

## The agent tool belt

Agents have access to a registry of composable tools, compiled to WASM:

| Tool | What it does |
|---|---|
| `web_search` | Real-time web search via Tavily |
| `fetch_page` | Fetch and parse any URL |
| `rss_reader` | Read RSS/Atom feeds |
| `vision` | Analyse images (via OpenAI Vision) |
| `python_interpreter` | Execute sandboxed Python |
| `calculator` | Evaluate mathematical expressions |
| `read_csv` | Parse and summarise CSV files |
| `sqlite_query` | Run SQL against an in-memory SQLite DB |
| `extract_pdf_text` | Extract text from uploaded PDFs |
| `http_request` | Make arbitrary HTTP calls |
| `diff` | Compare two text blocks |
| `send_email` | Send SMTP email |
| `memory` / `memory_persist` | Short-term and persistent agent memory |
| `get_price_history` | Stock price history (Alpha Vantage) |
| `get_income_statement` | Company financials (Alpha Vantage) |
| `get_news_sentiment` | Market news sentiment (Alpha Vantage) |
| `get_company_overview` | Company overview (Alpha Vantage) |
| `generate_document` | Produce structured document output |
| `build_dynamic_ui` | Emit layout hints and design tokens for the frontend |
| `finalize_mission_state` | Write the final typed result payload |
| `feedback` | Pause execution and request human input |

New tools are added as WASM modules — no server recompilation needed.

---

## Ecosystem

| Repo | Access | Contents |
|---|---|---|
| [opal-zero-engine](https://github.com/albertobarnabo/opal-zero-engine) | Public | HTTP server, REST API, SSE streaming layer |
| [opal-zero-kernel](https://github.com/albertobarnabo/opal-zero-kernel) | Private | Core reasoning engine, Governor, ContextBus |
| [opal-zero-professionals](https://github.com/albertobarnabo/opal-zero-professionals) | Public | WASM tool registry — the agent tool belt |
| [opal-zero](https://www.npmjs.com/package/opal-zero) | npm | TypeScript SDK (`npm install opal-zero`) |

The server binary (`opalzero-server`) is published as a Docker image on GHCR:

```
ghcr.io/albertobarnabo/opalzero-server:latest
```

---

## SDK at a glance

Full SDK documentation lives on the [npm package page](https://www.npmjs.com/package/opal-zero).

```ts
const client = new OpalZeroClient({
  baseUrl:          "http://localhost:8000",
  apiKey?:          string,   // X-OpalZero-Key — omit for local dev
  openAiKey?:       string,   // per-request OpenAI key override
  tavilyKey?:       string,   // enables web search for this request
  alphaVantageKey?: string,   // enables financial tools for this request
});

// Execute a mission
client.execute(intent, model?)              // → AsyncGenerator<MissionEvent>

// Manage missions
client.missions.list()                      // → MissionSummary[]
client.missions.get(id)                     // → MissionSnapshot
client.missions.refine(id, intent, model?)  // → AsyncGenerator<MissionEvent>
client.missions.export(id, format)          // → Blob  ("md" | "csv" | "html")
client.missions.delete(id)                  // → void

// Files & config
client.upload(file)                         // → UploadResult
client.configStatus()                       // → ConfigStatus
```

---

## Provider backends

OpalZero supports multiple AI backends. Switch with `AXION_PROVIDER` — no code changes required.

**OpenAI (default)**

```bash
AXION_PROVIDER=openai OPALZERO_MODEL=gpt-4o-mini OPENAI_API_KEY=sk-... cargo run --bin opalzero-server
```

**Ollama — fully local, no API key**

```bash
ollama pull llama3.1:8b
AXION_PROVIDER=ollama OPALZERO_MODEL=llama3.1:8b cargo run --bin opalzero-server
```

OpalZero requires a model that supports tool calling. Recommended:

| Model | Size | Notes |
|---|---|---|
| `llama3.1:8b` | 4.7 GB | Best balance of speed and quality |
| `mistral-nemo` | 7.1 GB | Strong reasoning, great for Analyst tasks |
| `qwen2.5:7b` | 4.7 GB | Fast, reliable tool-call support |

**Anthropic Claude**

```bash
AXION_PROVIDER=claude OPALZERO_MODEL=claude-sonnet-4-5 ANTHROPIC_API_KEY=sk-ant-... cargo run --bin opalzero-server
```

Haiku is automatically used for cheaper sub-tasks while your selected model handles planning and analysis.

**Any OpenAI-compatible endpoint** (Groq, Together, Mistral, LM Studio…)

```bash
AXION_PROVIDER=compatible \
  AXION_BASE_URL=https://api.groq.com/openai/v1 \
  OPALZERO_MODEL=llama-3.3-70b-versatile \
  AXION_API_KEY=gsk_... \
  cargo run --bin opalzero-server
```

---

## Self-hosting

OpalZero is designed to run on your own infrastructure. The server is a single Rust binary wrapped in a minimal Docker image — no external database, no telemetry, no callbacks home. Missions and uploads persist to local volumes you control.

```yaml
services:
  opalzero:
    image: ghcr.io/albertobarnabo/opalzero-server:latest
    ports: ["8000:8000"]
    environment:
      # Provider — "openai" (default) | "claude" | "ollama" | "compatible"
      AXION_PROVIDER: ${AXION_PROVIDER:-openai}
      OPALZERO_MODEL: ${OPALZERO_MODEL:-gpt-4o-mini}
      # Keys — only set the one(s) your provider needs
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY:-}
      AXION_API_KEY: ${AXION_API_KEY:-}         # for "compatible" endpoints
      AXION_BASE_URL: ${AXION_BASE_URL:-}       # for "compatible" endpoints
      TAVILY_API_KEY: ${TAVILY_API_KEY:-}       # enables web search
      OPALZERO_API_KEY: ${OPALZERO_API_KEY:-}   # optional inbound auth
    volumes:
      - missions:/app/missions
      - uploads:/app/uploads
```

---

## License

MIT
