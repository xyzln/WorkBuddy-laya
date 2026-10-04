---
name: laya-decision
version: 1.0.0
author: WorkBuddy-laya
description: >-
  自包含 System-1 决策协议（复刻 Laya 原理，零外部依赖），并作为 WorkBuddy 原生安全网关运行：
  在 Bash/Write/Edit/WebFetch 等工具执行前，由 PreToolUse 钩子调用决策引擎离线路径做风险判定，
  命中灾难性指令直接拦截(deny)、高危需确认转人工(ask)。同时提供「决策中枢 + 5 领域专家」专家团协议，
  让大模型针对固定题型(choice/score/noul)作答并给出校准式置信度，低于 min_confidence 自动弃权升级。
  适用于命令风险、代码评审、运维 SRE、需求分诊、质量闸门、RAG 相关性等需要「先决策后行动」的场景。
metadata:
  type: skill
  triggers:
    - 决策
    - 风险评估
    - 命令风险
    - 该不该执行
    - 高危操作
    - 兜底拦截
    - 专家评审
  allowed-tools:
    - Bash
    - Read
    - Write
---

# WorkBuddy-laya · System-1 决策协议（WorkBuddy 原生安全网关）

本 skill 做两件事，且**都不依赖任何外部模型 / 联网 / torch**：

1. **决策中枢 + 专家团协议**：把 Laya 的「固定答案空间 + 校准置信度 + 预测-行动分离」复刻成协议，
   让加载本 skill 的大模型自己当 System-1 引擎，针对固定题型作答。
2. **PreToolUse 安全网关**：一个 `hooks/hook_gate.py` 勾子，在 WorkBuddy 每次执行工具前，
   用引擎的离线路径（关键词/CJK 二元组启发式 + 灾难性黑名单）做风险判定，真正拦住危险操作。

> 为什么是「自包含复刻」而非原版 `laya` 包：原版需联网下载 300M+ 权重、装 torch，沙箱/离线环境跑不起来。
> 本版让大模型自身充当引擎（生产环境），并用纯标准库脚本做协议强制与离线兜底，零 pip 依赖即可运行。

## 何时触发本 skill

- 执行前需要**先评估风险再行动**：命令是否破坏性、是否提权、是否写敏感文件、是否出网。
- 让专家团对**代码评审 / 运维变更 / 需求分诊 / 发布质量**给出结构化、可审计的决策。
- 需要给 WorkBuddy 加一层**不依赖权限规则的硬闸门**（钩子会在权限弹窗之前生效）。

## 四阶段工作流（专家团）

1. **路由 route** — `python3 scripts/laya_engine.py route --state '<文本>'`
   选出最相关的领域专家（读 `experts/_registry.json`）。破坏性命令 → `ops-sre`，代码 diff → `code-reviewer`。
2. **出题 prompt** — `python3 scripts/laya_engine.py prompt --schema-file assets/schemas/<x>.json --state '<文本>'`
   生成 System-1 决策提示词，让大模型对固定题型作答（每条带 0~1 置信度）。
3. **作答** — 大模型按提示词输出 JSON：`{"<qid>": {"value": ..., "confidence": 0.0~1.0}}`。
4. **校准 / 闸门 / 共识 / 审计**：
   - `validate` 归一化 + `min_confidence` 闸门（低置信字段弃权）→ 路由到 System 2 或人工；
   - `consensus` 融合多专家结论（任一 human → 团队 human，否则 system2，否则 pass）；
   - `audit` 把每次决策留痕为 JSONL，可回溯升级率/弃权率。

## PreToolUse 安全网关（关键差异点）

勾子 `hooks/hook_gate.py` 由 WorkBuddy 在工具执行前调用（stdin 注入工具信息），输出：

```json
{ "hookSpecificOutput": { "hookEventName": "PreToolUse",
                          "permissionDecision": "allow|ask|deny",
                          "permissionDecisionReason": "..." } }
```

判定逻辑（全部离线）：
- **deny（硬拦截）**：命中灾难性黑名单 —— `rm -rf /`、`mkfs`、`dd if=`、`git push --force`、
  `git reset --hard`、`curl ... | sh`、fork-bomb 等。
- **ask（转人工确认）**：提权指令（`sudo`/`su`/`doas`）、写入敏感路径（`.env`/`id_rsa`/`*.pem`/`/etc`/`.git`/`.codebuddy`）、
  或引擎判出任一高危闸门（destructive/privileged/requires_confirmation/...）。
- **allow**：未命中任何风险模式。
- **fail-safe**：引擎异常时默认转 `ask`，绝不静默放行。

> 引擎判定只升到 `ask`，真正的 `deny` 仅来自灾难性黑名单 —— 这样 `ls`/`grep` 等正常命令不会被误伤。

## 安装与部署

```bash
# 1) 软链 skill + 合并 PreToolUse 钩子到用户级 settings（重启 WorkBuddy 后生效）
python3 scripts/install.py

# 2) 或部署到当前项目级（写入 .codebuddy/settings.json）
python3 scripts/install.py --scope project

# 3) 卸载
python3 scripts/install.py --uninstall
```

也可手动把 `config/settings.hooks.json` 的 `{{HOOK_PATH}}` 替换为
`hooks/hook_gate.py` 的绝对路径，并入 `~/.codebuddy/settings.json` 的 `hooks.PreToolUse`。

## 诚实的边界

- 置信度是**大模型自我报告**（非 RLCD 真·校准）。生产建议 `min_confidence` 设 0.6~0.7，
  高 stakes（删数据/提权/出网/退款/发布）一律走高闸门，**不自动执行**。
- 勾子里的离线路径是**兜底 baseline**，真实决策能力来自加载本 skill 的大模型本身。
- 详见 `references/`：`principle.md`（原理）、`engine.md`（API）、`calibration.md`（闸门策略）、
  `expert-team.md`（专家团）、`question-library.md`（题型库）、`hooks.md`（网关接入）。
