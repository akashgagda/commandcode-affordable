# commandcode-affordable

a Hermes Agent profile for [CommandCode](https://commandcode.ai/) users who want to keep spend low.

v0.1.6. prices and model ids below were re-read on **2026-10-06** from
`https://api.commandcode.ai/provider/v1/models` and `https://commandcode.ai/models`.
The pins themselves are v0.1.4's: v0.1.5 was released the same day and reverted
within the day (see the changelog). CommandCode changes its catalog and its deals,
so check those two pages before relying on any number here.

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

`--alias` must be passed at install time to get the shell wrapper; adding it later
is `hermes profile alias commandcode-affordable`.

## update

this is a distribution, so it updates itself:

```bash
hermes profile update commandcode-affordable
```

`update` replaces the files this distribution owns (`README.md`, `LICENSE`) and
**preserves `config.yaml`**, so any pins you edited yourself survive. the flip side
is that an upstream *pin* change will not reach you on a plain update:

```bash
hermes profile update commandcode-affordable --force-config
```

`--force-config` overwrites `config.yaml` with the shipped version, so your own pin
edits revert to the profile's defaults. memories, sessions, `.env` and credentials
are never touched by either form — that includes the API key `setup` wrote.

## what is set

- main model: `deepseek/deepseek-v4.1-flash-fast`. **$0.16 in / $0.58 out per
  million tokens, cache read $0.016, 1M context**, and 2× during the peak windows
  below. this is the *fast* tier of the DeepSeek V4.1 family, not the scored base
  model: CommandCode publishes no Intelligence Index for it. it is chosen here for
  latency, at the cost of a slightly higher input price and a 5× worse cache read
  than `deepseek/deepseek-v4.1-flash` ($0.15/$0.60, cache read $0.003, index 39.5).
- vision: `deepseek/deepseek-v4-flash-vision-exp`. **$0.15/$0.60, cache read
  $0.003, 1M context**, the only sub-$0.20 vision lane in the catalog. an explicit
  vision model also sends images through this describer when the main model has
  native vision ([docs](https://hermes-agent.nousresearch.com/docs/user-guide/features/vision)).
  falls back to `google/gemini-3.5-flash-lite`, then `google/gemini-3.1-flash-lite`.
- compression: `deepseek/deepseek-v4.1-flash-fast` at low reasoning. the compression
  summary is what the session remembers of itself, so this slot stays on the same
  DeepSeek V4.1 family as the main model instead of dropping to a cheaper lane.
- approval classifier: `z-ai/glm-5.3-flash` at low reasoning. **$0.15/$0.50, cache
  read $0.03, 1M context, published Intelligence Index 41.8**. v0.1.0–v0.1.3 used the
  *fast* tier `z-ai/glm-5.3-flashx` ($0.37/$1.25, cache read $0.07, **not scored**)
  and paid ~2.5× for latency alone; v0.1.4 takes the sibling instead, which is
  cheaper on all three rates *and* scored. the cost is latency — FlashX is the faster
  tier, so if approval lag ever shows up, FlashX is the deliberate step back up.
  worth knowing: against the `auto` behaviour it replaced (the main model,
  $0.16/$0.58 with cache read $0.016) Flash is cheaper on input and output but
  ~1.9× its **cache-read** rate off-peak, so whether this slot wins per call depends
  on how much of the classifier's prompt is a cache hit.
- titles, skills hub, MCP dispatch, profile descriptions, mail scoring, TTS audio
  tags and memory query rewrite: `gpt-6-luna` — **$0.10 in / $0.50 out per million,
  cache read $0.01**, the cheapest input price among scored models *worth using*
  here. these calls turn a few thousand input tokens into a handful of output
  tokens, so input price dominates their bill. (`stepfun/Step-3.5-Flash` is 10 %
  cheaper on input at $0.09, but scores 17.0 against Luna's 37.3 — a title or a
  memory-search string written by a 17-index model is not worth the cent.)
- kanban triage (`triage_specifier`) writes a spec rather than a few tokens, so it
  gets the same `z-ai/glm-5.3-flash` as approval.
- every pinned slot carries a paid model behind it: `z-ai/glm-5.3-flash` for the
  `gpt-6-luna` slots, `deepseek/deepseek-v4.1-flash-fast` for `triage_specifier`.
- **no free or stealth lanes.** the free tier is not used anywhere (see changelog).
- `display.show_cost: true` — set, but **it stays empty on CommandCode** (no
  provider pricing to read). see [cost visibility](#cost-visibility).

these stay on your main model on purpose: `background_review` (it writes your
memory and skills), `curator`, `kanban_decomposer`, `goal_judge` and `review`.
they shape what the agent keeps and plans, so they get the stronger model.

### the peak-hour trap

CommandCode bills the **DeepSeek family by time of day**, and the off-peak rate is
the one you see on the models page. peak is **01–04 and 06–10 UTC, Monday to
Friday only**, at 2×:

| lane | band | hours | input $/M | output $/M | cache read $/M |
|---|---|---|---|---|---|
| main + compression | off-peak | 17h/day | $0.16 | $0.58 | $0.016 |
| main + compression | peak | 01–04 & 06–10 UTC, Mon–Fri | $0.32 | $1.16 | $0.032 |
| vision | off-peak | 17h/day | $0.15 | $0.60 | $0.003 |
| vision | peak | 01–04 & 06–10 UTC, Mon–Fri | $0.30 | $1.20 | $0.006 |

the two DeepSeek lanes do **not** share a price. $0.15/$0.60 is the *vision* lane
(and was the main model before v0.1.2 replaced it with the fast tier) — reading that
row for the main model understates its input price and hides its 5× worse cache
read. read the row for the lane you are asking about.

for IST that is **06:30–09:30 and 11:30–15:30, weekdays** — both right in the
working day. weekends are always off-peak. `z-ai/glm-5.3-flash`, `gpt-6-luna`,
`Qwen/Qwen3.8-Flash` and `xiaomi/mimo-v2.6-flash` are flat-priced, so they are not
affected; only the DeepSeek lanes (main, vision, compression) are.

if you want a flat main model instead, `/model` over to `z-ai/glm-5.3-flash`
($0.15/$0.50 flat, 1M context, index 41.8) — the side tasks do not move. it
therefore costs *less* than the pinned main model on input ($0.15 vs $0.16) and
output ($0.50 vs $0.58), and being flat it never doubles in the peak windows. the
one rate the DeepSeek lane wins is cache read off-peak ($0.016 vs $0.03).

### cost visibility

`display.show_cost: true` is set in `config.yaml`, but on CommandCode there is
nothing for it to display. Hermes prices a model from the provider's own catalog,
and `provider/v1/models` publishes **no pricing fields at all** — each entry is only
`id`, `name`, `object`, `created`, `owned_by`, `context_length` and
`supported_endpoints`. the completions `usage` block carries token counts and no
`cost` either.

so every CommandCode session is recorded as unpriced. verified on a live session
(2026-10-05): `estimated_cost_usd` = `0.0`, `cost_status` = `unknown`, `cost_source`
= `none`. Hermes' pricing chain (OpenRouter adapter → bundled docs table → endpoint
metadata → models.dev) has no CommandCode source to fall back to, so this holds for
every pinned slot however cheap it is.

**to see real spend, use CommandCode's own account page:
<https://commandcode.ai/settings/usage>** (sign-in required). every figure in this
README is a list price read from the model pages — none of it is a measured bill.

leaving `show_cost: true` in place is harmless: it costs nothing and starts working
by itself if CommandCode ever serves pricing.

## keep your own profile instead

installing a distribution replaces that profile's config. to keep your current
setup and only move the side tasks, run these against your own profile:

```bash
hermes config set model.provider commandcode
hermes config set model.default deepseek/deepseek-v4.1-flash-fast
hermes config set auxiliary.vision.provider commandcode
hermes config set auxiliary.vision.model deepseek/deepseek-v4-flash-vision-exp
hermes config set auxiliary.compression.provider commandcode
hermes config set auxiliary.compression.model deepseek/deepseek-v4.1-flash-fast
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
- **price bands**, not just headline rates: each model page carries a band table
  under a "price bands" heading, and the headline figure is the *off-peak* one. the
  main and vision lanes have different bands, and the peak columns are where the 2×
  comes from. v0.1.3's peak table is read from those tables.
- **prices and Intelligence Index scores** read from the models table and the
  individual model pages on `https://commandcode.ai/models`. the one score quoted
  here as a comparison (39.5 for `deepseek-v4.1-flash`, 41.8 for `z-ai/glm-5.3-flash`,
  34.8 for the vision lane) is CommandCode's own published number. the fast tier
  this profile actually uses — `deepseek-v4.1-flash-fast` — is listed by CommandCode
  as **not yet scored**; no published accuracy figure is claimed for it. the GLM
  slots use the scored `z-ai/glm-5.3-flash` (41.8).
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

### v0.1.6 — 2026-10-06

- **v0.1.5 is reverted; the pins below are v0.1.4's.** v0.1.5 moved the main,
  compression, approval and triage lanes to `xiaomi/mimo-v2.6-flash` (cheaper on
  every rate, scored 37.9, flat-priced). it was reported the same day that the
  model does not reason — which leaves `reasoning_effort: low` on compression and
  approval buying nothing, and the main lane running with no chain of thought. a
  cheap lane that cannot reason is the wrong lane for the slots that exist to
  think. the four lanes are back: `deepseek/deepseek-v4.1-flash-fast` for main and
  compression, `z-ai/glm-5.3-flash` for approval and triage. this is a **pin
  change**: a plain `hermes profile update` will not deliver it, because `update`
  preserves `config.yaml`. use
  `hermes profile update commandcode-affordable --force-config`.
- **prices and ids re-verified on 2026-10-06**: all six pinned ids are still
  served by `provider/v1/models` (84 models), and every rate quoted in this README
  is unchanged since 2026-10-05.
- **corrected the `gpt-6-luna` claim**: it is not the cheapest *input* in the
  catalog — `stepfun/Step-3.5-Flash` undercuts it at $0.09/M against Luna's
  $0.10/M. it is the cheapest input *worth using* for these slots, since Step 3.5
  Flash scores 17.0 against Luna's 37.3.

### v0.1.4 — 2026-10-05

- **the GLM lanes moved back to `z-ai/glm-5.3-flash`**, undoing the v0.1.2 move to
  FlashX — `approval`, `triage_specifier`, and the fallback behind each `gpt-6-luna`
  slot and compression. this is a **pin change**: a plain `hermes profile update`
  will not deliver it, because `update` preserves `config.yaml`. use
  `hermes profile update commandcode-affordable --force-config`.
- why: Flash is cheaper than FlashX on *every* rate — $0.15/$0.50 vs $0.37/$1.25,
  cache read $0.03 vs $0.07 — and unlike FlashX it carries a published Intelligence
  Index (41.8, vs not scored). v0.1.2's trade bought latency at roughly 2.5× the
  price; that reversed here because a scored model being also the cheaper one leaves
  no argument for the other. the cost, stated plainly: **Flash is the slower tier**,
  and approval is the one slot where latency is felt.
- **no longer a cost increase over `auto`** on input and output: `z-ai/glm-5.3-flash`
  ($0.15/$0.50) undercuts both FlashX and the main model `deepseek-v4.1-flash-fast`
  ($0.16/$0.58). the exception is cache read off-peak, where the DeepSeek lane is
  cheaper ($0.016 vs $0.03) — so an approval check that is mostly a cache hit can
  still cost slightly more than `auto` did.

### v0.1.3 — 2026-10-05

- **accuracy pass. no pins changed** — `config.yaml`'s only edit is its version
  header, so a plain `hermes profile update` is enough; `--force-config` is not
  needed and would only reset that comment.
- **the peak-hour table was wrong.** it quoted $0.15/$0.60 off-peak and $0.30/$1.20
  peak, which are the *vision* lane's rates — and were the main model's rates before
  v0.1.2 moved it to the fast tier. the pinned main and compression model is
  `deepseek/deepseek-v4.1-flash-fast` at **$0.16/$0.58** off-peak and
  **$0.32/$1.16** peak. the table now lists both DeepSeek lanes separately and adds
  the cache-read column ($0.016/$0.032 for the main lane), because cache reads are
  what most of an agent loop's input actually is.
- **`display.show_cost: true` does not show spend on CommandCode.** the old bullet
  claimed it does. it cannot: `provider/v1/models` publishes no pricing and the
  completions `usage` block carries no `cost`, so Hermes records every session with
  `cost_status: unknown` / `cost_source: none` / `estimated_cost_usd: 0.0`. a new
  [cost visibility](#cost-visibility) section states the reason and points at
  <https://commandcode.ai/settings/usage> for real spend. the setting itself is kept
  — it is inert, and correct if the provider ever serves pricing.
- **stated the approval/triage cost against the main model**, not only against their
  GLM sibling. FlashX is ~2.3× the main model off-peak (~1.16× at peak, where
  DeepSeek doubles and FlashX is flat). the old text compared FlashX only to the
  cheaper `z-ai/glm-5.3-flash`, which hides that these two slots are the only ones
  costing more than the `auto` behaviour they replaced.
- **added an update section**, including that a plain `update` preserves
  `config.yaml` and so will not deliver upstream pin changes.

### v0.1.2 — 2026-10-05

- **main and compression lanes moved to the DeepSeek V4.1 *fast* tier**
  (`deepseek/deepseek-v4.1-flash` → `deepseek/deepseek-v4.1-flash-fast`), and
  **every GLM lane moved to GLM-5.3 FlashX** (`z-ai/glm-5.3-flash` →
  `z-ai/glm-5.3-flashx`) — approval, kanban triage, and the fallback behind each
  `gpt-6-luna` slot.
- this is a latency-for-price trade and it costs more: main goes from
  $0.15/$0.60 (cache read $0.003) to $0.16/$0.58 (cache read $0.016), and FlashX
  from $0.15/$0.50 to $0.37/$1.25 (cache read $0.075). both new models are listed
  by CommandCode as **not yet scored**, so the pins no longer rest on a published
  Intelligence Index the way v0.1.0–v0.1.1 did. the scored, cheaper siblings are
  named in `config.yaml` next to each slot if that trade is not wanted.

### v0.1.1 — 2026-10-05

- **removed every free and stealth lane.** `title_generation`, `skills_hub`, `mcp`,
  `profile_describer`, `monitor`, `tts_audio_tags` and `memory_query_rewrite` moved
  from `inclusionai/ling-3.1-flash:free` to `gpt-6-luna`; `triage_specifier` moved
  to `z-ai/glm-5.3-flashx`. each keeps a paid fallback.
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
