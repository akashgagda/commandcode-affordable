# commandcode-affordable

a Hermes Agent profile for [CommandCode](https://commandcode.ai/) users who want to keep spend low.

v0.1.1. prices and model ids below were read on **2026-10-05** from
`https://api.commandcode.ai/provider/v1/models` and `https://commandcode.ai/models`.
CommandCode changes its catalog and its deals, so check those two pages before
relying on any number here.

## why this profile

Hermes runs side tasks on "auxiliary" models: describing images, compressing long
sessions, naming sessions, and classifying risky shell commands. with the default
`provider: auto`, each of these runs on your main model
([docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration#auxiliary-models)).
on an expensive main model, every screenshot and every compression is billed at
that model's rate.

this profile pins each side task to a cheap paid CommandCode model. the
main model is cheap too. if you switch to a stronger model with `/model` for a
hard turn, the side tasks stay on their own models and their cost does not change.

## install

```bash
hermes profile install /path/to/commandcode-affordable --alias
commandcode-affordable setup     # pick "CommandCode" and paste your COMMANDCODE_API_KEY
commandcode-affordable chat
```

or, once this profile lives in a git repo of its own:

```bash
hermes profile install github.com/akashgagda/commandcode-affordable --alias
```

CommandCode API access needs a plan that exposes the API — the Provider (pay as
you go, $15/mo) or GOAT/Pro/Max. the $1 Go plan has no API access.

## what is set

- main model: `deepseek/deepseek-v4.1-flash`. **$0.15 in / $0.60 out per million
  tokens, cache read $0.003, 1M context**, and 39.5 on CommandCode's published
  Intelligence Index — the best scored model at that price.
- vision: `deepseek/deepseek-v4-flash-vision-exp`. **$0.15/$0.60, cache read
  $0.003, 1M context**, the only sub-$0.20 vision lane in the catalog. an explicit
  vision model also sends images through this describer when the main model has
  native vision ([docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/vision)).
  falls back to `google/gemini-3.5-flash-lite`, then `google/gemini-3.1-flash-lite`.
- compression: `deepseek/deepseek-v4.1-flash` at low reasoning. the compression
  summary is what the session remembers of itself, so this slot buys fact
  retention instead of the cheapest lane available.
- approval classifier: `z-ai/glm-5.3-flash` at low reasoning. **$0.15/$0.50**, and
  the highest published Intelligence Index (41.8) of any cheap model in the
  catalog — this is the slot where a miss is expensive.
- titles, skills hub, MCP dispatch, profile descriptions, mail scoring, TTS audio
  tags and memory query rewrite: `gpt-6-luna` — **$0.10 in / $0.50 out per million,
  cache read $0.01**, the cheapest input price of any scored model in the catalog.
  these calls turn a few thousand input tokens into a handful of output tokens, so
  input price dominates their bill.
- kanban triage (`triage_specifier`) writes a spec rather than a few tokens, so it
  gets `z-ai/glm-5.3-flash` instead.
- every pinned slot carries a paid model behind it: `z-ai/glm-5.3-flash` for the
  `gpt-6-luna` slots, `deepseek/deepseek-v4.1-flash` for `triage_specifier`.
- **no free or stealth lanes.** the free tier is not used anywhere (see changelog).
- `display.show_cost: true`, so spend shows in the status bar.

these stay on your main model on purpose: `background_review` (it writes your
memory and skills), `curator`, `kanban_decomposer`, `goal_judge` and `review`.
they shape what the agent keeps and plans, so they get the stronger model.

### the peak-hour trap

CommandCode bills the **DeepSeek family by time of day**, and the off-peak rate is
the one you see on the models page. peak is **01–04 and 06–10 UTC, Monday to
Friday only**, at 2×:

| band | hours | input $/M | output $/M |
|---|---|---|---|
| off-peak | 17h/day | $0.15 | $0.60 |
| peak | 01–04 & 06–10 UTC, Mon–Fri | $0.30 | $1.20 |

for IST that is **06:30–09:30 and 11:30–15:30, weekdays** — both right in the
working day. weekends are always off-peak. `z-ai/glm-5.3-flash`, `gpt-6-luna`,
`Qwen/Qwen3.8-Flash` and `xiaomi/mimo-v2.6-flash` are flat-priced, so they are not
affected; only the DeepSeek lanes (main, vision, compression) are.

if you want a flat main model instead, `/model` over to `z-ai/glm-5.3-flash`
($0.15/$0.50 flat, 1M context) — the side tasks do not move.

## keep your own profile instead

installing a distribution replaces that profile's config. to keep your current
setup and only move the side tasks, run these against your own profile:

```bash
hermes config set model.provider commandcode
hermes config set model.default deepseek/deepseek-v4.1-flash
hermes config set auxiliary.vision.provider commandcode
hermes config set auxiliary.vision.model deepseek/deepseek-v4-flash-vision-exp
hermes config set auxiliary.compression.provider commandcode
hermes config set auxiliary.compression.model deepseek/deepseek-v4.1-flash
hermes config set auxiliary.approval.provider commandcode
hermes config set auxiliary.approval.model z-ai/glm-5.3-flash
hermes config set auxiliary.title_generation.provider commandcode
hermes config set auxiliary.title_generation.model gpt-6-luna
hermes config set auxiliary.skills_hub.provider commandcode
hermes config set auxiliary.skills_hub.model gpt-6-luna
hermes config set auxiliary.memory_query_rewrite.provider commandcode
hermes config set auxiliary.memory_query_rewrite.model gpt-6-luna
```

(or `hermes model` → "Configure auxiliary models" for the interactive form —
[docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration#configuring-auxiliary-models-interactively).
fallback chains have no `config set` shorthand; edit `config.yaml` for those.)

## how the models were picked — and what was not measured

**measured:** nothing was spent. no live CommandCode calls were made from this
profile, so there is no per-call cost or accuracy figure here, unlike a profile
that has been run against its provider. what follows is what was actually checked
on 2026-10-05:

- **ids and context windows** read from the public models endpoint,
  `https://api.commandcode.ai/provider/v1/models` (85 models), so every id in
  `config.yaml` is the id the API itself returns.
- **prices and Intelligence Index scores** read from the models table and the
  individual model pages on `https://commandcode.ai/models`. the scores quoted
  (39.5, 41.8, 34.8) are CommandCode's own published numbers.
- **no free lanes**: CommandCode lists four models as free "while it lasts"
  (`inclusionai/ling-3.1-flash:free`, `inclusionai/ling-3.0-flash-sante:free`,
  `stealth/space-bunny-alpha`, `poolside/laguna-s-2.1-free`). none is used here. a
  lane that can vanish, throttle, or change behaviour without notice is a silent
  failure in a session title, a memory query or an approval check, and the saving
  is a fraction of a cent per call. every slot names a paid model.
- **resolution**: the profile was installed with `hermes profile install` (v0.21.5)
  and every pinned auxiliary task was read back with `hermes config get
  auxiliary.<task>.model`, which returns the value Hermes' own config resolver
  uses. `hermes status` reports the profile's main model and provider correctly.
  every model id in `config.yaml` was then diffed against the live
  `provider/v1/models` catalog: all 6 ids are served, and the context windows
  match what is claimed above. that is an integration check, not a model-quality
  test. (`auxiliary.<task>.fallback_chain` is a list, so `config get` does not
  recognise it as a leaf key and prints a warning — the key is read at runtime by
  `_try_configured_fallback_chain` in `agent/auxiliary_client.py`.)

**not measured:** whether `gpt-6-luna` actually names a session well,
whether `glm-5.3-flash` classifies risky shell commands as reliably as its index
score suggests, and whether `deepseek-v4-flash-vision-exp` reads an invoice
correctly. the slots were assigned by published price and published score, not by
running the tasks. a measured pass — a small vision and approval suite, a few
cents of credits — would replace this section with real results; ask if you want
that run before trusting the pins.

## changelog

### v0.1.1 — 2026-10-05

- **removed every free and stealth lane.** `title_generation`, `skills_hub`, `mcp`,
  `profile_describer`, `monitor`, `tts_audio_tags` and `memory_query_rewrite` moved
  from `inclusionai/ling-3.1-flash:free` to `gpt-6-luna`; `triage_specifier` moved
  to `z-ai/glm-5.3-flash`. each keeps a paid fallback.
- why: a free lane is free "while it lasts". when it ends or throttles, the slot
  fails or silently degrades, and the symptom is a missing title or a skipped
  skills lookup rather than an error anyone notices. the saving was a fraction of a
  cent per call. a paid model with a documented price is worth more than a free one
  with an unknown lifetime.

### v0.1.0 — 2026-10-05

- first version: main model and every auxiliary slot pinned, with the eight
  high-volume slots riding CommandCode's free tier.

## licence

MIT. see LICENSE.
