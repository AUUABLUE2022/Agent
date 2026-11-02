# video-agent

> 一个会**看视频**的 Agent：自己发现新视频、抓材料、转写、归类、归档、记状态、失败重试。给它一个触发器，就不用你管了。
>
> 给 DeepSeek Harness（DSH）用的技能包，也兼容任何支持 Markdown 指令 + Shell 的 Agent 框架。

## 这是 Agent 还是 Skill？

**它现在是 Agent**（v0.3 之前只是 Skill）。区别只有一条——**谁在决定下一步**：

| | 以前的 Skill | 现在的 Agent |
|---|---|---|
| 谁决定用不用 | 模型看着 description 决定 | 自己：`agent.mjs` 有主循环 |
| 谁决定做到哪一步 | 模型 | 自己：状态机 + 重试 + 放弃 |
| 谁触发 | 你丢链接 | 任务计划 / 常驻 / DSH 定时 |
| 模型还干什么 | 全部 | 只写"人读的那部分" |

`agent.mjs` 有 `run`（跑一轮）、`watch`（常驻循环）、`status`（看队列）。它自己判断哪些是新视频、哪些该重试、哪条够格归档，然后按模板产出笔记。模型只在"把模板笔记改成人话"这一步介入。

## 它解决什么问题

模型能读网页，但读不了视频。这个包把视频变成三样东西：

| 产物 | 从哪来 | 用途 |
|---|---|---|
| `sheet-01.png` … | B 站缩略图雪碧图，按秒采样拼成带时间码的联系表 | 让模型"看见"画面：PPT、代码、界面结构 |
| `transcript.txt` | 本地 vosk 中文模型离线转写 | 让模型"听见"旁白，拿到大意与话题切分 |
| `meta.json` / `frames.json` | 视频接口 | 标题、时长、分 P、每一帧的时间锚点 |

## 自主循环长什么样

```
发现新视频 → 抓 info/frames/asr → 按关键词初判学科 → 套模板写笔记
     ↑                                                    ↓
     └────── 失败进 retry，超次数标 failed ←──── 写 index.json + index.md
```

状态存在 `<课程库>/.agent-state.json`，每条记 `stage / attempts / reason / note / seconds`。跑多少遍都幂等。

自动生成的笔记长这样（真实产物，未编辑）：

```markdown
| 来源 | https://www.bilibili.com/video/BV1AP411A7PS/ |
| UP 主 | 瑞思棒瑟贝勒替 · 21:11 |
| 学科 | 待分类（关键词初判） |
| 采集 | 关键帧 20 张 / 转写 119 段 |

> 本文由 video-agent 自动生成：画面采样 + 本地转写，**未经模型校订**。

## 画面
![sheet-01.png](../../_inbox/BV1AP411A7PS/sheet-01.png)

## 时间轴
- `00:34` 正交设计的原理
- `01:37` 第一步是选择正确的合适的正交表……

## 待校订
- 学科初判为「待分类」，按标题关键词打分，可能偏；需要模型/人复核一次。
```

注意最后那节：**它知道自己哪里不确定**，把判断留给模型。这是"自动"和"瞎猜"的分界线。

## 实测数据（不是估计）

| 场景 | 结果 |
|---|---|
| 50:41 教程取帧 | 47 帧 / 4 张联系表，10 秒内 |
| 50:41 音频转写 | 162 段、16,624 字，约 15 分钟（3.6× 实时） |
| 21:11 讲义转写 | 119 段、3,660 字，约 5 分钟 |
| 自主跑一条完整流水线 | 155 秒：info → 20 帧 → 转写 → 写笔记 → 归档 → 更新索引 |
| 巡查一个 UP 主 | 7 个合集 185 条视频，秒级枚举，增量幂等 |

## 快速开始

