---
description: Build or refresh oh-my-opencode-slim presets from the live model catalog
agent: build
---

Build or refresh presets in `~/.config/opencode/oh-my-opencode-slim.json`.

Target: $ARGUMENTS

Format: `<provider>[+<provider>...] <tier-prefix | preset-name>`
Examples: `opencode-go+claude-agent gc` · `github-copilot p-hi`
Empty target = revalidate every existing preset without changing model choices.

## Steps

1. **Inspect.** Read the config. Note the active `preset`, existing families,
   `fallback`, `council.presets`, and `marketplace.agents_add`. Never touch
   presets outside the target.

2. **Catalog.** For each target provider, call the `models` tool once with
   `provider`, `all: true`, `limit: 100` (100 is the max). Record each model's
   id, supported variants, and cost.

3. **Capability.** Fetch `https://models.dev/api.json` and read per-model
   `modalities.input` (needs `"image"` for vision), `limit.context`, `reasoning`,
   `tool_call`. The `claude-agent` provider is absent there; treat its models as
   vision-capable.

4. **Design each agent by its binding constraint.**

   | Agent | Needs |
   |---|---|
   | orchestrator | tool-call discipline, long-horizon instruction following |
   | oracle | deepest reasoning, read-only |
   | explorer | cheap, fast, large context |
   | designer | vision and frontend taste |
   | fixer | precise, code-tuned diffs |
   | observer | **vision is mandatory** |
   | council | lab diversity over single-model strength |
   | paladin / hunter | adversarial reasoning |
   | arbiter | contract and correctness review |
   | janitor / chronicler | breadth and prose, not depth |

   Rules:
   - Every chain spans at least 2 providers, so one provider's rate limit
     degrades the chain instead of stalling it.
   - `observer` and `designer` chains must be 100% vision-capable.
   - Flat-rate subscriptions lead judgment seats; metered models lead fan-out.
   - Tiers differ by who leads, not only by model size.

5. **Hard constraints. These fail silently, so check every entry.**
   - A model whose variants list is empty must be a plain string, never
     `{id, variant}`.
   - A variant must appear in that model's own variants list. Examples:
     `deepseek-v4-pro` is `high|max` only, `minimax-m3` is `none|thinking`,
     `gemini-4-argon` has no `max`.
   - No non-vision model in a vision chain.

6. **Write.**
   - Back up first: `cp <config> <config>.bak.$(date +%Y%m%d-%H%M%S)`.
   - Edit with a throwaway `python3 -I` script that `json.load`s and
     `json.dump`s the whole file, so unrelated keys survive. Delete the script
     afterward.
   - Update `council.presets.<name>` alongside `presets.<name>`.

7. **Verify.** Do not report success before both pass:
   - Assert every model id exists in the catalog and every variant is legal.
   - Validate against the plugin schema:
     ```
     S=$(ls -d ~/.cache/opencode/npm/oh-my-opencode-slim@latest/*/node_modules/oh-my-opencode-slim | tail -1)
     npx --yes ajv-cli@5 validate --spec=draft2020 -s "$S/oh-my-opencode-slim.schema.json" -d ~/.config/opencode/oh-my-opencode-slim.json
     ```

8. **Report.**
   - A per-agent chain table.
   - The number of entries validated.
   - Any bugs fixed in the old presets.
   - Any spend increase (for example a higher variant on a default council).
   - Whether `preset` was left unchanged.
   - End with: "This should apply on the next OpenCode run; restart OpenCode if
     you need it immediately."

## Boundaries

- Do not change the active `preset` or `council.default_preset` unless asked.
- Do not commit or push; tell the user what changed.
- Report failures as failures.
