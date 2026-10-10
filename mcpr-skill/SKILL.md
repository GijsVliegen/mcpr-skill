---
name: mcpr-skill
description: This skill should be used when working with R through the MCPR MCP server (tools execute_r_code, inspect_object, manage_r_sessions, show_plot and view), or when the user asks to run R code, inspect an R workspace, make plots, or analyse data in R. It covers two modes: co-working in the user's own live R console, and doing independent R work in the agent's own persistent private session. It also teaches how to keep printed output short to save context.
---

# MCPR: persistent R sessions for agents

Most agents run R statelessly: one `Rscript` call per command. That is a poor fit for serious data analysis, which is iterative and state-dependent. An analysis is a cumulative process of exploration, modelling and validation that can span hours or days. Intermediate steps can be computationally heavy, and a small downstream change such as a plot aesthetic would force a stateless agent to re-run the entire script.

MCPR removes this problem. It gives the agent a **persistent, interactive session inside a live R environment**. Objects, loaded packages, fitted models and expensive intermediate results survive between calls, so complex multi-step workflows can be built up incrementally and small changes only re-run the small piece that changed.

There are two ways to use it:

| Mode | Whose session | Use when |
|---|---|---|
| **A. Co-working** | The user's own console, which the agent attaches to | The user wants to see, share or build on the same workspace |
| **B. Private session** | The agent's own session, invisible to the user's workspace | The agent is doing its own analysis, experiments or heavy computation |

Default to the mode the user asks for. If they say "in my session" or "my workspace", use A. If they ask for analysis without referring to their session, use B.

## Tools

| Tool | Purpose |
|---|---|
| `execute_r_code(code)` | Run R code in the **currently active** session. Returns results, printed output, warnings and errors. Variables persist in the global environment across calls. |
| `inspect_object(name)` | Deep structural and statistical summary of one named object. Auto-detects data frames, vectors, lists, factors, matrices, functions, S3/S4/R6 objects, formulas, dates, environments. Use it to understand data structures, function definitions and models. |
| `manage_r_sessions(action, ...)` | Server-owned session control: list available sessions, attach a human session, start an MCPR-owned secondary session, detach back to the private session, close a secondary session. Use only when session management is actually needed. Ordinary tools take no session argument, they act on the active session. |
| `show_plot(plot, target)` | Display a plot. Pass the **name of an existing variable** (not R code) or a recorded plot reference such as `"plot:3"` or `"plot:last"`. `target = "user"` (default) shows it to the user; `target = "agent"` returns it as an image for the agent to look at. |
| `view(what, ...)` | Inspect session state: session info, terminal output, errors, warnings, packages, workspace files, search path, last computed value, help documentation. Use it for system and session state, and `inspect_object` for specific objects. |

### Rules that apply to every tool call

- **Output from `execute_r_code` goes to the agent only. The user cannot see it.** This includes `print()`, `cat()` and `message()`. Report findings to the user in chat.
- Keep code lean: no decorative prints, no banner headers, no comments that only restate the code. The code is not shown to the user as a report.
- **Plots are recorded automatically.** Every plot page drawn by `execute_r_code` is reported as `plot:N`. Show it with `show_plot(plot = "plot:N")` or `show_plot(plot = "plot:last")`. Do not wrap plotting code in `capture_plot()` when the plot is drawn through `execute_r_code`. Recorded plots re-render at any width and height.
- For a ggplot or other plot object, create it in one call and then show it by name:

  ```r
  execute_r_code("p <- ggplot(mtcars, aes(wt, mpg)) + geom_point()")
  show_plot(plot = "p")
  ```
- For base-graphics side-effect plotting that is not recorded, capture it first, then show it:

  ```r
  execute_r_code("base_plot <- MCPR::capture_plot(plot(cars))")
  show_plot(plot = "base_plot")
  ```
- When the agent itself needs to check how a plot looks (layout, overlaps, colours), use `show_plot(..., target = "agent")`, and only send it to the user once it looks right.

