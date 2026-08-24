# 名贵商品复核 Prestige-Goods-Recheck

一个轻量级 **维多利亚 3** MOD，**定期复核你国家的公司**，并**添加本可以被激活、却因原版机制而无法激活的名贵商品日志条目**。

A lightweight Victoria 3 mod that **periodically re-checks your country's companies** and **re-adds any prestige-good Journal Entry** that should have been activated but was missed by the vanilla game.

> 兼容性：`1.13.*` 支持简中 / 英文
> 
> Compatibility: Victoria 3 `1.13.*`. Target scope: English & Simplified Chinese.
>
> 范围说明：与后续发布版不同，本快照版本**对所有国家（玩家和每个 AI 国家）**周期复核，
> 并利用错峰起始偏移把各国的检查分散到 12 个不同月份。AI 检查默认开启，可通过全局开关
> `pgs_ai_check_disabled` 关闭。
>
> Scope note: unlike the later release, this snapshot runs the periodic recheck for **every
> country (player and each AI)** and spreads the checks across 12 different months via staggered
> startup offsets. AI checking is on by default and can be toggled off with the global switch
> `pgs_ai_check_disabled`.


# README.md
- en [English](README.en.md)
- zh_CN [简体中文](README.md)


---

## 实现路径

### 决策 —— 打开面板
`common\decisions\00_pgs_decisions.txt`
- `pgs_show_monitor_decision` → `add_journal_entry = { type = je_pgs_monitor_panel }`（手动入口）。
- `pgs_toggle_ai_check_decision` → 切换全局开关 `pgs_ai_check_disabled`（设置＝关闭 AI 检查；
  移除＝开启）。两个决策均 `ai_chance = 0`。

### 日志条目 —— 循环宿主
`common\journal_entries\00_pgs_monitor.txt`
- `je_pgs_monitor_panel`（组 `je_group_pgs_monitor`，国家作用域）。
- `is_shown_when_inactive` = 有公司在生产名贵商品（仅控制显示）。
- `possible = always = yes` → 面板在**每一个**国家的每月脉冲中参与。
- `on_monthly_pulse` → 驱动逻辑（见下节）。
- 按钮：`pgs_check_now_button`、`pgs_disable_ai_check_button`、`pgs_enable_ai_check_button`。

### 脚本效果 —— 检查逻辑
`common\scripted_effects\00_pgs_effects.txt`
- `pgs_check_single_good`（商品/解锁变量/日志）：若持有解锁变量的公司存在 **且** 日志未激活 **且**
  有公司合格（可生产＋未生产＋繁荣＋无变量）→ 补激活日志。
- `pgs_run_all_checks`：重置 `pgs_readded_count`，对全部 16 种商品执行检查，最后调用钩子
  `pgs_extra_prestige_good_checks`。
- `pgs_extra_prestige_good_checks` —— 供其他 MOD 追加商品的空覆盖钩子。

### 脚本按钮
`common\scripted_buttons\00_pgs_buttons.txt`
- **立即检查** → `pgs_run_all_checks`。
- **关闭 AI 自动检查** → `set_global_variable pgs_ai_check_disabled = 1`。
- **启用 AI 自动检查** → `remove_global_variable pgs_ai_check_disabled`。

### 日志组
`common\journal_entry_groups\00_pgs_groups.txt` —— `je_group_pgs_monitor`（国家作用域）。

### 本地化
`localization\` —— 日志、按钮、决策的英/简中界面文案。

---

## 12 个月循环如何工作

`on_monthly_pulse` 采用“各国自持计数器＋全局错峰”的设计。

1. **早期短路（每月）：**
   `if = { limit = { is_ai = yes, has_global_variable = pgs_ai_check_disabled } }` → 跳过主体。
   关闭 AI 检查后，**AI 国家**仅付一次布尔判断＋一次标志位判断即止；玩家不受影响照常运行。
2. **懒初始化起始偏移（首次脉冲，任何国家，仅一次）：**
   若国家没有 `pgs_check_counter`，就占用手**全局**计数器 `pgs_next_turn` 分配的 `0–11` 序号，
   随后 `pgs_next_turn` 自增，到 11 后回绕为 0。
3. **每月自增：** `change_variable pgs_check_counter add 1`。
4. **满 12：** `pgs_check_counter = 0`，随后 `pgs_run_all_checks`。
5. **自洽回绕：** 计数器归 0 后继续循环，不依赖任何单一国家 —— 新国家首次脉冲自动领号，
   消亡国家无残留。

由于每个国家的起始偏移是 `0–11`，其**检查月（对 12 取模）＝ 12 − 偏移**，12 个偏移正好覆盖全年 12 个月。
因此任意一个月约有 **全部国家的 1/12** 达到 `>= 12` 并执行扫描，各国的检查被**均匀分摊到全年**，
而非集中在同一个月。

---

## 性能影响

以典型负载估算：**1 玩家 + 约 300 AI 国家，每国约 4 家公司，1–2 个名贵商品生产商，AI 检查开启。**

### 每国月度成本（约 301 个国家，含玩家与 AI）

| 步骤 | 成本 | 频率 |
|------|------|------|
| `is_ai = yes` 短路 | 1 次布尔判断 | 每月 · 每国 |
| `has_global_variable pgs_ai_check_disabled` | 1 次标志位查询 | 每月 · 每国 |
| 偏移懒初始化 | 1 次全局读 + 1 次全局写 | 仅首次脉冲 |
| `pgs_check_counter` +1 | 1 次整数操作 | 每月 |
| 完整扫描（约占 1/12） | 16 × 2 趟小型 `any_company` | 每 12 个月一次 |

**12 个月里有 11 个月**，一个国家只付几行标志位判断＋一次整数自增。昂贵的 `any_company` 扫描
**每个国家每 12 个月仅一次**。

### 300 AI 时的每月汇总（AI 检查开启）

- **稳定态：** 每国检查月（对 12 取模）＝ `12 − 偏移`；偏移取 `0–11` 时检查均匀分散，
  每月约有 **300/12 ≈ 25 个国家**执行完整扫描，而非 300 国挤在同一个月。
- **单次扫描成本** ≈ `16 商品 × 2 趟小型 any_company` 遍历约 `4 家公司` → 各自主循环内远小于一微秒。
- **每月汇总** = 约 301 × 平凡标志位/自增操作 ＋ 约 25 × 16 × 2 趟小型 `any_company`，均匀分布，
  **无单月尖峰**。

### 数量级汇总

| 项目 | 近似成本 |
|------|----------|
| 非扫描月 · 全部约 301 国 | 约 301 ×（2 次标志位判断＋1 自增）≈ 可忽略 |
| 每月执行扫描的国家数（AI 开启） | 约 25（按 0–11 偏移错峰，300/12） |
| 每国扫描月 | 16 商品 × 约 2 趟小型 `any_company` |
| 单月峰值 | 由 0–11 错峰偏移规避 |

**结论：** 该设计用“永远分摊到 12 个月”的开销换取对 AI 的均等覆盖。AI 开启时，每月约 25 国扫描、
而非 300 国同时扫描；AI 关闭时，AI 国家被短路为每月两次标志位判断、仅玩家扫描。可测开销保持在毫秒级。
