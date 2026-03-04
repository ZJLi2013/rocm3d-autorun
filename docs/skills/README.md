# docs/skills

This directory contains Cursor / Claude Code agent skills for generating ROCm install scripts.

| Subdirectory | Kind | Consumed by |
|---|---|---|
| `rocm-install-script-generator/` | Cursor / Claude Code Agent Skill | Cursor / Claude Code agent |

---

## rocm-install-script-generator

A Cursor / Claude Code skill that generates bash install/run scripts for ML repos targeting ROCm GPU nodes.

**Trigger:** User asks "给这个 repo 生成 ROCm install 脚本".  
**Output:** `samples/auto_gen/<repo>_install.sh` structured in Block A–H.

See [`rocm-install-script-generator/SKILL.md`](rocm-install-script-generator/SKILL.md).

---

## 经验沉淀通道

脚本生成和 auto-patch 的经验分别沉淀到不同位置：

| 经验类型 | 位置 | 消费者 |
|---|---|---|
| 新的 ROCm 兼容规则（Block 逻辑、EXCLUDE_PKGS 扩展等） | `rocm-install-script-generator/SKILL.md` | Cursor 生成脚本时 |
| 具体 error → patch 示例 | `src/docker_agent/prompts/analyzer_fewshot.md` | `llm_log_analyzer` auto-patcher |

**维护原则：**
- 每次新 repo 踩坑后：如果是通用规则，更新 `SKILL.md`；如果是可复用的 patch 模式，在 `analyzer_fewshot.md` 追加一个 `error signal → JSON patch` 示例。
- `analyzer_fewshot.md` 要求每个示例给出完整的错误特征 + 正确的 patch（op/match/content），LLM 直接模仿。
