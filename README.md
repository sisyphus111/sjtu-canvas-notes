# SJTU Canvas Notes

面向上海交大的 AI Agent Skill：获取 Canvas 课件与课堂字幕，生成有出处、可独立学习的逐讲笔记。
核心写作规则来自本仓库积累的 [讲义提示词](references/lecture-notes.md)：
按概念依赖组织内容，解释术语与机制，保留公式、例题和课堂推理，并检查材料覆盖。

资源获取使用 [sjtu-canvas-assistant 适配 fork](https://github.com/sisyphus111/sjtu-canvas-assistant)
（[原项目](https://github.com/Aspirin0000/sjtu-canvas-assistant)），
以 Git submodule 固定在 `third_party/sjtu-canvas-assistant/`。
它通过 Canvas API token 和 sessionless LTI 获取资料。仓库本身是一个 skill，
资源工具的代码、依赖和 CLI 均由 submodule 提供。

## 使用 Skill

让支持本地文件和命令执行的 agent 读取根目录 [SKILL.md](SKILL.md)，例如：

> 使用本仓库的 sjtu-canvas-notes skill，用现有课件和课堂字幕整理操作系统第 3 讲。
> 保留原课件图、公式和课堂例子，写成没有听过课也能学习的中文讲义。

获取资源并整理整门课时，可以直接说：

> 使用本仓库的 sjtu-canvas-notes skill，查找“课程名称”（某学期），获取课件、课堂字幕
> 和 PPT 切片，为整门课生成逐讲中文学习笔记，并整理课程导航和知识总览。
> 凭证已在本地 .env。不要覆盖已有笔记；材料缺失处请注明。

只处理一部分时，把范围改为“第 1–3 讲”或指定课件/主题。Agent 负责查询课程 ID、
匹配讲次、下载、检查素材、逐讲写作和检查覆盖，不需要用户手动串联下方命令。

已有本地资料时可以直接写作，无须配置 Canvas。安装到其他客户端时应保留完整仓库
和已初始化的 submodule，不要只复制 `SKILL.md`，也不要只下载不含 submodule 的源码 ZIP。
本仓库不自动修改全局客户端设置。

向客户端安装时，将完整仓库根目录注册或链接为 `sjtu-canvas-notes` skill；
客户端入口应指向本仓库的 `SKILL.md`，随附的 `references/` 和 `third_party/` 必须可访问。

## 准备资源获取工具

克隆完整 skill（包括固定版本的资源工具）：

```bash
git clone --recurse-submodules https://github.com/sisyphus111/sjtu-canvas-notes.git
cd sjtu-canvas-notes
```

对于已有 checkout，在仓库根目录执行：

```bash
git submodule update --init --recursive
```

先检查本机 Python 和当前虚拟环境。上游工具支持 Python 3.9+；没有合适的项目环境时，
在本仓库创建 `.venv`，只安装上游资源工具：

```bash
python3 --version
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -e ./third_party/sjtu-canvas-assistant
.venv/bin/sjtu-canvas --help
```

阅读 PDF/PPT 时，agent 使用宿主提供的文档工具；只有确实需要额外文本提取器时，
才检查并在项目虚拟环境中补装对应依赖。

## 获取 Token（首次使用）

1. 用 jAccount 登录 [SJTU Canvas](https://oc.sjtu.edu.cn)，打开
   [账户设置](https://oc.sjtu.edu.cn/profile/settings)；也可以从“账户 → 设置”进入。
2. 找到“已批准的集成 / Approved Integrations”（中文也可能显示为
   “允许融入使用的外部软件”），选择“创建新访问许可证 / New Access Token”。
3. 用途填写便于识别的名称，例如 `sjtu-canvas-notes`，按页面要求设置到期时间，生成
   token 并复制保存。如果页面有待激活状态，按页面提示激活后再验证。
4. 只在本地配置 token，不填写 jAccount 密码。没有创建按钮或按钮不可用时，可能是
   账号权限或学校设置限制；需要向 Canvas 管理方确认，不能保证所有账号都能创建。

操作位置参见 [Canvas 官方用户说明](https://community.instructure.com/en/kb/articles/662901-how-do-i-manage-api-access-tokens-in-my-user-account)
和 [上游配置说明](third_party/sjtu-canvas-assistant/README.md#创建-canvas-access-token)。
按钮名称和到期要求以交大实际页面为准。

在仓库根目录复制配置模板，已有 `.env` 时只编辑原文件：

```bash
cp -n .env.example .env
chmod 600 .env
```

用本地编辑器把 `.env` 中的 `CANVAS_TOKEN=` 填成生成的 token，其他配置保留默认值。
`.env` 已被 Git 忽略。token 若已发到聊天或其他公开位置，应在 Canvas 撤销并换新，
只在本地更新。更新时无需改动 skill 或上游代码。

从本仓库根目录运行；上游配置加载器会向父目录查找 `.env`，进程环境变量优先。
不要在 submodule 内另放一份 `.env`，避免遮蔽根目录配置。不要将 token 放进聊天、
命令参数或 Git。先验证认证：

```bash
.venv/bin/sjtu-canvas whoami
```

认证通过后，将课程名、学期与目标范围交给 skill，后续过程由 agent 完成。

## 从课程到学习笔记

| 阶段 | Agent 执行的工作 | 可检查的结果 |
| --- | --- | --- |
| 定位课程 | 查询课程名、学期、ID，匹配目标范围 | 已确定的课程与讲次 |
| 获取资源 | 下载课件、目标讲次字幕及 PPT 切片 | 原文件、可用字幕/切片，或明确的不可用状态与缺口 |
| 对齐素材 | 建立课件页与字幕时间段的对应 | 课程 `sources.md` 中的来源与缺口 |
| 写逐讲笔记 | 按原 prompt 解释概念、机制、公式、例题并检查覆盖 | `lectures/` 中可独立阅读的讲义 |
| 汇总课程 | 检查全部范围，整理知识依赖和复习导航 | `index.md`、`summary.md`、剩余缺口 |

整门课任务先验证一讲素材，再继续完成指定范围；进度写入来源记录，方便续做。
没有字幕时明确标注基于课件；下载成功不等于笔记已经完成。
新版录播列表会显示可点击状态；灰色节次的底层原因需另行核实。回放中的 PPT 视频
通道、AI 静态切片与 Canvas 附件分别处理。AI 素材不可用时不生成空 SRT/PDF，
不把这个状态解释成整门课没有资料。

下列命令用于手工验证和理解资源层。课程 ID 和视频 ID 由查询结果取得：

```bash
.venv/bin/sjtu-canvas courses --json
.venv/bin/sjtu-canvas plan <course-id>
.venv/bin/sjtu-canvas video list <course-id> --json
```

将占位符替换为查询所得 ID，再下载课程课件和目标讲次的字幕、图片：

```bash
.venv/bin/sjtu-canvas download <course-id> --output data/materials
mkdir -p data/recordings/<course-id>/<video-id>
.venv/bin/sjtu-canvas video subtitle <course-id> <video-id> --output data/recordings/<course-id>/<video-id>/transcript.srt
.venv/bin/sjtu-canvas video ppt-export <course-id> <video-id> --output data/recordings/<course-id>/<video-id>/ppt --json > data/recordings/<course-id>/<video-id>/ppt-export.json
```

`download` 按课程下载文件；先用 `plan` 检查范围和大小。
`ppt-export` 保存切片图片和默认合成的 `ppt.pdf`，上面的重定向另存 JSON 导出记录。
记录课件页码与字幕时间戳后，按 skill 中的提示词写作。
更多能力、输出约定和故障边界见 [资源获取说明](references/resources.md)。

## 目录与已有材料

| 路径 | 用途 |
| --- | --- |
| `SKILL.md` | Agent 工作入口 |
| `references/lecture-notes.md` | 原 prompt 提炼的讲义写作规则和正反例 |
| `references/resources.md` | 上游 CLI、素材交接与限制 |
| `third_party/sjtu-canvas-assistant/` | MIT 资源工具的新版课堂视频适配 fork，以 submodule 固定 |
| `study_materials/` | 运行 skill 后生成的课程讲义，不进入 Git |
| `data/`、`.env`、`.venv/` | 本地素材、凭证与依赖，不进入 Git |

原始素材只作写作输入，最终讲义保存在 `study_materials/<真实课程名>/lectures/`。
课程目录中的 `sources.md` 记录来源页码、字幕时间段、覆盖情况和材料缺口。
本仓库发布 skill 和资源工具，不附带课程课件、字幕、截图或课程笔记。
运行时已有的课程笔记与本地素材保留原位置。

## 验证

安装上游测试依赖后，可在无 token 的环境中检查本地 CLI 和上游 mock 流程：

```bash
.venv/bin/python -m pip install -e './third_party/sjtu-canvas-assistant[dev]'
.venv/bin/sjtu-canvas --help
.venv/bin/python -m pytest third_party/sjtu-canvas-assistant/tests
```

真实平台验证需配置有效 token，检查课件、录播列表、播放元数据及目标讲次的 AI 素材状态。
字幕或切片存在时验证实际内容；平台未提供时记录 `unavailable`，不要求生成空文件。
本地帮助和 mock 测试通过不能证明真实平台链路成功。

## 来源与许可证

本仓库的 skill 指令、讲义写作规则和使用文档以 [MIT License](LICENSE) 发布。
上游 submodule 的 MIT 许可证和版权声明保持原样，详见
[THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。分发包含上游代码的副本时，
同时保留对应许可证。软件许可证不授予 Canvas 课程课件、字幕、图片等教学材料的
使用或传播权限；用户获取的材料和生成的课程笔记应按各自来源和权利范围处理。

submodule 版本由父仓库的 gitlink 固定。普通初始化使用 `git submodule update --init --recursive`；
只有有意升级并检查上游变更时，才切换 submodule revision、重新安装依赖并验证。
