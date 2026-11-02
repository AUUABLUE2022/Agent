# video-agent

> 一个会**看视频**的 Agent：把 B 站（以及任何能拿到元信息的网页视频）拆成能读的文字与画面，看完写笔记，还能定时追更。
>
> 给 DeepSeek Harness（DSH）用的技能包，也兼容任何支持 Markdown 指令 + Shell 的 Agent 框架。

---

## 它解决什么问题

模型能读网页，但读不了视频。这个包把视频变成三样东西：

| 产物 | 从哪来 | 用途 |
|---|---|---|
| `sheet-01.png` … | B 站缩略图雪碧图，按秒采样拼成带时间码的联系表 | 让模型"看见"画面：PPT、代码、界面结构 |
| `transcript.txt` | 本地 vosk 中文模型离线转写 | 让模型"听见"旁白，拿到大意与话题切分 |
| `meta.json` / `frames.json` | 视频接口 | 标题、时长、分 P、每一帧的时间锚点 |

外加一个**课程库**模式：定时巡查某个 UP 主，把新视频的材料自动抓进 `_inbox/`，由 Agent 写成笔记归档到 `学科/`。

## 实测数据（不是估计）

| 场景 | 结果 |
|---|---|
| 50:41 教程视频取帧 | 47 帧 / 4 张联系表，10 秒内完成 |
| 50:41 音频转写 | 162 段、16,624 字，约 15 分钟（3.6× 实时） |
| 21:11 讲义转写 | 119 段、3,660 字，约 5 分钟 |
| 巡查一个 UP 主 | 7 个合集 185 条视频，秒级枚举，增量幂等 |

## 快速开始

```powershell
# 1) 装依赖（sharp / vosk / ffmpeg）
npm install

# 2) 下语音模型（约 42 MB，一次性）
node scripts/setup-model.mjs

# 3) 自检
node scripts/doctor.mjs

# 4) 用
node scripts/video.mjs info   "BV1REx46NEc4"
node scripts/video.mjs frames "BV1REx46NEc4" --every 60 --max 48
node scripts/video.mjs asr    "BV1REx46NEc4"          # 后台跑，别干等
```

## 装进 DSH（变成"能力"而不是"脚本"）

把整个目录放进下面任一路径，DSH 会自动把它登记为技能（`SKILL.md` 的 frontmatter 就是登记信息）：

| 作用范围 | 路径 |
|---|---|
| 当前项目 | `<项目>/.dsh/skills/video-agent/` |
| 所有项目 | `~/.dsh/skills/video-agent/` |

之后在对话里输入 `/video-learn`，或者直接丢一个视频链接说"看完讲给我听"，Agent 会自己走流程。

`install.ps1` 能一步到位（复制 + 装依赖 + 下模型 + 自检）。

## 命令一览

```
video.mjs  info    <url|BV>  元信息
video.mjs  frames  <url|BV>  [--every 秒] [--max 张] [--cols 4] [--rows 3]
video.mjs  asr     <url|BV>  [--max-seconds N] [--audio 文件] [--model 目录]
video.mjs  audio   <url|BV>  [--quality low|mid|best]
video.mjs  subs    <url|BV>  [--cookie-file 文件] [--verbose]
video.mjs  report  <url|BV>  汇总成 video.md
watch.mjs  --mid <UP> --vault <目录> [--limit 3] [--asr] [--dry-run]
```

## 诚实的边界

- **没有字幕就没有逐字稿**。B 站多数视频的字幕要登录态，包里留了 `--cookie-file` 的钩子，但要你自己提供 `SESSDATA`。
- **小模型听不准专有名词**。实测把 "DSH" 听成"定性蚶子"、"Harness" 听成"哈里斯"。转写只用于定大意和找时间点，**引用原话必须用官方字幕**。
- **画面是缩略图级的**。单帧约 160×90，只够读大标题、界面结构、代码轮廓，读不清小字。
- **散视频会漏**。UP 主列表走"合集/系列"接口（`space/arc/search` 无登录态会被风控），不在任何合集里的新视频抓不到。
- **只对 B 站做了适配**。其他站点需要各自的接口，`resolveTarget()` 是唯一的接入点。

## 目录结构

```
video-agent/
├── SKILL.md               Agent 的操作手册（frontmatter = 技能登记信息）
├── README.md              本文件
├── INSTALL.md             手动安装步骤 + 排错
├── install.ps1            一键安装
├── package.json           依赖声明
├── scripts/
│   ├── video.mjs          取帧 / 转写 / 字幕 / 音频 / 汇总
│   ├── watch.mjs          巡查 UP 主、建基线、抓材料、生成草稿
│   ├── setup-model.mjs    下载 libvosk.dll 与中文模型
│   └── doctor.mjs         自检
└── templates/
    ├── subjects.json      学科 → 关键词映射
    └── index.md           课程库索引模板
```

## 设计取舍

- **为什么用 Node 而不是 yt-dlp + ffmpeg 全家桶**：目标机器上 curl/Python 的 TLS 可能被策略拦掉，Node 的 `fetch` 反而是最稳的；也少一个外部二进制依赖。
- **为什么用 vosk 而不是 whisper**：42 MB vs 500 MB+，CPU 上 3.6× 实时，够用。想要准确率就换大模型或云端 ASR，接口只有 `cmdAsr` 一处。
- **为什么转写不默认开**：50 分钟要跑 15 分钟。默认只取帧，`--asr` 显式开。

## 许可

见 `LICENSE`。B 站接口为非官方用法，请遵守站点规则、别用来批量抓取。

