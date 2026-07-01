# Cross-Border Price Compare

跨国比价助手 —— 一个基于 Claude Code / Cherry Studio 的 AI Skill，帮你判断出国旅行时哪些东西值得从国内带。

---

## 启发

2026年我准备去加拿大，打包行李时面对一个经典问题：**这东西到了那边再买行不行？**

作为一个经常做饭的人，我的行李箱里有大量空间留给了"厨房装备"——火锅底料、老干妈、花椒八角……但到底哪些是真需要带的？哪些到了当地 Walmart 随手就能买到、价格也差不多？

手动一个个搜索对比太累了。于是我想：能不能让 AI 帮我干这件事？

这个 Skill 就是答案。它同时搜索中国电商（通过值得买海纳购物管家 API）和加拿大 Walmart 官网，算出价格比值，直接告诉你：**带，还是不带。**

跑了几轮对比后发现一些有趣的结论：

- 火锅底料加拿大价格是中国的 4 倍多
- 干花椒在加拿大 Walmart 根本买不到
- 桂皮差价接近 10 倍
- 厨房纸巾反而不必带——加拿大价格持平甚至更便宜
- 出刃包丁（日式鱼刀）加拿大完全断货

这些"反直觉"的发现，就是做这个项目的意义。

---

## 功能

- 同时搜索**中国市场**（京东、天猫、淘宝等全平台）和**外国 Walmart**（默认加拿大 walmart.ca）
- 自动获取实时汇率，统一换算为人民币比较
- 计算价格比值，设置阈值判定（默认 2.0 倍）
- 特殊场景处理：
  - 商品仅中国有售 → 比值 `∞`（必带）
  - 商品仅国外有售 → 比值 `0`（不必带）
- 支持批量对比多种商品
- 输出清晰的表格 + 打包建议

---
## 结果举例
> "请帮我对比干香菇、木耳、干辣椒、干花椒、八角、桂皮、紫菜的价格。"


| # | 商品 | 中国 ¥/100g | 加拿大 $/100g | 加拿大 ¥/100g | 比值 | 判定 |
|---|------|------------|-------------|-------------|------|------|
| 1 | 干香菇 | ¥11.9 | $6.39 | ¥32.30 | **2.71x** | 🔴 必带 |
| 2 | 木耳 | ¥19.6 | $6.39 | ¥32.30 | 1.65x | 🟡 可带 |
| 3 | 干辣椒 | ¥15.2 | $24.63 | ¥124.40 | **8.18x** | 🔴 必带 |
| 4 | 干花椒 | ¥15.2 | **沃尔玛无售** | — | **∞** | 🔴 必带 |
| 5 | 八角 | ¥11.2 | $12.56 | ¥63.40 | **5.66x** | 🔴 必带 |
| 6 | 桂皮 | ¥12.9 | $24.00 | ¥121.20 | **9.40x** | 🔴 必带 |
| 7 | 紫菜 | ¥23.0 | $10.61 | ¥53.60 | **2.33x** | 🔴 必带 |


结论： 干货香料是这次比价中差异最大的品类！7样中有6样超过2倍阈值。花椒加拿大完全断货，桂皮差价近10倍。这些都是轻量干货，行李有空隙就塞。
---


## 工作原理

```
用户输入商品名
    │
    ├──→ 海纳购物管家 API ──→ 京东/淘宝/天猫价格 (CNY)
    │
    ├──→ Exa Web Search ──→ Walmart.ca 商品页 ──→ 价格 (CAD)
    │
    ├──→ Exa Web Search ──→ XE/Wise 实时汇率
    │
    └──→ 计算比值 → 输出报告
```

两个搜索**并行执行**，一次查询通常在 10-20 秒内完成。

---

## 前置依赖

### 1. 运行环境

- **Claude Code** 或 **Cherry Studio**（支持 Skill 机制的 AI 客户端）
- **Python 3.9+**

### 2. Python 依赖

```bash
pip install requests
```

海纳购物管家的搜索脚本依赖 `requests` 库调用 API。

### 3. API Key

本 Skill 依赖两个外部服务：

