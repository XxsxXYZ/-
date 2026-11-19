# 核心方法 · 两遍法 + 逐字卡点 + 场景制

> 这三样是"让片子不像 PPT"的全部技术底座。缺一样就会退回幻灯片。

---

## 一、两遍法（先量真实配音，再排时间轴）

### 为什么必须两遍

按"字数 ÷ 每秒字数"估时间轴，一定会错：

| 假设 | 实测 |
|---|---|
| edge-tts 语速 5.1 字/秒 | **4.2 字/秒** |
| 顿号不停顿 | **每个顿号停 0.2–0.4 秒** |

后果：大量台词被截尾、`build.sh` 拒绝、画面和声音整体错位。

### 第一遍：合成配音并测量

```python
# 逐句合成，缓存到 audio/_tts/l{index:03d}.mp3
tts_one(text, mp3, rate="+25%")
d = ffprobe_duration(mp3)          # 真实时长
```

- **逐句合成**，不要整段合成（整段取不到句级时间）
- **缓存**：只删改动过的句子的 mp3，其余复用 → 改一句只重录一句
- 语速建议 **+25%**（4.2 × 1.25 ≈ 5.25 字/秒，接近中文知识片舒适区）

### 第二遍：按真实时长铺时间轴

```python
t = 0.0
for line in lines:
    line.t0 = t
    line.t1 = t + line.dur + 0.02
    t += line.dur + GAP          # GAP = 0.12
DUR = t + 1.0
```

**再反推段落边界**：每段第一句的 t0 就是段落的 t0。

### 产出 timeline.js 的四张表

| 表 | 内容 | 用途 |
|---|---|---|
| `DUR` | 总时长 | 渲染帧数、音频长度 |
| `SHOTS` | 18 段的 `{id, t0, t1}` | 段落切换 |
| `LINES` | 68 句的 `{t0, t1, s, kw}` | 字幕、字幕关键词高亮 |
| `CUES` | 67 个 `{t, sfx}` | 音效卡点 |
| `CHAR_T` | 每句每个字的**发音时刻** | **逐字卡点**（见下节） |

---

## 二、逐字卡点（画面挂在"字"上，不挂在"秒"上）

### 关键数据：CHAR_T

`CHAR_T[i][j]` = 第 i 句第 j 个字的发音时刻（秒）。

**用 whisper 生成，不要按字数等比估算。**

```python
# 1) 对整条 narration.wav 跑词级时间戳
model = WhisperModel("faster-whisper-small", device="cpu", compute_type="int8")
segs, _ = model.transcribe(audio_array, language="zh",
                           word_timestamps=True, vad_filter=False)
# 注意：传 numpy array，不要传文件路径（PyAV 版本冲突）

# 2) 把词时间对齐到每个字
for w in words_for_this_line:
    for k, ch in enumerate(w["w"]):
        t = w["t0"] + (w["t1"] - w["t0"]) * k / max(len(w["w"]), 1)
# 对不上（TTS 跳过标点）的位置插值补全
```

- 实测效果：「望」按字数估算是 1.14s，whisper 给出 **1.02s**
- 一个 245 秒片子约 **1000 个词**，CPU small 模型 2–3 分钟

### 检索函数：wt / cueIn

```js
// 在第 li 句里找第 nth 次出现的字 ch
function wt(li, ch, nth = 0) {
  const L = LINES[li], arr = CHAR_T[li] || [];
  let idx = -1;
  for (let k = 0; k <= nth; k++) { idx = L.s.indexOf(ch, idx + 1); if (idx < 0) break; }
  if (idx < 0 && nth > 0) idx = L.s.indexOf(ch);   // 找不到退回第 0 次
  if (idx < 0) return L.t0;
  return arr[idx] !== undefined ? arr[idx] : L.t0;
}

// 在整个段落里跨句找词组
function cueIn(segId, phrase, nth = 0) {
  const st = S(segId).t0, en = S(segId).t1;
  let n = 0;
  for (let i = 0; i < LINES.length; i++) {
    const L = LINES[i];
    if (L.t1 < st - 0.1 || L.t0 > en + 0.1) continue;
    const arr = CHAR_T[i] || [];
    let idx = -1;
    while (true) {
      idx = L.s.indexOf(phrase, idx + 1);
      if (idx < 0) break;
      if (n === nth) return arr[idx] !== undefined ? arr[idx] : L.t0;
      n++;
    }
  }
  return st;    // 找不到就退回段首
}
```

