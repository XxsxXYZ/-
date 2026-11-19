# 工具链 · 完整命令与配置

> 本项目实际使用的全部工具、版本、安装方式、命令。
> 换机器照着装一遍就能跑。

---

## 一、总览

```
选题 ──► 脚本(v5.py) ──► edge-tts 逐句合成 ──► whisper 词级时间戳
                                                      │
                                                      ▼
                                            relayout.py 两遍法排时间轴
                                                      │
                                                      ▼
                              timeline.js (DUR/SHOTS/LINES/CUES/CHAR_T)
                                                      │
                    index.html (render(t))  ◄──────────┤
                                                      │
                                                      ▼
                          motion-showreel render.mjs ──► 7367 张 PNG
                                                      │
                                                      ▼
                                    ffmpeg 合成 + 混音 ──► final.mp4
```

---

## 二、渲染引擎：motion-showreel

| 项 | 值 |
|---|---|
| 来源 | `github.com/ConardLi/garden-skills` |
| 许可 | MIT |
| 作者 | code秘密花园 / ConardLi |
| Star | 15.4k |

### 目录

```
garden-skills/skills/motion-showreel/
├── scripts/
│   ├── scaffold.sh       建项目脚手架
│   ├── render.mjs        逐帧渲染 + ffmpeg 合成
│   ├── inspect.mjs       QA 技术检查
│   ├── audio.mjs         配乐/音效合成与检查
│   ├── narration.mjs     配音对齐检查
│   ├── lib/synth.mjs     合成器（20+ 音色）
│   └── node_modules/     ← playwright 装在这里！脚本必须在此目录跑
├── references/
│   ├── sound.md          音色 API 全表
│   ├── review.md         成片审阅标准
│   └── ...
└── SKILL.md
```

### 安装 playwright

```bash
rm -f package-lock.json          # 作者 lock 指向内网源 bnpm.byted.org，必删
npm install playwright --registry=https://registry.npmmirror.com --no-package-lock
export PLAYWRIGHT_DOWNLOAD_HOST=https://cdn.npmmirror.com/binaries/playwright
npx playwright install chromium
```

---

## 三、项目三件套

```bash
sh <engine>/scripts/scaffold.sh <slug> --audio synth
```

只改三个文件：

| 文件 | 职责 |
|---|---|
| `<slug>/index.html` | 视觉 — 实现 `window.render(t)` |
| `<slug>/timeline.js` | 时间 — `DUR / SHOTS / LINES / CUES / CHAR_T` |
| `<slug>/score.mjs` | 声音 — BGM + 音效 |

### 硬规则（违反必出问题）

- 画面只由 `t` 决定；**不用 `Math.random` / `Date.now` / 计时器 / CSS 动画**
- 随机数用固定种子
- 字体全部加载完再 `window.READY = true`
- 布局从 `stageSize()` 推导，不写死像素

---

## 四、常用命令

```bash
# 渲关键帧（拼成一张对比图）
node scripts/render.mjs <slug> --still 22,25,28 --size 1080x1920 --sheet

# 单帧
node scripts/render.mjs <slug> --still 150 --size 1080x1920

# 单段重渲（改一处只渲这一段）
node scripts/render.mjs <slug> --fps 30 --size 1080x1920 \
     --start 140 --duration 20 --out out/seg.mp4

# 可复现性检查
node scripts/render.mjs <slug> --fps 30 --size 1080x1920 --determinism

# 完整构建（推荐）
FPS=30 SIZE=1080x1920 AUDIO=file AUDIO_FILE=audio/final.wav sh build.sh
```

`build.sh` 会依次做：音频校验 → 可复现检查 → 渲帧 → 出片 → 文字检查 → QA。

---

## 五、配音：edge-tts

```python
import edge_tts, asyncio
async def go():
    c = edge_tts.Communicate(text, "zh-CN-XiaoxiaoNeural", rate="+25%")
    buf = b""
    async for ch in c.stream():
        if ch["type"] == "audio": buf += ch["data"]
    open(mp3, "wb").write(buf)
asyncio.run(go())
```

| 项 | 值 |
|---|---|
| 音色 | `zh-CN-XiaoxiaoNeural` |
| 语速 | `+25%`（约 5.25 字/秒） |
| 缓存 | `audio/_tts/l{index:03d}.mp3` |
| 实测语速 | **+0% 时 4.2 字/秒**（不是 5.1） |

> edge-tts 中文只给 SentenceBoundary，**没有词级时间戳**，所以必须接 whisper。

---

