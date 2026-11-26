# xyz · 知识短片工坊 v2（中文知识短片制作技能）

把一个选题做成可发布的竖屏中文知识短片（1080×1920 / 30fps / 约 4 分钟）。

**纸感手账视觉 + 3D emoji 图标 + 逐字卡点动效 + 人声优先混音。**

输入只有选题和少量特殊要求，其余全部自动决策；只在「关键帧」和「成片预览」两处停下等确认，交付前必须自检三遍。

---

## 这个 Skill 里有什么

| 文件 | 内容 |
|---|---|
| `SKILL.md` | 主文件：工作流、两个停车点、核心方法、视觉/叙事/文案/音效/合规/核查标准、交付清单 |
| `references/00-hard-bans.md` | **36 条明令禁止**（开拍前必读） |
| `references/06-research-gate.md` | **研究关卡**：写稿前四问 + 输出模板（v3 新增） |
| `checkpoint.yaml` | 停车点策略 + 时间预算 + 硬红线（v3 新增） |
| `references/01-pitfalls.md` | **28 个真实踩坑**：现象 → 根因 → 正解 |
| `references/02-method-sync.md` | 两遍法 / 逐字卡点 / 场景制 / 持续运动 / 双层视觉 的实现 |
| `references/03-qa.md` | 三遍验收流程与判定标准 + 降本增效对照表 |
| `references/04-tutorial-8tips.md` | 教学视频 8 条技巧**原文摘录** + 落地对照 |
| `references/05-toolchain.md` | 完整命令：motion-showreel / edge-tts / whisper / ffmpeg / playwright |
| `references/style-paper.md` | 纸感手账视觉系统全部参数 |
| `references/narrative.md` | 阶梯式叙事骨架 + 四拍子 |
| `references/copywriting.md` | 文案七项指标 + 五刀 |
| `references/compliance.md` | 中医/健康题材合规红线 |
| `references/sfx.md` | 音效类型与卡点密度 |
| `references/brief-template.md` | 结构化提示词模板 |

---

## 蒸馏来源

1. `ConardLi/garden-skills` 的 **motion-showreel** 渲染引擎（MIT，15.4k★）
2. 《Codex 能力分 5 级》的**视觉系统与叙事骨架**拆解
3. 《AI做动画视频完全指南》教学片的 **8 条技巧**
4. 一个中医科普项目（《闻诊：望闻问切的「闻」到底在闻什么》）的**完整制作过程**
5. 用户逐条下达的**明令禁止**与纠正记录

---

## v3 新增（2026-11）

从 [OpenMontage](https://github.com/OpenMontageInc/OpenMontage)（8.3 万星）借了两样**管理方法**，没抄任何代码、没引入任何依赖、许可仍是 MIT：

1. **研究关卡**（`references/06-research-gate.md`）—— 写稿前必须凑够 ≥3 同类内容 + ≥1 差异点 + ≥5 条 A 档事实。把"事实错误"从靠自觉变成硬门槛。
2. **结构化停车点**（`checkpoint.yaml`）—— 七关各配 `checkpoint` / `ask_human` / `pass_when`，外加时间预算与硬红线。

文件数 16 → 18，行数 1600 → 2100，**没有引入 schema 校验器、没有 pipeline runner**（那是另一个量级的工程）。

---

## 三条最值钱的结论

1. **两遍法**：永远不要按"字/秒"估时间轴。edge-tts 实测 4.2 字/秒，顿号还会额外停顿。先合成量真实时长，再排时间轴。
2. **逐字卡点**：画面要挂在"第几个字"上，不是"第几秒"上。用 whisper 词级时间戳生成 `CHAR_T`。
3. **一拍一幕**：上一拍必须退场，否则一定堆成幻灯片。分支也按句切，不按秒切。

---

## 最短上手路径

```bash
sh <engine>/scripts/scaffold.sh <slug> --audio synth
# 改三个文件：index.html / timeline.js / score.mjs
node <engine>/scripts/render.mjs <slug> --still 22,25,28 --sheet   # 先看静帧
FPS=30 SIZE=1080x1920 AUDIO=file AUDIO_FILE=audio/final.wav sh build.sh
```

---

## License

MIT。引擎部分来自 `ConardLi/garden-skills`（MIT）。
