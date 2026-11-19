# knowledge-reel-v2 · 知识短片工坊 v2

把一个选题做成可发布的竖屏中文知识短片（1080×1920 / 30fps / 约 4 分钟）。

**纸感手账视觉 + 3D emoji 图标 + 逐字卡点动效 + 人声优先混音。**

输入只有选题和少量特殊要求，其余全部自动决策；只在「关键帧」和「成片预览」两处停下等确认，交付前必须自检三遍。

---

## 这个 Skill 里有什么

| 文件 | 内容 |
|---|---|
| `SKILL.md` | 主文件：工作流、两个停车点、核心方法、视觉/叙事/文案/音效/合规/核查标准、交付清单 |
| `references/00-hard-bans.md` | **36 条明令禁止**（开拍前必读） |
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
