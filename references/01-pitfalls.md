# 踩坑记录 · 错误目录

> 全部来自本项目真实发生过的失败。格式：**现象 → 根因 → 正解**。
> 遇到同类问题先查这张表，别重新试一遍。

---

## A. 视觉 / 图标

### A1 图标"太劣质，鼻子耳朵看不出来"
- **现象**：用户直接说"劣质"，指出鼻子耳朵认不出
- **根因**：参考片图标是 **3D 软陶/emoji**，我做成了**黑描边平涂线稿**。参考片其实是两层：3D 图标 + 扁平描边卡，我把图标层也做成了扁平的
- **正解**：图标用 3D emoji（Segoe UI Emoji 彩色字形）或真 3D 素材。**绝不手画器官**

### A2 emoji 渲染成空方块
- **现象**：`🪣 🫀 🫧 🫙 🫁` 画出来是空框
- **根因**：Segoe UI Emoji 缺少这些字形
- **正解**：**先做字形可用性测试**——写一个 HTML 网格把候选 emoji 全画出来，截图看哪些是空框，再定用哪几个
- **本项目实测可用**：👂 👃 👁️ 👄 🖐️ 📖 🧓 🍎 🔥 🐟 🏠 📡 ⏰ 🏺 🥣 🚽 🍶 🧴 💨 👴
- **实测不可用**：🪣 🫀 🫧 🫙 🫁

### A3 手画器官必然出错
- **现象**：耳朵像 C 形、鼻子像水滴、透视不对
- **根因**：解剖形态没法用几段贝塞尔曲线表达
- **正解**：不画。用 emoji / 素材 / 不做

### A4 红圈压字
- **现象**：用户截图指出红圈挡住「门」字
- **根因**：圈线画在字的笔画上
- **正解**：**关键词直接变红**。做法：先画整行黑字，再用 `rgba(红)` 按进度 alpha 覆盖那个字

### A5 文字重叠
- **现象**：难经段"三十四难"和"五脏各有五臭"叠在一起
- **根因**：两个元素用了同一个 y 坐标（800）
- **正解**：同级元素分层留距（书名 812 / 解释 900），**交付前逐帧核**

### A6 元素堆叠
- **现象**：开头六拍全部同时出现在屏幕上互相压
- **根因**：只在"出现"挂钩子，**没做"退场"**
- **正解**：一拍一幕
  ```js
  const A = i => {
    if (K === i)   return outCubic(cl((t - T(i)) / 0.30));      // 当前拍淡入
    if (K === i+1) return 1 - outCubic(cl((t - T(i+1)) / 0.24)); // 上一拍淡出
    return 0;
  };
  ```

---

## B. 同步

### B1 画面跑在旁白前面
- **现象**：字幕说"儿子接着整理"，画面已经把四个节点全画完了
- **根因**：元素用 `u = t - seg.t0` 的**固定秒数**出场
- **正解**：改成逐字卡点 `wt(句号, 字)`。本项目 18 段全部改造完

### B2 逐字时间按字数等比估算不准
- **现象**：字幕和插画差 0.3–0.6 秒
- **根因**：TTS 在顿号、句号处停顿，字数相等 ≠ 时长相等
- **正解**：用 whisper 词级时间戳重建 CHAR_T（见 `02-method-sync.md`）

### B3 分支按秒切 → 错位
- **现象**：25 秒字幕还在讲「门」，画面已跳到甲骨文三卡
- **根因**：`if (u < 7.8)` 硬编码秒数，配音时长一变就错
- **正解**：`if (li0(t) - SEG_L0[seg] < 3)` **按句切**

### B4 字幕行间空档 → 闪一帧
- **现象**：自检脚本报 193.47s 有一帧没有字幕
- **根因**：`t1 = t0 + dur + 0.02`，下一句 `t0' = t1 + 0.10`，中间 0.08s 无人覆盖
- **正解**：`curLine` 返回"最后一句已开始的"，保持到下一句开始，不留缝

### B5 顿号密集的句子永远超窗
- **现象**：「臊、焦、香、腥、腐」需要 7.2 字/秒，TTS 做不到
- **根因**：每个顿号都要停顿，实际语速远低于字数估算
- **正解**：**两遍法**——先合成量真实时长，再按真实时长排时间轴，不要按字/秒估算

---

## C. 音频

### C1 "还有沙沙声，跟前面没区别"
- **现象**：改了配乐用户仍说没区别
- **根因**：**真正来源不是配乐**
  1. edge-tts 旁白自带底噪：8–12kHz 比 4–8kHz 还高（正常语音到 8k 就该掉了）
  2. `ffmpeg loudnorm` **动态模式把高频抬高了 7dB**
- **正解**：
  1. 旁白过 `highshelf=f=7000:g=-4.5:width_type=q:width=1,lowpass=f=13000:poles=2`
  2. loudnorm 改**两遍法 + `linear=true`**（只做静态增益，不改动态和音色）
- **验证**：修复后 8k 从 +0.5dB 降到 −9.8dB，16k 从 +0.1 降到 −39.1