## Context control: never print full tables

This applies in both modes, because every printed row costs context.

Right after attaching or starting a session, run once:

```r
.mcpr_old_options <- options(
  tibble.print_max = 5,
  tibble.print_min = 5,
  pillar.print_max = 5,
  pillar.print_min = 5,
  width = 100
)
```

- Options only cover tibbles. Base `data.frame`s and `data.table`s have no such option, so **never print a full table: wrap it in `head(x, 5)`** or use `dplyr::slice_head(x, n = 5)`.
- Prefer compact summaries: `dim(x)`, `names(x)`, `str(x)`, `summary(x)`, `dplyr::glimpse(x)`, or `inspect_object("x")`.
- Apply the same care to long vectors, lists and model summaries.
- If the user asks for more rows or the full table, do what they ask. The limit is a default for the agent, not a restriction on the user.
- In the user's session (Mode A) this changes options in their live console. Tell them, and mention that `options(.mcpr_old_options)` restores the previous settings. In the private session (Mode B) no notice is needed.

---

# Mode A: co-working in the user's live R session

The user starts a listener in the R console they are already using, and the agent **attaches to that same session**. Agent and user then share one workspace: the user sees the objects the agent creates, and the agent sees what the user has loaded.

## A1. Attach to the user's own console first

1. Call `manage_r_sessions` with the list action to enumerate sessions.
2. Pick the session that belongs to the user's interactive console, which is the one that appeared after the user ran `mcpr_session_start()`. Attach to it with the attach action for a human session.
3. Do **not** silently work in the agent's private session when the user asked for co-working, and do not start a new session unless the user explicitly asks for a separate one.
4. If only the private session is listed, the user's console is not discoverable yet. Tell the user to run the following in their own R console, leave it open, then list again:

   ```r
   library(MCPR)
   mcpr_session_start()
   ```
5. If several sessions look plausible, ask which one they are working in instead of guessing.
6. After attaching, confirm it is the right one with `view` (session info) and tell the user briefly what you see, for example which objects exist, so they can verify you joined the correct console.
7. Run the context-control options from the section above.

## A2. Work as a careful co-worker

The workspace belongs to the user and they may be working in it at the same time.

- **Do not destroy their state.** Avoid `rm(list = ls())`, `setwd()`, restarting the session, detaching packages, or overwriting existing objects unless asked.
- **Name new objects distinctly**, for example with an `ai_` prefix, so they do not collide with the user's objects. Tell the user which objects you created.
- **Read before writing.** Look at existing objects with `view` and `inspect_object` before building on them.
- **Do more per call**, but keep each step inspectable and the printed output small.
- **Communicate through chat.** The user does not see tool output, so summarise results, state what changed in their workspace, and use `show_plot` (target user) for visuals they should see.
- **Report errors and warnings** instead of hiding them, and fix them with a minimal change.
- **Reproducibility:** use `view` on terminal output to see what has been run, and put the final working code in the user's script file if they ask for it.
- When finished, detach from the user's session so the agent no longer acts on it, unless the user wants to continue.

---

# Mode B: using a private R session for the agent's own work

By default the MCPR server runs a **private R session owned by the agent**. It is persistent, so it is the right place for exploration, modelling and validation that the agent drives itself, without touching the user's workspace.

## B1. When to use the private session

- The user wants analysis done and does not need to share a console.
- Experiments, trial models, scratch work or heavy computation that should not pollute the user's workspace.
- Long-running workflows where intermediate results must survive across many turns.
- Anything the agent would otherwise do with a fresh `Rscript` per command. Never fall back to repeated stateless `Rscript` calls when a session is available.

If the agent is currently attached to a user session, detach back to the private session before doing this kind of work (see `manage_r_sessions`). Use the secondary-session actions only if the work needs a **second** isolated session, and close it when done.

## B2. Exploit persistence: the workflow

The whole point is that state accumulates. Work in layers, and never recompute what already exists in memory.

