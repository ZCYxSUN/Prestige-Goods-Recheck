# Prestige Goods Recheck (Victoria 3)

A lightweight Victoria 3 mod that **periodically re-checks your country's companies** and **re-adds any prestige-good Journal Entry** that should have been activated but was missed by the vanilla game.

> Compatibility: Victoria 3 `1.13.*` (also intent-tested against later 1.14-era save formats). Target scope: English & Simplified Chinese.
>
> Scope note: unlike the later release, this snapshot runs the periodic recheck for **every
> country (player and each AI)** and spreads the checks across 12 different months via staggered
> startup offsets. AI checking is on by default and can be toggled off with the global switch
> `pgs_ai_check_disabled`.

# README.md
- en [English](README.en.md)
- zh_CN [简体中文](README.md)


---

## Implementation path

### Decision — open the panel
`common/decisions/00_pgs_decisions.txt`
- `pgs_show_monitor_decision` → `add_journal_entry = { type = je_pgs_monitor_panel }` (manual entry point).
- `pgs_toggle_ai_check_decision` → toggles the global switch `pgs_ai_check_disabled`
  (set = disable AI checks; remove = enable). Both decisions are `ai_chance = 0`.

### Journal entry — the loop host
`common/journal_entries/00_pgs_monitor.txt`
- `je_pgs_monitor_panel` (group `je_group_pgs_monitor`, country context).
- `is_shown_when_inactive` = any company producing prestige goods (display only).
- `possible = always = yes` → the panel participates in the monthly pulse for **every** country.
- `on_monthly_pulse` → the driver (see section below).
- Buttons: `pgs_check_now_button`, `pgs_disable_ai_check_button`, `pgs_enable_ai_check_button`.

### Scripted effects — the checks
`common/scripted_effects/00_pgs_effects.txt`
- `pgs_check_single_good` (good / unlock var / JE): if a company holds the unlock var **and** the JE is
  inactive **and** a company qualifies (can produce + not producing + prosperous + no var) → re-add JE.
- `pgs_run_all_checks`: resets `pgs_readded_count`, runs all 16 goods, then calls the hook
  `pgs_extra_prestige_good_checks`.
- `pgs_extra_prestige_good_checks` — empty override hook for other mods to append goods.

### Scripted buttons
`common/scripted_buttons/00_pgs_buttons.txt`
- **Check Now** → `pgs_run_all_checks`.
- **Disable AI Auto-Check** → `set_global_variable pgs_ai_check_disabled = 1`.
- **Enable AI Auto-Check** → `remove_global_variable pgs_ai_check_disabled`.

### Journal entry group
`common/journal_entry_groups/00_pgs_groups.txt` — `je_group_pgs_monitor` (country context).

### Localization
`localization/` — English + Simplified Chinese strings for the journal, buttons and decisions.

---

## How the 12-month cycle works

The `on_monthly_pulse` effect uses a **per-country self-sustaining counter** plus a global stagger.

1. **Early short-circuit (every month):**
   `if = { limit = { is_ai = yes, has_global_variable = pgs_ai_check_disabled } }` → skip the body.
   When AI checking is disabled, **AI countries** pay one boolean + one flag check and stop there;
   the player still runs normally.
2. **Lazy startup offset (first pulse, any country, once):**
   if the country has no `pgs_check_counter`, it claims the next ticket `0–11` from the **global**
   counter `pgs_next_turn`, then `pgs_next_turn` increments and wraps back to 0 after 11.
3. **Monthly increment:** `change_variable pgs_check_counter add 1`.
4. **On reaching 12:** `pgs_check_counter = 0`, then `pgs_run_all_checks`.
5. **Self-sustaining reset:** the counter wraps and the loop never depends on any single country —
   new countries auto-claim a ticket on their first pulse; dead countries leave no residue.

Because each country's startup offset is `0–11`, a country's **check month (mod 12) = 12 − offset**,
so the 12 offsets map to all 12 months. At any given month roughly **1/12 of all countries** hit
`>= 12` and run their scan, so per-country checks are **spread evenly across the year** rather than
firing all at once.

---

## Performance impact

Measured under a representative load: **1 player + ~300 AI countries, ~4 companies per country,
1–2 prestige goods producers, AI checking enabled.**

### Per-country monthly cost ($\approx$ 301 countries, player + AI)

| Step | Cost | Frequency |
|------|------|-----------|
| `is_ai = yes` short-circuit | 1 boolean | every month, every country |
| `has_global_variable pgs_ai_check_disabled` | 1 flag lookup | every month, every country |
| offset init (lazy) | 1 global read + 1 global write | first pulse only |
| `pgs_check_counter` +1 | 1 integer op | monthly |
| full 16-good scan (~1/12) | 16 × 2 small `any_company` | every 12th month |

For **11 of every 12 months** a country only pays a handful of flag checks + one integer increment.
The expensive `any_company` scan runs **once per 12 months per country**.

### Monthly aggregate with 300 AI (AI checking on)

- **Steady state:** for every country the check month (mod 12) = `12 − offset`; with offsets `0–11` the
  checks are uniformly spread, so about **300/12 ≈ 25 countries run their full scan each month**, not
  300 in a single month.
- **Per scan cost** ≈ `16 goods × 2 small any_company passes` over `~4 companies` → well under a
  microsecond on-loop each.
- **Monthly aggregate** = ~301 × trivial flag/increment ops + ~25 × 16 × 2 small `any_company` passes,
  distributed evenly — no single-month spike.

### Order-of-magnitude summary

| Quantity | Approximate cost |
|----------|------------------|
| Non-scan months, all ~301 countries | ~301 × (2 flag checks + 1 increment) ≈ negligible |
| Countries scanning each month (AI on) | ~25 (offset-staggered, 300/12) |
| Per-country scan month | 16 goods × ~2 small `any_company` passes |
| Single-month peak | avoided by 0–11 offset stagger |

**Bottom line:** the design trades even AI coverage for workload that is **always spread across 12
months**. With AI checking on, roughly 25 countries scan per month instead of all 300 at once; with AI
checking disabled, AI countries are short-circuited to two flag checks per month and only the player
scans. Measurable cost stays at the millisecond level.
