---
name: mcpr-skill
description: This skill should be used when working with R through the MCPR MCP server (tools such as manage_r_sessions, execute_r_code, create_plot and view), or when the user asks to run R code, inspect their R workspace, make plots, or analyse data in their live R session. It teaches how to join the console the user is working in, how to co-work in it safely, and how to keep printed tables short to save context.
---

# MCPR: co-working in the user's live R session

MCPR (Model Context Protocol Tools for R, by phisanti) gives an agent a **persistent, stateful R session** instead of a fresh `Rscript` call per command. Objects, loaded packages and fitted models survive between calls, so analysis can be built up step by step.

The key idea: the user starts a listener in the R console they are already using, and the agent **joins that same session**. The agent and the user then share one workspace.

## Tools

| Tool | Purpose |
|---|---|
| `manage_r_sessions(action, session)` | Discover and connect to R sessions. Actions: `"list"` (enumerate active sessions), `"join"` (connect by session ID), `"start"` (launch a new private R session). |
| `execute_r_code(code)` | Run R code in the joined session. Returns results, printed output, warnings and errors. |
| `create_plot(expr, width, height, format)` | Run a plotting expression and return a base64 image. Uses `httpgd` when installed, otherwise standard graphics devices. |
| `view(what, max_lines)` | Inspect the session. `what` is one of `'session'` (object summaries), `'terminal'` (command history), `'workspace'` (file system context), `'installed_packages'`. |

## Step 1: Connect to the user's own console first

Because this is co-working, **always try to join the console the user is actively using** before doing anything else.

1. Call `manage_r_sessions("list")`.
2. Pick the session that belongs to the user's interactive console, which is the one that appeared after the user ran `mcpr_session_start()`. Join it with `manage_r_sessions("join", session = <ID>)`.
3. Do **not** use the private session that MCPR launches for the agent, and do **not** call `manage_r_sessions("start")`, unless the user explicitly asks for a separate session.
4. If the list shows only a private or local session, the user's console is not discoverable yet. Do not silently work in the private session. Tell the user to run this in their own R console, leave it open, then list again:

   ```r
   library(MCPR)
   mcpr_session_start()
   ```

5. If several sessions look plausible, ask the user which one they are working in rather than guessing.
6. After joining, confirm it is the right session with `view("session")` and mention to the user briefly what you see (for example the objects it contains), so they can tell you joined the correct console.

## Step 2: Limit table output to 5 rows (context control)

Immediately after joining, run this once with `execute_r_code` so printed tables stay short:

```r
.mcpr_old_options <- options(
  tibble.print_max = 5,   # tibbles print at most 5 rows...
  tibble.print_min = 5,   # ...and show 5 rows when truncated
  pillar.print_max = 5,
  pillar.print_min = 5,
  width = 100
)
```

Notes:

- This changes options in the **user's live session**. Tell the user you did it, and that `options(.mcpr_old_options)` restores the previous settings.
- Options only cover tibbles. Base `data.frame`s and `data.table`s have no "first 5 rows" print option, so also follow this rule: **never print a full table. Always wrap it in `head(x, 5)`**, or use `dplyr::slice_head(x, n = 5)`.
- Prefer compact summaries over printing data: `dim(x)`, `names(x)`, `str(x)`, `summary(x)`, `dplyr::glimpse(x)`.
- Apply the same care to long vectors and lists: print `head()` of them or summarise them.
- If the user asks for more rows or the full table, do what they ask. The 5-row limit is a default, not a restriction on the user.

## Step 3: Work as a careful co-worker

The workspace is shared with the user and they may be working in it at the same time.

- **Do not destroy their state.** Avoid `rm(list = ls())`, `setwd()`, restarting the session, detaching packages or overwriting existing objects unless asked.
- **Name new objects distinctly** (for example a `ai_` prefix) so they do not collide with the user's objects, and tell the user which objects you created.
- **Prefer reading before writing.** Look at the existing objects with `view("session")` before building on them.
- **Do more per call.** Multi-step code in one `execute_r_code` call is fine, but keep each step inspectable and keep the printed output small (see Step 2).
- **Report errors and warnings** that come back in the result instead of hiding them, and fix them with a minimal change.
- **Reproducibility:** use `view("terminal")` to see the command history, and put final, working code in the user's script file if they ask for it.

## Examples

Run code and keep output small:

```r
execute_r_code("
  library(dplyr)
  res <- mtcars |>
    filter(mpg > 20) |>
    select(mpg, cyl, wt)
  nrow(res)
  head(res, 5)
")
```

Make a plot:

```r
create_plot("
  library(ggplot2)
  ggplot(mtcars, aes(wt, mpg)) +
    geom_point() +
    geom_smooth(method = 'lm')
", width = 600, height = 450)
```

Check the packages available before loading something:

```r
view("installed_packages")
```

## Setup (when MCPR is not connected yet)

Install the package in the R installation the user's console uses:

```r
if (!require("remotes")) install.packages("remotes")
remotes::install_github("phisanti/MCPR")
```

Then register the MCP server with the agent. The helper writes the agent-specific config (supported agents: `claude`, `gemini`, `copilot`, `codex`):

```r
library(MCPR)
install_mcpr(agent = "copilot")
```

Manual form of the server entry (Claude Desktop shown; other agents use their own config file with the same command and arguments):

```json
{
  "mcpServers": {
    "mcpr": {
      "command": "R",
      "args": ["--quiet", "--slave", "-e", "MCPR::mcpr_server()"]
    }
  }
}
```

Approximate config locations: Claude Desktop `~/.config/Claude/claude_desktop_config.json` (Linux), `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS), `%APPDATA%\Claude\claude_desktop_config.json` (Windows); Gemini `~/.gemini/settings.json`; Copilot `.vscode/mcp.json`; Codex `~/.codex/config.toml`.

If the server's R cannot see packages installed from the IDE console, point it at the same library by setting `R_LIBS_USER` in the server's environment (the `env` block of the MCP config). If R is not on the `PATH`, use the full path to the R binary as `command`. Restart the agent session after changing the config.

## Troubleshooting

- **Connection failed / only a private session is listed:** make sure `mcpr_session_start()` is running in the user's console. Set the `MCPTOOLS_LOG_FILE` environment variable to a valid file path and inspect the log for details.
- **Tools not found:** check the config path is correct, the agent has been restarted, and the MCP server starts on its own.
- **Plotting errors:** check that the plotting expression is valid, that all needed libraries are loaded, and install `httpgd`.
- **Agent joined the wrong session:** list sessions again and join the interactive one, then re-run Step 2.

If problems persist, open an issue on https://github.com/phisanti/MCPR with logs and context.