### 用法

```js
const k = outBack(cl((t - cueIn('decode','烧脂肪')) / 0.40));
if (k > 0.01) { /* 只有念到"烧脂肪"这张卡才落 */ }
```

**不许再出现** `pop(u, 6.0, ...)` 这种按秒的写法。

### 坑：词组不存在
- `C(2,'没',1)` 但句子里只有一个「没」→ 退回句首 → 叉提前 3 秒出现
- `C(4,'听')` 但第 5 句根本没有「听」字 → 同上
- **正解**：`wt` 内置"退回第 0 次"；关键元素实在没有对应词，就用 `S(seg).t0 + 0.12` 按段首出场

---

## 三、场景制（一拍一幕）

### 为什么

一张画布只允许一种构图。上一拍的元素必须**退场**，否则六拍全堆在一起互相压。

### 结构

```js
if (inSeg(t, 'hook')) {
  const l0 = SEG_L0['hook'];
  const K  = li0(t) - l0;                       // 现在是第几拍
  const T  = i => LINES[l0 + i] ? LINES[l0 + i].t0 : 1e9;
  const A  = i => {
    if (K === i)   return outCubic(cl((t - T(i)) / 0.30));       // 入场
    if (K === i+1) return 1 - outCubic(cl((t - T(i+1)) / 0.24));  // 退场
    return 0;
  };
  const vis = i => A(i) > 0.004;

  if (vis(0)) { /* 只画第 1 拍 */ }
  if (vis(1)) { /* 只画第 2 拍 */ }
  ...
}
```

### 分支也要按句切

```js
// ✗ 错：秒数一变就错位
if (u < 7.8) { ... }

// ✓ 对：按句切
if ((li0(t) - SEG_L0['word']) < 3) { ... }
```

### 空白兜底
每拍开头必须有元素。若第一张卡要等某个词，先立一个标题/符号顶上，**空白不超过 1.5 秒**。

---

## 四、持续运动（让画面"活着"）

所有贴纸和文字按**自身坐标取不同相位**缓慢漂移：

```js
let __t = 0;
function drift(x, y) {
  const ph = (x * 0.0137 + y * 0.0171) % 6.283;
  return {
    dx: Math.sin(__t * 0.72 + ph) * 3.2 * U,
    dy: Math.cos(__t * 0.58 + ph) * 3.0 * U,
    rot: Math.sin(__t * 0.41 + ph) * 0.7,
  };
}
```

在 `gl()` 和 `sticker()` 内部接入 `drift()`，一次改动全片生效。
`render(t)` 开头设 `__t = t`。

---

## 五、双层视觉：3D 图标 + 扁平贴纸卡

参考片不是单一风格，是**两层叠出来的**：

| 层 | 样式 | 实现 |
|---|---|---|
| **物件层** | 3D 软陶 / emoji：无黑描边、径向渐变打光、左上高光斑、彩色落地柔影 | `icons.js` — `ctx.fillText('👂', x, y)` |
| **卡片层** | 扁平 + 黑描边 5U + 零模糊偏移投影 8U + 主题色填充 | `sticker()` |

**不要**把物件层也做成黑描边线稿——这是本项目最大的一个返工。

### emoji 图标实现要点

```js
const FONT_EMOJI = '"Segoe UI Emoji","Apple Color Emoji","Noto Color Emoji",sans-serif';

export function glyph(o, ch, x, y, s = 1, a = 1) {
  const c = o.ctx, size = 128 * s * o.U;
  shadow(o, x, y, size, a);              // 径向渐变画软影（不用 ctx.filter）
  c.save();
  c.globalAlpha = a;
  c.font = size + 'px ' + FONT_EMOJI;
  c.textAlign = 'center'; c.textBaseline = 'middle';
  c.fillText(ch, x, y);
  c.restore();
}
```

- 必须 `textBaseline = 'middle'`，否则不同 emoji 的基线不齐
- 落地影用**径向渐变椭圆**，不要 `ctx.filter = 'blur()'`（慢）
- 使用前**必须先测字形支持**（见 `01-pitfalls.md` A2）

---

## 六、角色与图标可以不同风格

参考片里 **2D 平涂 Q 版角色 + 3D emoji 图标** 是共存的，不冲突。
不必强行统一，但角色必须只做三件事：呼吸缩放、呆毛/小动作、位置切换。
