# Prestige Goods Recheck (Victoria 3)

A lightweight Victoria 3 mod that **periodically re-checks your country's companies** and **re-adds any prestige-good Journal Entry** that should have been activated but was missed by the vanilla game.

> Compatibility: Victoria 3 `1.13.*` (also intent-tested against later 1.14-era save formats). Target scope: English & Simplified Chinese.

# README.md
- en [English](README.en.md)
- zh_CN [简体中文](README.md)


---

## Table of Contents

- [Why it exists](#why-it-exists)
- [Features](#features)
- [Implementation path](#implementation-path)
- [How the 12-month cycle works](#how-the-12-month-cycle-works)
- [Performance impact](#performance-impact)
- [Extensibility](#extensibility)
- [Compatibility & known limitations](#compatibility--known-limitations)
- [File structure](#file-structure)

---

## Why it exists

In vanilla, each of the 16 prestige goods has its own Journal Entry (e.g. `je_prestige_goods_clothes`). The vanilla `possible` block only activates a JE when a company **currently** satisfies:

- `can_potentially_produce_prestige_goods` = the good, **and**
- `is_producing_prestige_goods` = no, **and**
- `company_is_prosperous` = yes, **and**
- does **not** already hold the unlock variable (`prestige_good_*_var`).

Because the check is evaluated at specific moments, several real-game situations leave a good "stuck":
- A company qualifies only **after** the JE's `possible` pass already ran.
- The qualifying company is created later and the good was previously unlocked by another company.
- A company loses its unlock/prosperity status and a *new* company becomes eligible, but no re-check happens.

The net effect: the prestige-good JE for an already-unlocked good can silently **fail to appear** even though a valid potential producer now exists. Vanilla only re-adds the JE in the specific case of a company being disbanded (`re_add_prestige_good_je_if_lost`). （Hotfix 1.13.10 20260812）

**Prestige Goods Recheck** adds a generic safety net: on a fixed 12-month cycle it scans for any **unlocked good whose JE is inactive but which has a qualifying potential producer**, and re-adds that JE. It also gives the player a **manual "Check Now"** button and a global **auto-check toggle**.

---

## Features

- **Fixed 12-month recheck cycle** — no per-month `any_company` scanning.
- **Re-adds** any prestige-good JE that is: already unlocked by a company (a company holds the `*_var`) **+** currently inactive (`NOT has_journal_entry`) **+** has at least one qualifying potential producer.
- **"Check Now"** button — triggers a full 16-good scan immediately.
- **Enable / Disable Auto-Check** — a global toggle that turns the periodic loop on/off; when off, the player's monthly pulse short-circuits instantly.
- Covers **all 16 prestige goods**: generic clothes, groceries, furniture, fertilizer, tools, fish, meat, coffee, steel, opium, small arms, artillery, merchant marine, grain, paper, explosives.
- Friendly **localization** in English and Simplified Chinese.

---

## Implementation path

### 1. Decision — show the panel
`common/decisions/00_pgs_decisions.txt`

A decision (`pgs_show_monitor_decision`, always shown, `ai_chance = 0`) calls `add_journal_entry = { type = je_pgs_monitor_panel }`. This is the manual entry point that creates the monitoring panel.

### 2. Journal entry — the loop host
`common/journal_entries/00_pgs_monitor.txt`

`je_pgs_monitor_panel` is a player-facing host JE (group `je_group_pgs_monitor`, country context). It holds:

- **`is_shown_when_inactive`** = `is_ai = no` + `year >= 1` — display only, no CPU cost.
- **`possible`** = `is_ai = no` + at least one company producing prestige goods — i.e. the monitor is live for the player when any prestige good is unlocked.
- **`on_monthly_pulse`** — the actual driver. See [How the 12-month cycle works](#how-the-12-month-cycle-works).
- **Buttons** — `pgs_check_now_button`, `pgs_disable_ai_check_button`, `pgs_enable_ai_check_button`.

### 3. Scripted effects — the checks
`common/scripted_effects/00_pgs_effects.txt`

- **`pgs_check_single_good`** (param: good, unlock var, JE) — runs in country scope. If: a company holds the unlock var **and** the JE is not active **and** a company qualifies (`can_potentially_produce` + `is_producing = no` + `company_is_prosperous` + `NOT has_variable`), then `add_journal_entry` and bump `pgs_readded_count`.
- **`pgs_run_all_checks`** — resets `pgs_readded_count`, runs `pgs_check_single_good` for all 16 goods, then calls the extension hook `pgs_extra_prestige_good_checks`.
- **`pgs_extra_prestige_good_checks`** — an **empty override hook** so other mods can append their own checks without touching this file.

### 4. Scripted buttons
`common/scripted_buttons/00_pgs_buttons.txt`

- **Check Now** → `pgs_run_all_checks` + tooltip.
- **Disable Auto-Check** → `remove_global_variable = pgs_yearly_check_enabled`.
- **Enable Auto-Check** → `set_global_variable = { name = pgs_yearly_check_enabled value = 1 }`.

The global variable (`pgs_yearly_check_enabled`) is the master switch that the monthly pulse checks, and it works **across all countries** (global scope), independent of any single country.

### 5. Journal entry group
`common/journal_entry_groups/00_pgs_groups.txt`

Defines `je_group_pgs_monitor` with `context = country`.

### 6. Localization
`localization/english/pgs_l_english.yml` · `localization/simp_chinese/pgs_l_simp_chinese.yml`

All UI strings for the journal, buttons and decision.

---

## How the 12-month cycle works

The `on_monthly_pulse` effect uses a **self-sustaining per-country counter** plus short-circuiting:

1. **Early short-circuit (every month):**
   `if = { limit = { is_ai = no, has_global_variable = pgs_yearly_check_enabled } }`
   — Every country pays only **one boolean check (`is_ai = no`) + one global-variable check** per month. AI countries fail the `is_ai = no` check instantly and the body is skipped — **no company scan ever runs for AI**.
2. **Counter init (player only, once):** if `pgs_check_counter` is missing, it takes the next ticket from the global counter `pgs_next_turn` (0–11) as its startup offset. `pgs_next_turn` is a global, so offsets are handed out once and then wrap from 11 back to 0.
3. **Monthly increment:** `change_variable pgs_check_counter add 1`.
4. **On reaching 12:** `pgs_check_counter = 0`, then `pgs_run_all_checks`.
5. **Reset:** the counter wraps to 0 and the loop continues forever with no external dependence on any single country (new countries wean onto a ticket automatically; dead countries leave no residue).

> **Design note:** the original plan staggered the offset across **all** AI countries (0–11) so their checks would spread over 12 different months and avoid a single-month spike. In the final build the AI country checks were **dropped as not meaningful**, but the counter/ticket data structures were kept for future extension. As shipped, the periodic loop runs **for the player only**.

---

## Performance impact

Measured under a representative load: **1 player + ~300 AI countries, ~4 companies per country, 1–2 prestige-good producers.**

### Per-country, per-month (all ~301 countries)

| Step | Cost | Frequency |
|------|------|-----------|
| `is_ai = no` | 1 boolean | every month, every country |
| `has_global_variable` | 1 flag lookup | every month, every country |
| `pgs_check_counter` +1 | 1 integer op | monthly, player only |
| **full 16-good scan** | 16 × small `any_company` | **player only**, every 12th month |

The **expense for every country — including all AI — is two trivial flag checks per month.** The potentially expensive `any_company` walk is:

- **Amortized**: run only **1 of every 12 months** (counter reaches 12).
- **Player-only**: AI countries never run it, so the worst case (300 countries scanning every month) is avoided entirely.

### What a full scan actually costs (player, every 12th month)

`pgs_run_all_checks` iterates **16 goods**; each `pgs_check_single_good` runs two small `any_company` passes:

1. `any_company has_variable` — stops at the first holder; with `~4 companies` this is a couple of predicate evaluations.
2. `any_company { can_potentially_produce + is_producing = no + company_is_prosperous + NOT has_variable }` — one predicate pass over a handful of companies.

Per goods: well under a microsecond of on-the-loop cost; per cycle (16 goods): still negligible. Crucially, **for 11 of every 12 months the player's monthly pulse is just `counter += 1`** — there is no per-month `any_company`, no per-month journal re-evaluation, and no AI scanning.

### Hard numbers (order of magnitude)

| Quantity | Approximate cost |
|----------|------------------|
| Monthly cost, all ~301 countries | ~300 × 2 trivial flag checks ≈ negligible (microseconds) |
| Monthly cost, player non-scan months | 1 integer increment |
| Monthly cost, player scan month | 16 goods × ~2 small `any_company` passes |
| AI countries ever scanning | **0** |

**Bottom line:**  The measurable performance overhead is at the microsecond level and effectively invisible in normal play.

---

## Extensibility

Other mods can add their own prestige-good rechecks without editing this mod by **overriding the empty effect**:

```
pgs_extra_prestige_good_checks = {
    # your additional checks here
}
```

The effect is called at the end of every full check (both manual and cyclic).

---

## Compatibility & known limitations

- **Vanilla JE untouched** — only watches and re-adds; safe with other content mods that don't already override the same monitor.
- **Save compatibility**: adding/removing mid-game requires a **restart + save reload** (no runtime hotload).
- **Mid-game first load** may cause a one-shot batch re-trigger if several goods were already unlockable (see above).
- Supported in **English and Simplified Chinese** locales.

---

## File structure

```
prestige_goods_recheck/
├── common/
│   ├── decisions/
│   │   └── 00_pgs_decisions.txt        # Decision to open the monitor panel
│   ├── journal_entries/
│   │   └── 00_pgs_monitor.txt          # je_pgs_monitor_panel + 12-month cycle driver
│   ├── journal_entry_groups/
│   │   └── 00_pgs_groups.txt           # je_group_pgs_monitor (country context)
│   ├── scripted_buttons/
│   │   └── 00_pgs_buttons.txt          # Check Now / Enable / Disable Auto-Check
│   └── scripted_effects/
│       └── 00_pgs_effects.txt          # pgs_check_single_good / pgs_run_all_checks + hook
├── localization/
│   ├── english/
│   │   └── pgs_l_english.yml
│   └── simp_chinese/
│       └── pgs_l_simp_chinese.yml
└── README.md
```
