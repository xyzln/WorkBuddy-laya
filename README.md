# WorkBuddy-laya

把 **Laya 的 System-1 决策协议**复刻成 **WorkBuddy 原生集成包**：一个能自己跑起来的安全决策网关 +
专家团。零外部依赖、零联网、不装任何模型权重——加载本项目的 WorkBuddy 大模型本身就是决策引擎。

> 与原版 [`laya-decision-skill`](https://github.com/xyzln/laya-decision-skill) 的区别：
> 这里把它做成 **WorkBuddy 专属集成**——核心是 `PreToolUse` 勾子，在工具执行前真实拦截危险操作，
> 而不只是一个通用 skill。

---

## 它能做什么

- **工具执行前拦截**：`Bash` / `Write` / `Edit` / `WebFetch` 跑之前，勾子先做一次离线风险判定。
  - 灾难性指令（`rm -rf /`、`git push --force`、`curl … | sh`、fork-bomb…）→ **直接拦截 (deny)**
  - 提权 / 写敏感文件（`.env`、`id_rsa`、`/etc`、`.git`、`.codebuddy`…）→ **转人工确认 (ask)**
  - 正常命令（`ls`、`grep`、`Write src/app.py`）→ 放行
- **专家团决策协议**：`决策中枢 + 5 领域专家`（安全合规 / 代码评审 / 运维 SRE / 需求分诊 / 质量闸门），
  针对固定题型 `choice/score/noul` 作答并给出校准式置信度，低置信自动弃权升级。
- **审计留痕**：每次闸门判定写入 `hook_audit.jsonl`，可回溯升级率/弃权率。

## 快速开始

```bash
# 1. 克隆
git clone https://github.com/xyzln/WorkBuddy-laya.git
cd WorkBuddy-laya

# 2. 部署：软链 skill + 合并 PreToolUse 钩子到用户级 settings
python3 scripts/install.py

# 3. 重启 WorkBuddy —— 之后每次危险操作都会被网关拦一道

# 4. 自测
python3 tests/test_hook.py            # 10 个闸门用例
python3 scripts/laya_engine.py selftest
```

项目级部署（写入仓库 `.codebuddy/settings.json`，便于团队共享）：
```bash
python3 scripts/install.py --scope project
```

## 目录结构

```
WorkBuddy-laya/
├── SKILL.md                  # WorkBuddy skill（决策中枢 + 网关说明）
├── hooks/
│   └── hook_gate.py          # PreToolUse 安全闸门（核心差异点）
├── scripts/
│   ├── laya_engine.py        # 零依赖决策引擎：prompt/validate/decide/route/consensus/audit/selftest
│   ├── calibrate.py          # min_confidence 阈值扫描
│   ├── batch_eval.py         # 批量评估报表
│   ├── schema_gen.py         # 从自然语言生成 schema 草稿
│   └── install.py            # 一键部署（skill 软链 + 钩子合并）
├── experts/                  # 5 领域专家（SKILL.md + schema.json）+ _registry.json
├── assets/schemas/           # 6 套现成题型（code-review/issue-triage/command-risk/.../write-risk）
├── config/settings.hooks.json  # 钩子配置模板（{{HOOK_PATH}} 占位）
├── references/               # 原理 / 引擎 API / 校准 / 专家团 / 题型库 / 钩子接入
└── tests/                    # test_hook.py + samples.jsonl
```

## 设计取舍（诚实说明）

- 置信度是**大模型自我报告**（非 RLCD 真·校准）。生产建议 `min_confidence` 设 0.6~0.7，
  高 stakes 操作一律走高闸门，**不自动执行**。
- 勾子里跑的是**离线条目**（关键词/CJK 二元组 + 黑名单），是兜底 baseline；真实决策能力来自加载本项目的 LLM。
- 详细原理与协议见 [`references/`](references/)。

## License

MIT —— 自由用于个人与团队的安全决策网关集成。