```powershell
# 1) 装依赖（sharp / vosk / ffmpeg）
npm install

# 2) 下语音模型（约 42 MB，一次性）
node scripts/setup-model.mjs

# 3) 自检
node scripts/doctor.mjs

# 4) 让它自己跑一轮
node scripts/agent.mjs run --limit 2

# 5) 看它记了什么
node scripts/agent.mjs status
```

`agent.config.json` 里改 `targets`（追哪些 UP 主）、`vault`（放哪儿）、`asr`（要不要转写）。

## 三个触发器

| 触发器 | 怎么开 | 需要什么 |
|---|---|---|
| Windows 任务计划 | `triggers\register-task.ps1 -Hours 6` | 什么都不用开，但**没有模型**，只能出模板笔记 |
| 常驻循环 | `node scripts/agent.mjs watch --interval 120` | 一个不关的终端 |
| DSH 定时（带模型） | `schedule_create` + `triggers/dsh-schedule.md` 的 prompt | DSH 在运行，笔记质量最好 |

细节见 `triggers/README.md`。

## 装进 DSH（变成"能力"而不是"脚本"）

放进下面任一路径，DSH 会自动登记为技能（`SKILL.md` 的 frontmatter 就是登记信息）：

| 作用范围 | 路径 |
|---|---|
| 当前项目 | `<项目>/.dsh/skills/video-agent/` |
| 所有项目 | `~/.dsh/skills/video-agent/` |

之后输入 `/video-learn`，或者直接丢一个视频链接说"看完讲给我听"。

## 命令一览

```
agent.mjs  run     [--vault 目录] [--limit N] [--force BV] [--asr|--no-asr]
agent.mjs  status  [--vault 目录]
agent.mjs  watch   [--interval 分钟]

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
- **画面是缩略图级的**。单帧约 160×90，只够读大标题、界面结构、代码轮廓。
- **关键词分类会漏**。实测把"正交试验设计"归进了"待分类"——所以笔记里有「待校订」那一节，等模型或人接手。
- **任务计划那条路没有模型**。它能归档，但写不出解读。
- **散视频会漏**。UP 主列表走"合集/系列"接口（`space/arc/search` 无登录态会被风控），不在任何合集里的新视频抓不到。
- **只对 B 站做了适配**。其他站点改 `resolveTarget()`。

## 目录结构

```
video-agent/
├── SKILL.md               Agent 的操作手册（frontmatter = 技能登记信息）
├── README.md              本文件
├── INSTALL.md             手动安装步骤 + 排错
├── PUBLISH.md             发布清单
├── agent.config.json      追谁、放哪儿、开不开转写
├── install.ps1            一键安装
├── scripts/
│   ├── agent.mjs          自主循环：run / status / watch
│   ├── video.mjs          取帧 / 转写 / 字幕 / 音频 / 汇总
│   ├── watch.mjs          列视频、巡查、生成草稿
│   ├── setup-model.mjs    下载 libvosk.dll 与中文模型
│   └── doctor.mjs         自检
├── triggers/
│   ├── README.md          三种触发器怎么选
│   ├── register-task.ps1  Windows 任务计划
│   └── dsh-schedule.md    DSH 定时 prompt 模板
└── templates/
    ├── AGENTS.md          课程库的项目记忆示例
    ├── subjects.json      学科 → 关键词映射
    └── index.md           索引模板
```

## 设计取舍

- **为什么用 Node 而不是 yt-dlp + ffmpeg 全家桶**：目标机器上 curl/Python 的 TLS 可能被策略拦掉，Node 的 `fetch` 反而最稳；也少一个外部二进制依赖。
- **为什么用 vosk 而不是 whisper**：42 MB vs 500 MB+，CPU 上 3.6× 实时，够用。想换准确率，接口只有 `cmdAsr` 一处。
- **为什么自动版笔记不写解读**：没有模型就没有判断。模板保证"结构对、证据全、承认不确定"，解读交给下一次带模型的会话。
- **为什么转写默认开**：无人值守时有的是时间。

## 许可

见 `LICENSE`。B 站接口为非官方用法，请遵守站点规则、别用来批量抓取。