### C2 hi-hat 就是"沙沙"
- **现象**：背景音乐一直有"嚓嚓"噪声
- **根因**：starter 模板是电子舞曲编配：`S.hat()` 每拍一次 + clap + crash + riser
- **正解**：知识片不要这些。改古琴拨弦（羽调式）+ 弦乐铺底（cutoff 900）+ sub/bass + 钟声

### C3 24-bit PCM 当 16-bit 读
- **现象**：测出"平谱噪声"、文件时长显示 1.5 倍
- **根因**：`audio.mjs` 输出 `pcm_s24le`（3 字节/样本），按 `int16` 解码
- **正解**：分析前先 `ffmpeg -c:a pcm_s16le` 转一遍

### C4 音频时长不匹配
- **现象**：`build.sh` 报 `prepared audio duration conflicts with timeline`
- **根因**：混音时多写了 1 秒缓冲
- **正解**：`N = round(DUR * SR)`，**精确等于 DUR**（容差只有 1.5/fps ≈ 0.05s）

### C5 TTS 实际语速与假设不符
- **现象**：大量句子被截尾
- **根因**：按 5.1 字/秒排时间轴，edge-tts 实测 **4.2 字/秒**
- **正解**：两遍法。永远不要相信"字/秒"这个估算

---

## D. 渲染 / 代码

### D1 `u` 未定义导致整片崩溃
- **现象**：渲染到 40% 抛 `ReferenceError: u is not defined`
- **根因**：重写段落时删了 `const u = at(t,'hook')`，但声波那行还在用
- **正解**：**改完立刻渲 3–6 张关键帧**，不要直接跑整片（整片 4 分钟，静帧 30 秒）

### D2 `ctx.filter = 'blur()'` 慢
- **根因**：canvas filter 走软件光栅
- **正解**：软阴影用**径向渐变**画，不用 filter

### D3 括号不平衡
- **现象**：`Unexpected token '}'`
- **根因**：把 `if(x){ ... }` 改成 `const a=...;` 时删了 `{` 但留着配对的 `}`
- **正解**：替换 `if` 为语句时，必须同时检查配对的 `}`

### D4 PowerShell 引号地狱
- **现象**：`*`、`&&`、中文引号在 `pwsh -c "..."` 里被解析成表达式
- **根因**：PowerShell 解析器
- **正解**：**所有含特殊字符或多行的脚本，写成 `.py` 文件再执行**，不要在命令行里内联

### D5 write 工具要求先读
- **现象**：`cannot write: file changed since it was read` / `file no longer exists`
- **正解**：先 `read` 一遍；或先删除→新建空文件→`read` 空文件→再写

### D6 文件被播放器占用
- **现象**：`Copy-Item` 报"正由另一进程使用"
- **正解**：`Get-Process wmplayer | Stop-Process -Force` 后再拷

### D7 Chromium fullPage 截图失败
- **现象**：`Protocol error (Page.captureScreenshot): Unable to capture screenshot`
- **正解**：去掉 `fullPage: true`，把 viewport 设准

### D8 playwright 找不到
- **现象**：`ERR_MODULE_NOT_FOUND`
- **正解**：脚本要放在 `garden-skills/skills/motion-showreel/scripts/` 目录下跑（node_modules 在那里）

### D9 中文控制台编码崩
- **现象**：`UnicodeEncodeError: 'gbk' codec can't encode character '\u2713'`
- **正解**：Python 脚本里**只用 ASCII 日志**（`OK` 而不是 `✓`）；或设 `PYTHONIOENCODING=utf-8`

---

## E. 环境 / 依赖

### E1 图像生成不可用
- `image_generate` → `saving images failed: Declared image type does not match its bytes`
- `dsh_subscriptions_image_generate` → `not logged in to ChatGPT (Codex)`
- **正解**：**快速失败**，立刻改用 emoji 或代码绘制，不要反复重试

### E2 npm 装不上
- **现象**：`ENOTFOUND bnpm.byted.org`
- **根因**：作者仓库的 `package-lock.json` 指向内网源
- **正解**：删 `package-lock.json`，单独装 playwright，`--registry=https://registry.npmmirror.com --no-package-lock`

### E3 faster-whisper 与 PyAV 版本冲突
- **现象**：`open() got an unexpected keyword argument 'metadata_errors'`
- **正解**：**传 numpy array 而不是文件路径**

### E4 ffmpeg 参数失效
- `-vsync vfr` 不支持 → 用 `-fps_mode vfr`

---

## F. 流程

### F1 没验收就交付
- **现象**：用户明确批评"你做完以后需要自行检查…再给我"
- **正解**：见 `03-qa.md`，交付前必须跑三遍检查

### F2 反复在同一个错误上打转
- **现象**：多次"修复"用户仍说没区别
- **根因**：**没有找到真正的根因就动手**
- **正解**：先做**隔离实验**（只留一个变量），再改。例：把音频拆成"旁白 / 配乐 / 混音"三条分别测频谱
