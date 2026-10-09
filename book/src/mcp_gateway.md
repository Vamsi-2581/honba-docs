# MCP Gateway

Honba exposes its capabilities through a Model Context Protocol (MCP)
server. This allows external AI agents — Claude Code, Cursor, Cline,
or a custom agent — to research, backtest, and validate strategies by
calling Honba tools directly.

The MCP gateway is the mechanism that turns Honba from a tool you use
into a tool that AI agents use.

## What MCP Is

Model Context Protocol is an open standard for connecting AI agents to
external systems. An MCP server exposes:

- **Tools** — functions the agent can call (e.g., `run_backtest`).
- **Resources** — data the agent can read (e.g., `catalog://nifty50`).
- **Prompts** — templates the agent can use.

The agent sees these as capabilities and decides when to use them.

## Why Honba Exposes MCP

An AI agent that can only chat is a general-purpose assistant. An AI
agent that can call `run_backtest` and `validate_strategy` is a
quantitative research assistant. The MCP gateway is what makes the
difference.

Common workflows this enables:

- "Find me a strategy on Alpha30 with Sharpe above 1.0 after costs."
- "Take this strategy I wrote and run the full validation suite."
- "Compare these three strategies across the last five years."
- "Explain why this backtest failed the walk-forward test."

## The Exposed Tools

### Backtest tools

| Tool | Description |
|---|---|
| `run_backtest` | Run Tier 2 backtest, return summary + tearsheet path |
| `run_vector_backtest` | Run Tier 1, fast, return summary |
| `run_concurrent_sweep` | Run Tier 3 parameter sweep |
| `run_monte_carlo` | Run Monte Carlo simulation |
| `run_walk_forward` | Run walk-forward validation |
| `validate_strategy` | Run full validation suite |

### Research tools

| Tool | Description |
|---|---|
| `list_universes` | Available universes (NIFTY 50, Alpha 30, etc.) |
| `universe_constituents` | Constituents of a universe on a date |
| `get_instrument` | Instrument metadata |
| `get_price_history` | Historical prices for an instrument |
| `get_market_calendar` | Trading days for a period |
| `get_cost_model` | Cost breakdown for a segment |

### Journal tools

| Tool | Description |
|---|---|
| `journal_search` | Semantic search over past hypotheses |
| `journal_list` | List hypotheses with filters |
| `journal_get` | Full hypothesis record |
| `journal_stats` | Aggregate statistics |

### Strategy tools

| Tool | Description |
|---|---|
| `list_strategies` | Strategies in the catalog |
| `get_strategy` | Full strategy code and metadata |
| `write_strategy` | Write a new strategy to disk |
| `run_strategy_test` | Unit test a strategy in isolation |

## Example Agent Session

An agent connected to the MCP server can hold this conversation:

    User: Find a momentum strategy on Alpha30 that survives validation.

    Agent: [calls list_universes()]
           → nifty50, banknifty, nifty200_alpha30, alpha30

    Agent: [calls universe_constituents(universe="nifty200_alpha30",
                                        date="2020-01-01")]
           → [RELIANCE, TCS, HDFCBANK, ...]

    Agent: [calls journal_search(query="momentum alpha30")]
           → 12 prior hypotheses, 2 validated

    Agent: [calls run_vector_backtest(
              strategy="momentum_top_n",
              universe="nifty200_alpha30",
              start="2020-01-01",
              end="2024-12-31")]
           → Sharpe 1.18 (Tier 1)

    Agent: [calls run_backtest(...)]           → Sharpe 1.02 (Tier 2)
    Agent: [calls run_walk_forward(...)]       → 6/8 folds positive
    Agent: [calls run_monte_carlo(...)]        → 5th pct Sharpe 0.61
    Agent: [calls validate_strategy(...)]      → PASSED

    Agent: The momentum strategy on Alpha30 passed all validation
           checks. Walk-forward showed 6 of 8 folds profitable,
           Monte Carlo 5th percentile Sharpe was 0.61, and the
           deflated Sharpe was 0.58. The full tearsheet is at
           ./reports/momentum_alpha30.html.

The agent orchestrates the tools. Honba does the computation.

## Running the Server

    honba mcp serve

By default, this starts a stdio server. For HTTP transport:

    honba mcp serve --transport http --port 8765

Configuration:

    # configs/ai/mcp.toml
    [server]
    transport = "stdio"       # or "http"
    name = "honba"
    version = "0.1.0"

    [tools]
    enable = ["backtest", "research", "journal", "strategy"]
    # Individual tools can be disabled for safety:
    disable = ["write_strategy"]

    [safety]
    max_backtests_per_session = 100
    max_sweep_combinations = 10_000
    require_confirmation_for = ["write_strategy"]

## Connecting an Agent

### Claude Code / Claude Desktop

Add to `claude_desktop_config.json`:

    {
      "mcpServers": {
        "honba": {
          "command": "honba",
          "args": ["mcp", "serve"],
          "cwd": "/path/to/honba-workspace/honba"
        }
      }
    }

### Custom Agents

Any MCP-compatible client can connect. The protocol is documented at
[modelcontextprotocol.io](https://modelcontextprotocol.io).

For Python agents:

    from mcp import ClientSession, StdioServerParameters
    from mcp.client.stdio import stdio_client

    server_params = StdioServerParameters(
        command="honba",
        args=["mcp", "serve"],
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tools = await session.list_tools()
            result = await session.call_tool(
                "run_backtest",
                arguments={
                    "strategy": "SmaCrossover",
                    "universe": "nifty50",
                    "start": "2020-01-01",
                    "end": "2024-12-31",
                },
            )

## Resources

Besides tools, the MCP server exposes resources that agents can read
directly:

    catalog://nifty50              — catalog metadata for NIFTY 50
    universe://nifty200_alpha30    — current constituents
    calendar://nse/2026            — trading days for 2026
    strategy://SmaCrossover        — strategy source and metadata
    journal://h_2024_03_17_001     — full hypothesis record
    config://backtest/default      — default backtest config

Resources are read-only and do not consume backtest budget.

## Prompts

Pre-built prompt templates are exposed for common workflows:

    prompt://research/hypothesis-generation
    prompt://research/strategy-critique
    prompt://research/regime-analysis
