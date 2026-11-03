# SJTU Canvas 资源获取

从 skill 根目录使用 `.venv/bin/sjtu-canvas`。本文件中的命令由
`third_party/sjtu-canvas-assistant/` 提供。
环境安装和根目录 `.env` 配置见 [README](../README.md)。
首次配置后先用 `.venv/bin/sjtu-canvas whoami` 检查认证，再查询课程。
这是网络请求；没有有效凭证时应引导本地配置，不把帮助命令通过当成认证成功。

## 查询与下载

```bash
.venv/bin/sjtu-canvas courses --json
.venv/bin/sjtu-canvas modules <course-id> --json
.venv/bin/sjtu-canvas plan <course-id>
.venv/bin/sjtu-canvas download <course-id> --output data/materials
.venv/bin/sjtu-canvas video list <course-id> --json
mkdir -p data/recordings/<course-id>/<video-id>
.venv/bin/sjtu-canvas video subtitle <course-id> <video-id> --output data/recordings/<course-id>/<video-id>/transcript.srt
.venv/bin/sjtu-canvas video ppt-export <course-id> <video-id> --output data/recordings/<course-id>/<video-id>/ppt --json > data/recordings/<course-id>/<video-id>/ppt-export.json
```

- 查询结果决定课程和视频 ID，不根据本仓库历史例子猜 ID。
- `plan` 不下载文件，但会访问真实 Canvas API；`download` 下载该课程文件，
  按课程/文件夹保留结构并生成 `course_index.json` 和 `course_index.md`。
- `video subtitle` 保存 SRT，不需要下载视频。没有字幕时返回 `availability=unavailable`，
  `srt_path=null`，不生成空文件或覆盖原文件；不能视为已取得完整课堂内容。
  需要原始字幕字段时可加 `--json` 保存到本地 `data/`。
- `video ppt-export` 保存原始切片图片和默认合成的 `ppt.pdf`；JSON 结果包含切片时间
  与本地图片的对应关系。上面的重定向将它另存为 `ppt-export.json`，工具不会自动写
  清单文件。只需图片时使用 `--no-pdf`。AI 切片缺失时返回
  `availability=unavailable`，不会创建 PDF；已有 PDF 不代表本次导出成功。
- 命令中的 `data/...` 是默认本地工作区示例；用户已有工作目录时替换为其实际路径。
  向 skill 之外的课程目录输出时使用绝对路径。调用前检查现有文件，避免覆盖
  已有讲义或特意保留的字幕与图片。

通常只需查询课程与目标录播、下载课件，再取所需讲次字幕和切片。
用户仅请求本地笔记时不访问网络；只有下载需求时不扩展为整门课写作任务。

## 交给笔记工作流的材料

| 素材 | 应保留的信息 |
| --- | --- |
| PDF/PPT/PPTX 原课件 | 文件名、页码/幻灯片序号、图表、公式和例题 |
| SRT 字幕 | 时间戳、原始句子，疑似识别错误与补充解释分开 |
| PPT 切片与清单 | 图片路径、切片顺序和时间定位 |
| 课程/视频元数据 | 课程名、课程 ID、视频 ID、讲次或日期 |

先建立课件页与字幕段落的对应关系，再读
[讲义写作规则](lecture-notes.md) 写作。PDF/PPT 文本提取是阅读辅助，
不是最终讲义；公式、图表或扫描页必须回到原图核对。
如使用本地来源记录，写入文件路径、页码、时间段与缺口，不记录带签名下载 URL。

讲义插图应使用实际存在的本地图片，能复制到课程目录时保留清晰的来源与时间定位。
若引用 `data/` 中的图片，要说明它们是未纳入 Git 的本地依赖；不能只提交 Markdown
就声称讲义已可完整移植。已有笔记和图片不因整理而批量搬动。

## 认证与边界

Canvas API token 用于普通 Canvas API。视频客户端通过
`external_tools/sessionless_launch` 获得 LTI 启动 URL，再用 HTTP 请求完成表单和
跳转换取新版视频 JWT，再从 `/lms/launch-context` 取得教学班 ID，
用 `jwt-token` 请求头访问新资源接口，不需要本地浏览器 profile。
列表中的录播 ID、教学班 ID、流条目的 `courId` 不可混用；字幕/切片使用录播 ID。

- 401/403 时检查 token 有效期、课程权限和网络；不改成无限重试或自动模拟 jAccount 登录。
- Canvas API 可访问不代表视频平台或校内媒体地址一定可访问；分别核对失败的阶段。
- `video list` 分页返回录播，并提供 `playable` 可点击状态；灰色的具体原因不能仅凭状态猜测。
- HTTP 200 且 AI `data:null` 表示未提供素材；HTTP 错误、业务失败和未知结构仍报错。
  空 AI 字幕/切片不等于没有回放或附件，回放的 PPT 通道也是视频，不是静态切片。
  可根据已有课件写作，并标明素材范围。
- 上游暂不下载视频本体，也不做本地语音转写；需要这些能力时另行确定工具和任务范围。
- token、临时 LTI URL 和带签名下载链接仅留在受忽略的本地配置/素材中。
  特别是 `files --json` 等原始元数据可能包含完整文件 URL，不直接贴到聊天或提交到 Git。

## 本地验证

以下检查不需要真实凭证，只验证本地安装、参数和上游 mock 行为：

```bash
.venv/bin/sjtu-canvas --help
.venv/bin/sjtu-canvas video subtitle --help
.venv/bin/sjtu-canvas video ppt-export --help
.venv/bin/python -m pytest third_party/sjtu-canvas-assistant/tests
```

运行测试需要在项目虚拟环境中安装上游的 `dev` extra。
真实连通性验证需要有效 token 和对应课程权限，应检查课件、录播列表、播放元数据，
逐项记录字幕与切片的 `available/unavailable`。有素材时检查非空 SRT/可打开图片，
没有素材时检查不生成空文件且保留原文件。`--help`、源码检查和 mock 测试不能替代该验证。