1. **Load once.** Read data, load packages and set up options in the first calls. They stay available.
2. **Cache expensive results.** Store the output of heavy steps (large joins, fitted models, simulations, embeddings) as named objects and reuse them. Check `exists("ai_fit")` before refitting.
3. **Iterate on the cheap end.** Plot aesthetics, summaries, subsets and diagnostics are re-run against stored objects, not against the raw pipeline. Create the plot object once, then modify and redisplay it:

   ```r
   execute_r_code("p <- ggplot(ai_df, aes(x, y)) + geom_point()")
   show_plot(plot = "p")
   execute_r_code("p <- p + theme_minimal() + labs(title = 'Residuals')")
   show_plot(plot = "p")
   ```
4. **Build up in small inspectable steps.** Explore with `inspect_object`, model, then validate (residuals, cross-validation, sanity checks) in separate calls so each result can be checked before building on it.
5. **Checkpoint long work.** For analyses that span hours or days, save key objects to disk (`saveRDS`) so a session crash or restart does not lose them, and note the file path.
6. **Use `view` to orient.** After a long break or an error, check session info, workspace files, packages, warnings and the last value instead of guessing what state the session is in.

## B3. Keeping the private session healthy

- Check available packages with `view` before loading or installing anything.
- Do not `rm(list = ls())` as a habit: it throws away the persistence that makes this mode valuable. Remove specific objects when needed.
- Keep a clear naming scheme for objects so the state remains understandable after many steps. Use `view` or `inspect_object` to review the workspace.
- Warnings are reported in results. Treat them as signals, not noise, and resolve them.
- Because the user cannot see the private session, **tell them what was done**: which data was loaded, which objects exist, which models were fitted, and where any files were saved. Offer to export the final code as a script, or to hand results over to the user's own session.
- Visuals for the user go through `show_plot` with the default user target.

---

## Examples

Run code and keep output small:

```r
execute_r_code("
  library(dplyr)
  ai_res <- mtcars |> filter(mpg > 20) |> select(mpg, cyl, wt)
  nrow(ai_res)
  head(ai_res, 5)
")
```

Understand an object without printing it:

```r
inspect_object("ai_res")
```

Fit once, reuse many times:

```r
execute_r_code("ai_fit <- lm(mpg ~ wt + cyl, data = mtcars)")
execute_r_code("summary(ai_fit)$r.squared; head(residuals(ai_fit), 5)")
```

## Setup (when MCPR is not connected yet)

Install the package in the R installation the user's console uses:

```r
if (!require("remotes")) install.packages("remotes")
remotes::install_github("phisanti/MCPR")
```

Register the MCP server with the agent. The helper writes agent-specific config (supported agents: `claude`, `gemini`, `copilot`, `codex`):

```r
library(MCPR)
install_mcpr(agent = "copilot")
```

Manual server entry (Claude Desktop shown; other agents use their own config file with the same command and arguments):

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

If the server's R cannot see packages installed from the IDE console, set `R_LIBS_USER` in the `env` block of the MCP config. If R is not on the `PATH`, use the full path to the R binary as `command`. Restart the agent session after changing the config.

## Troubleshooting

- **Only a private session is listed (Mode A):** make sure `mcpr_session_start()` is running in the user's console. Set `MCPTOOLS_LOG_FILE` to a valid file path and inspect the log.
- **Tools not found:** check the config path, restart the agent, and verify the MCP server starts on its own.
- **Plot not visible to the user:** plots drawn through `execute_r_code` are only recorded, not shown. Call `show_plot`. For side-effect base graphics use `MCPR::capture_plot()` first.
- **Output seems missing for the user:** that is expected. Tool output is agent-only, so summarise it in chat.
- **Agent is acting in the wrong session:** use `view` (session info) to check, then list and switch with `manage_r_sessions`, and re-run the context-control options.

If problems persist, open an issue on https://github.com/phisanti/MCPR with logs and context.