## 六、词级时间戳：faster-whisper

```python
from faster_whisper import WhisperModel
model = WhisperModel("faster-whisper-small", device="cpu", compute_type="int8")
segs, info = model.transcribe(
    audio_numpy_array,        # ← 传 numpy array，不要传文件路径（PyAV 版本冲突）
    language="zh",
    word_timestamps=True,
    vad_filter=False,
    beam_size=1,
    condition_on_previous_text=False,
)
```

| 项 | 值 |
|---|---|
| 版本 | faster-whisper 1.2.1 |
| 模型 | `faster-whisper-small`（约 480MB，也可用 medium） |
| 耗时 | 245 秒音频 → 2–3 分钟（CPU int8） |
| 产出 | 约 1000 个词的时间戳 |

---

## 七、音频处理：ffmpeg

### 旁白去嘶

```bash
ffmpeg -i narration.wav \
  -af "highshelf=f=7000:g=-4.5:width_type=q:width=1,lowpass=f=13000:poles=2" \
  -ac 1 -ar 44100 -c:a pcm_s16le narration_eq.wav
```

### 两遍法响度归一（静态增益，不改音色）

```bash
# 第一遍：测量
ffmpeg -i mix.wav -af "loudnorm=I=-14:TP=-1.5:LRA=11:print_format=json" -f null -

# 第二遍：应用（linear=true 只做静态增益）
ffmpeg -i mix.wav \
  -af "loudnorm=I=-14:TP=-1.5:LRA=11:measured_I=..:measured_TP=..:\
measured_LRA=..:measured_thresh=..:offset=..:linear=true" \
  -ac 1 -ar 44100 -c:a pcm_s16le final.wav
```

> ❌ **不要用动态模式**（不加 `linear=true`），会把高频抬 7dB → 听感"沙沙"。

### 频谱分析前先转 16-bit

```bash
ffmpeg -i x.wav -ac 1 -ar 48000 -c:a pcm_s16le _t.wav
```

> `audio.mjs` 输出 `pcm_s24le`，按 int16 解码会得到假噪声和 1.5 倍时长。

### 抽帧总检图

```bash
ffmpeg -i out/final.mp4 -vf "fps=1/9.82,scale=320:-1,tile=5x5" -frames:v 1 qa/check.png
```

---

## 八、音色速查（motion-showreel synth）

| 类别 | 函数 |
|---|---|
| 鼓 | `kick()` `snare()` `clap()` `hat()` `crash()` `tom(note)` |
| 手鼓 | `shaker()` `rim()` `cowbell()` |
| 低音 | `bass(note,dur)` `sub(note,dur)` |
| 和声 | `pad(notes,dur)` `keys()` `organ()` `vox()` |
| 旋律 | `pluck(note,dur)` `mallet(note,dur)` `lead()` `bell(note,dur)` |
| 音效 | `riser()` `whoosh(dur)` `impact()` `boom()` `click()` `tick()` `pop(note)` `blip()` `swipe()` `glitch()` `shimmer()` |
| 环境 | `ambience(dur,{kind})` |

**知识片默认编配**：`pad`（cutoff 900）+ `sub/bass` + `pluck`（bright 0.22 / damp 0.62 接近古琴）+ `bell`（片头片尾）。
**禁用**：`hat` `clap` `crash` `riser` `shaker` —— 这些是"沙沙"的来源。

---

## 九、浏览器自动化（自检用）

```js
import { chromium } from 'playwright';
const browser = await chromium.launch();
const page = await browser.newPage({ viewport: { width: 1200, height: 1500 }, deviceScaleFactor: 2 });
await page.goto('file://' + path.resolve(file));
await page.waitForTimeout(600);
await page.screenshot({ path: out });   // 不要加 fullPage，会报 Protocol error
await browser.close();
```

**脚本必须放在 `garden-skills/skills/motion-showreel/scripts/` 下**（playwright 在那里的 node_modules）。

---

## 十、Python 依赖

```
numpy
Pillow
edge-tts
faster-whisper      1.2.1
```

---

## 十一、环境备忘

| 项 | 值 |
|---|---|
| Python | `C:\Python312\python.exe` |
| Node | `C:\Program Files\nodejs\node.exe` |
| Git Bash | `C:\Program Files\Git\bin\bash.exe` |
| ffmpeg | 需在 PATH |
| Chromium | 由 playwright 安装管理 |

> **所有含特殊字符的脚本都写成 `.py` 文件再执行**，不要在 `pwsh -c "..."` 里内联。