| 服务 | 用途 | 获取方式 |
|------|------|----------|
| **值得买海纳购物管家 API** | 搜索中国电商商品价格 | 访问 [ai.zhidemai.com/skills](https://ai.zhidemai.com/skills) 或邮件联系 group-content@zhidemai.com |
| **Exa Search** | 搜索 Walmart.ca 商品 + 汇率 | 注册 [exa.ai](https://exa.ai) 获取 API Key（Claude Code 内置 MCP 支持） |

> Exa Search 在 Claude Code 中通过 MCP 工具 `mcp__exa__web_search_exa` 和 `mcp__exa__web_fetch_exa` 直接调用，无需额外配置。

---

## 安装

### 方式一：通过技能市场安装（推荐）

在 Claude Code / Cherry Studio 中：

```
/find-skills "price compare"
```

找到 `Cross-Border Price Compare` 并安装。

### 方式二：手动安装

1. 将本仓库克隆或复制到 Skills 目录：

```bash
# Claude Code
cp -r price-compare/ ~/.claude/skills/price-compare/

# Cherry Studio
cp -r price-compare/ "<CherryStudio Data>/Skills/price-compare/"
```

2. 安装依赖 Skill —— **海纳购物管家**：

```bash
# 同样放入 Skills 目录
cp -r Haina-Shopping-Assistant/ "<Skills目录>/Haina-Shopping-Assistant/"
cd "<Skills目录>/Haina-Shopping-Assistant/scripts"
python3 -m venv .venv
source .venv/bin/activate
pip install requests
```

3. 配置 API Key：

```bash
export ZHIDEMAI_CONTENT_SEARCH="你的内容搜索Key"
export ZHIDEMAI_PRODUCT_SEARCH="你的商品搜索Key"
```

---

## 使用方法

在 Claude Code / Cherry Studio 中直接对话即可：

```
搜索以下商品，比较中加价格：火锅底料、老干妈、干香菇、八角
```

或者：

```
帮我对比沃尔玛中国和加拿大的不锈钢炒锅价格
```

Skill 会自动触发，执行搜索并输出对比报告。

### 自定义参数

可以在对话中指定：

- **目标国家**：默认加拿大（walmart.ca），可改为美国（walmart.com）、墨西哥（walmart.com.mx）等
- **阈值**：默认 2.0（国外价格 ≥ 中国 2 倍视为"值得带"）

示例：

```
对比美国和中国的咖啡机价格，阈值设为 1.5
```

---

## 输出示例

```
## 价格对比报告

> 汇率：1 CAD = 5.05 CNY | 阈值：2.0

| 商品 | 中国价格 | 加拿大价格 | 折合CNY | 比值 | 建议 |
|------|---------|-----------|---------|------|------|
| 火锅底料 | ¥6.91 | $5.97 | ¥30.15 | 4.36x | 🔴 必带 |
| 老干妈 | ¥10.20 | $3.57 | ¥18.03 | 1.77x | 🟡 可带 |
| 干花椒 | ¥15.20 | 无售 | — | ∞ | 🔴 必带 |
| 厨房纸 | ¥10/卷 | $1.66/卷 | ¥8.38 | 0.84x | 🟢 不必 |

### 最终建议
🔥 必带：火锅底料(4.36x)、干花椒(断货)
🤔 可选：老干妈(1.77x)
🟢 不必：厨房纸(价格持平)
```

---

## 注意事项

### API 相关

1. **海纳购物管家 API Key 需要单独申请**。体验 Key 有调用频次限制，正式使用建议申请专属 Key。申请入口：[ai.zhidemai.com/skills](https://ai.zhidemai.com/skills) 或邮件 group-content@zhidemai.com。

2. **不要把 API Key 硬编码在代码中或提交到 Git**。本 Skill 通过环境变量 `ZHIDEMAI_CONTENT_SEARCH` 和 `ZHIDEMAI_PRODUCT_SEARCH` 读取 Key。建议写入 `~/.zshrc` 或 `~/.bashrc`。

3. 海纳商品搜索返回的是全平台最低价，可能与 Walmart 自营商品品质不完全对等。对比时注意规格（克数、尺寸等）。

### Walmart 相关

4. **Walmart 中国（walmart.cn）没有公开的商品搜索页面**。中国 Walmart 主要通过微信小程序和京东旗舰店运营。因此本 Skill 使用海纳 API 搜索**全电商平台**作为中国市场价格参考，而非仅限于 Walmart 中国。

5. Walmart.ca 的部分商品页面有反爬机制（机器人验证），可能导致抓取失败。遇到此情况 Skill 会使用搜索结果摘要中的价格信息作为替代。

### 价格与汇率

6. **汇率为实时查询结果**，不同时间查询可能有微小差异。

7. 中国电商平台价格波动频繁（促销、优惠券等），Skill 取当前搜索结果的最低价。

8. 加拿大 Walmart 显示价格为税前价格，实际购买时各省税率不同（5%-15%），本 Skill 暂未计入税费。

### 使用局限

9. 本 Skill 运行在 AI Agent 环境中，每次查询依赖 LLM 理解和执行技能指令。搜索结果质量受搜索引擎和 LLM 能力双重影响。

10. 刀具等特殊物品请注意航空托运规定，不能随身携带。

---

## 致谢

- [值得买海纳购物管家](https://ai.zhidemai.com/skills) —— 提供中国电商商品搜索能力
- [Exa](https://exa.ai) —— 提供海外网站搜索能力
- [Anthropic Claude Code](https://claude.ai/code) —— 提供 AI Agent 运行环境
