# Trading with AI Personas

A multi-agent stock trading simulation where four AI traders — each embodying a distinct investment philosophy inspired by legendary investors — autonomously research, decide, and execute trades using real market data.

Each trader runs as an independent AI agent powered by the [OpenAI Agents SDK](https://github.com/openai/openai-agents-python), communicates with external tools via [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) servers, and is monitored through a live Gradio dashboard.

---

## How It Works

Each trader agent follows this loop on a configurable schedule:

1. **Research** — A sub-agent searches the web (Brave Search), fetches pages, and draws on a persistent memory graph to find opportunities aligned with the trader's strategy
2. **Decide** — The trader agent reviews its account, holdings, and the research findings, then decides what to buy or sell
3. **Execute** — Trades are placed via the Accounts MCP server, which updates the SQLite database to track the transactions
4. **Notify** — A push notification summarising the session is sent via Pushover
5. **Alternate** — Traders alternate between "trade" mode (seeking new opportunities) and "rebalance" mode (reviewing existing holdings)

---

## Project Structure

```
Trading with AI Personas/
├── .env.example            # Template for required environment variables
├── .gitignore
├── pyproject.toml          # Project dependencies (uv)
└── src/
    │
    ├── app.py              # Gradio UI dashboard for traders portfolio
    ├── trading_floor.py    # Scheduler
    │
    │── Personas ──────────────────────────────────────────────────────
    ├── reset.py            # Defines each trader's investment strategy (persona prompt)
    ├── templates.py        # System prompts and message templates for all agents
    ├── traders.py          # Trader class builds and runs the agent for each persona
    │
    │── MCP Servers ────────────────────────────────────────────────────
    ├── mcp_params.py       # MCP server configuration for trader and researcher agents
    ├── accounts_server.py  # MCP server: buy/sell shares, get balance/holdings
    ├── market_server.py    # MCP server: look up share prices
    ├── push_server.py      # MCP server: send Pushover push notifications
    │
    │── Core ────────────────────────────────────────────────────────────
    ├── accounts.py         # Account model — balance, holdings, transactions, P&L
    ├── accounts_client.py  # MCP client helper to call the accounts server
    ├── database.py         # SQLite read/write for accounts, logs, and market data
    ├── market.py           # Polygon.io integration for share prices
    ├── tracers.py          # Custom tracing processor — writes agent spans to the DB log
    ├── util.py             # CSS, JS, and Color enum for the UI
    │
    ├── memory/             # Per-trader persistent memory
    └── accounts.db         # SQLite database
```

---

## The Four Trader Personas

Personas are defined across three files that work together:

| File | Role |
|---|---|
| `reset.py` | The **strategy prompt** — the core investment philosophy injected into each trader's account |
| `trading_floor.py` | **Identity** — assigns names, surnames, and LLM models to each trader |
| `templates.py` | **Behavioural instructions** — the system prompt and message format that shapes how the agent acts at runtime |

### Warren — *"Patience"*
> Inspired by Warren Buffett

A value-oriented investor focused on long-term wealth creation. Identifies high-quality companies trading below intrinsic value. Relies on fundamental analysis, steady cash flows, strong management, and competitive moats. Rarely reacts to short-term market noise.

### George — *"Bold"*
> Inspired by George Soros

An aggressive macro trader hunting significant market mispricings. Watches large-scale economic and geopolitical events for contrarian opportunities. Willing to bet boldly against prevailing sentiment when macro analysis signals a major imbalance.

### Ray — *"Systematic"*
> Inspired by Ray Dalio

A principles-based, systematic investor with a focus on diversification and risk parity. Invests across asset classes, monitors macroeconomic indicators and central bank policy, and adjusts the portfolio to manage risk across varying market cycles.

### Cathie — *"Crypto"*
> Inspired by Cathie Wood

An innovation-focused investor targeting disruptive technology and crypto ETFs. Accepts higher volatility for exceptional return potential. Closely tracks regulatory changes, tech breakthroughs, and market sentiment in the crypto space.

 Each trader can use a different model provider (GPT, DeepSeek, Gemini, Grok), giving each a subtly different reasoning style on top of their strategy.

---

## MCP Servers

The agents interact with the world exclusively through MCP servers, keeping the agent logic clean and tool logic isolated:

| Server | Transport | Tools exposed |
|---|---|---|
| `accounts_server.py` | stdio | `get_balance`, `get_holdings`, `buy_shares`, `sell_shares`, `change_strategy` |
| `market_server.py` | stdio | `lookup_share_price` (free tier fallback) |
| `push_server.py` | stdio | `push` (Pushover notification) |
| `mcp-server-fetch` | uvx | Fetch web pages |
| `@modelcontextprotocol/server-brave-search` | npx | Web search |
| `mcp-memory-libsql` | npx | Per-trader persistent knowledge graph |

For paid/realtime Polygon plans, `market_server.py` is replaced by the official `mcp_polygon` server (set `POLYGON_PLAN=paid` or `POLYGON_PLAN=realtime`).

---

## Setup

### Prerequisites
- Python 3.10+
- [uv](https://docs.astral.sh/uv/) — for running Python scripts and managing dependencies
- Node.js / npx — for the Brave Search and Memory MCP servers

### Install dependencies
```bash
cd src
uv sync
```

### Configure environment
```bash
cp src/.env.example .env
# Edit .env with your API keys
```

### Initialise traders
Run once to seed each trader's account with their strategy:
```bash
cd src
uv run reset.py
```

### Launch the dashboard
```bash
cd src
python app.py
```

### Start the trading scheduler
In a separate terminal:
```bash
cd src
python trading_floor.py
```