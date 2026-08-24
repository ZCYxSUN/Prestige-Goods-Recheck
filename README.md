# 名贵商品复核 Prestige-Goods-Recheck

一个轻量级 **维多利亚 3** MOD，**定期复核你国家的公司**，并**添加本可以被激活、却因原版机制而无法激活的名贵商品日志条目**。

A lightweight Victoria 3 mod that **periodically re-checks your country's companies** and **re-adds any prestige-good Journal Entry** that should have been activated but was missed by the vanilla game.

> 兼容性：`1.13.*` 支持简中 / 英文
> 
> Compatibility: Victoria 3 `1.13.*`. Target scope: English & Simplified Chinese.

# README.md
- en [English](README.en.md)
- zh_CN [简体中文](README.md)


---

## 目录

- [为什么需要它](#为什么需要它)
- [功能特性](#功能特性)
- [实现路径](#实现路径)
- [12 个月循环如何工作](#12-个月循环如何工作)
- [性能影响](#性能影响)
- [可扩展性](#可扩展性)
- [兼容性与已知限制](#兼容性与已知限制)
- [文件结构](#文件结构)

---

## 为什么需要它

原版每种名贵商品各有一个自己的日志条目（如 `je_prestige_goods_clothes`）。原版的 `possible` 只有在一家公司**当下**满足以下全部条件时才会激活该日志：

- 可以生产该名贵商品（`can_potentially_produce_prestige_goods`），且
- 当前未在生产该名贵商品（`is_producing_prestige_goods = no`），且
- 公司繁荣（`company_is_prosperous = yes`），且
- 尚未持有该解锁变量（`prestige_good_*_var`）。

由于该判断只在特定时机被评估，常有通用名贵商品被“卡住”的情况：
-  新公司更晚才出现并达到要求，而该商品之前已被其他公司解锁；


结果就是：即便现在存在合格的潜在生产商，已解锁名贵商品的日志也可能**不再出现**。原版只在“公司解散”这一种情况下补激活日志（`re_add_prestige_good_je_if_lost`）。（Hotfix 1.13.10 20260812）

**本 MOD** 提供一个通用兜底：以固定 **12 个月为周期**扫描——**凡是“已解锁”+“日志未激活”+“存在合格潜在生产商”**的名贵商品日志，就重新添加。并提供一键 **“立即检查”** 按钮和全局 **自动检查开关**。

---

## 功能特性

- **固定 12 个月复核周期** —— 不做每月 `any_company` 扫描。
- **补激活** 满足以下条件的名贵商品日志：某公司已解锁（持有 `*_var`）**＋** 日志当前未激活（`NOT has_journal_entry`）**＋** 至少存在一家合格潜在生产商。
- **“立即检查”** 按钮 —— 立刻对全部 16 种名贵商品做一次完整扫描。
- **启用 / 关闭自动检查** —— 全局开关；关闭后玩家不再每年检查。
- 覆盖 **全部 16 种名贵商品**：服饰、杂货、家具、化肥、工具、鱼、肉、咖啡、钢、鸦片、轻武器、火炮、商船、谷物、纸张、炸药。
- 完整 **简中 / 英文** 本地化。

---

## 实现路径

### 1. 决策 —— 打开面板
`common\decisions\00_pgs_decisions.txt`

决策 `pgs_show_monitor_decision`（恒显示，`ai_chance = 0`）调用 `add_journal_entry = { type = je_pgs_monitor_panel }`，是手动创建监控面板的入口。

### 2. 日志条目 —— 循环主体
`common\journal_entries\00_pgs_monitor.txt`

`je_pgs_monitor_panel` 是面向玩家的日志（组 `je_group_pgs_monitor`，国家作用域）。包含：

- **`is_shown_when_inactive`** = `is_ai = no` + `year >= 1` —— 仅控制显示，无 CPU 成本。
- **`possible`** = `is_ai = no` + 至少一家公司正在生产名贵商品 —— 即玩家国家在解锁了任意名贵商品时面板生效。
- **`on_monthly_pulse`** —— 真正的驱动逻辑。见[12 个月循环如何工作](#12-个月循环如何工作)。
- **按钮** —— `pgs_check_now_button`、`pgs_disable_ai_check_button`、`pgs_enable_ai_check_button`。

### 3. 脚本效果 —— 检查逻辑
`common\scripted_effects\00_pgs_effects.txt`

- **`pgs_check_single_good`**（参数：商品、解锁变量、日志）—— 国家作用域。若：有公司持有解锁变量 **且** 日志未激活 **且** 有公司合格（可生产＋未在生产＋繁荣＋`NOT has_variable`），则 `add_journal_entry` 并将 `pgs_readded_count` +1。
- **`pgs_run_all_checks`** —— 重置 `pgs_readded_count`，对全部 16 种商品执行 `pgs_check_single_good`，最后调用扩展钩子 `pgs_extra_prestige_good_checks`。
- **`pgs_extra_prestige_good_checks`** —— **空覆盖钩子**，供其他 MOD 在不改动本文件的前提下追加自己的检查。

### 4. 脚本按钮
`common\scripted_buttons\00_pgs_buttons.txt`

- **立即检查** → `pgs_run_all_checks` + 提示。
- **关闭自动检查** → `remove_global_variable = pgs_yearly_check_enabled`。
- **启用自动检查** → `set_global_variable = { name = pgs_yearly_check_enabled value = 1 }`。

全局变量 `pgs_yearly_check_enabled` 是每月校验的总开关，作用域为**全局**，可用于跨国家判断，不依赖任何单一国家。

### 5. 日志组
`common\journal_entry_groups\00_pgs_groups.txt`

定义 `je_group_pgs_monitor`，`context = country`。

### 6. 本地化
`localization\english\pgs_l_english.yml` · `localization\simp_chinese\pgs_l_simp_chinese.yml`

日志、按钮、决策的全部界面文案。

---

## 12 个月循环如何工作

`on_monthly_pulse` 采用“各国自持计数器＋短路跳出”的设计：

1. **早期判定（每月）：**
   `if = { limit = { is_ai = no, has_global_variable = pgs_yearly_check_enabled } }`
   —— 每个国家每月只付 **一次布尔判断（`is_ai = no`）＋ 一次全局变量判断**。AI 国家在 `is_ai = no` 处立刻判否并跳过主体 —— **AI 国家永远不会做公司扫描**。
2. **计数器初始化（仅玩家、仅一次）：** 若没有 `pgs_check_counter`，就占用手全局计数器 `pgs_next_turn` 分配的 0–11 中的序号作为起始偏移。`pgs_next_turn` 是全局变量，偏移只分配一次，到 11 后回绕为 0。
3. **每月自增：** `change_variable pgs_check_counter add 1`。
4. **满 12：** `pgs_check_counter = 0`，随后 `pgs_run_all_checks`。
5. **回绕：** 计数器归 0 后继续循环，自洽运行、不依赖任何单一国家（新国家自动领号，消亡国家无残留）。

> **设计说明：** 最初计划给**所有 AI 国家**也分配 0–11 的错峰偏移，让它们的检查分散到 12 个不同月份，避免同月尖峰。最终版**放弃**了 AI 国家检查（判定意义不大），但保留了计数器/领号数据结构以备扩展。**当前发布版的周期循环仅对玩家生效。**

---

## 性能影响

以典型负载估算：**1 玩家 + 约 300 AI 国家，每国约 4 家公司，1–2 个名贵商品生产商。**

### 每国·每月（全部约 301 个国家）

| 步骤 | 成本 | 频率 |
|------|------|------|
| `is_ai = no` | 1 次布尔判断 | 每月 · 每国 |
| `has_global_variable` | 1 次标志位查询 | 每月 · 每国 |
| `pgs_check_counter` +1 | 1 次整数操作 | 仅玩家 · 每月 |
| **完整 16 商品扫描** | 16 × 小型 `any_company` | **仅玩家** · 每 12 个月一次 |

**所有国家（含全部 AI）的月度开销只有两行近乎零成本的标志位判断。** 真正昂贵的 `any_company` 遍历被大幅优化：

- **摊薄到 1/12**：只在计数器满 12（每 12 个月）执行一次。
- **仅玩家**：AI 国家永不执行，从根本上避免了“300 国每月扫描”的最坏情况。

### 一次完整扫描的实际成本（仅玩家，第 12 个月）

`pgs_run_all_checks` 遍历 **16 种商品**；每种 `pgs_check_single_good` 做两趟小型 `any_company`：

1. `any_company has_variable` —— 命中第一个持有者即停；约 **4 家公司**下仅数次谓词求值。
2. `any_company { 可生产 + 未生产 + 繁荣 + NOT has_variable }` —— 对几家公司做一趟谓词遍历。

单商品成本：主循环内远小于一微秒；单周期（16 种）：仍可忽略不计。关键在于 **12 个月里有 11 个月的玩家月度脉冲只是 `counter += 1`** —— 既无每月 `any_company`，也无每月日志重评估，AI 更无扫描。

### 数量级汇总

| 项目 | 近似成本 |
|------|----------|
| 全部约 301 国的月度开销 | 约 300 × 2 次平凡标志位判断 ≈ 可忽略（微秒级） |
| 玩家非扫描月份的月度开销 | 1 次整数自增 |
| 玩家扫描月份的月度开销 | 16 商品 × 约 2 趟小型 `any_company` |
| AI 国家是否曾扫描 | **从不** |

**结论：** 可测量的性能开销处于微秒级，正常对局中完全无感。

---

## 可扩展性

其他 MOD 可通过**覆盖这个空效果**追加自己的名贵商品复核，无需改动本 MOD：

```
pgs_extra_prestige_good_checks = {
    # 这里写你额外的检查
}
```

每次完整检查（手动或周期）结束时都会调用该效果。

---

## 兼容性与已知限制

- **不改动原版日志** —— 只监视与补激活；与不覆盖同一监控器的其他内容 MOD 安全共存。
- **存档兼容**：局中添加/移除需**重启＋重载存档**（不支持运行时热加载）。
- **局中首次加载**在开启AI国家检查的分支中开启AI检查可能一次性集中补触发多个（见上文说明）。
- 支持 **简中 / 英文** 两种语言环境。

---

## 文件结构

```
prestige_goods_recheck/
├── common/
│   ├── decisions/
│   │   └── 00_pgs_decisions.txt        # 打开监控面板的决策
│   ├── journal_entries/
│   │   └── 00_pgs_monitor.txt          # je_pgs_monitor_panel + 12 个月周期驱动
│   ├── journal_entry_groups/
│   │   └── 00_pgs_groups.txt           # je_group_pgs_monitor（国家作用域）
│   ├── scripted_buttons/
│   │   └── 00_pgs_buttons.txt          # 立即检查 / 启用 / 关闭自动检查
│   └── scripted_effects/
│       └── 00_pgs_effects.txt          # pgs_check_single_good / pgs_run_all_checks + 钩子
├── localization/
│   ├── english/
│   │   └── pgs_l_english.yml
│   └── simp_chinese/
│       └── pgs_l_simp_chinese.yml
└── README_zh-CN.md
```
