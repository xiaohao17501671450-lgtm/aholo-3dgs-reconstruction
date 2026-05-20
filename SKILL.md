---
name: aholo-3dgs-reconstruction
description: "仅用于 Aholo OpenAPI v1 的 3D 任务（reconstruction/generation）：素材上传后创建任务（成功响应 WorldAsyncOperation / worldId），随后 status/poll 查询与结果获取（含 PLY/SPZ/SOG）；创建接口高成本，每轮用户任务仅允许一次 POST 创建；不适用于 2D 出图或未明确 3D 结果的请求。"
---

# Aholo 3D 重建技能（OpenAPI v1）

> 核心定位：**Aholo 国内开放平台**（`api.aholo3d.cn`）的 3D 任务调用技能，不是 2D 文生图技能。

## 适用 / 不适用（先判定）

### 适用（应使用本 skill）

- 用户要“重建 3D 模型 / 3D 空间”（reconstruction）
- 用户要“生成 3D 空间 world / worldId / 3D 可预览链接 / PLY-SPZ-SOG 文件”（generation）
- 用户提到“查询 worldId 状态、继续轮询任务进度”

### 不适用（不要使用本 skill）

- 用户只要“一张图片效果图/概念图/渲染图”（2D 出图）
- 用户未要求 3D 结果（worldId、模型文件、在线 3D 预览）
- 用户要的是普通图像编辑或文生图（应走图片生成能力）

### 歧义请求处理（必须先澄清）

当用户说“帮我生成一个房间，参考某图风格”这类请求时，先问：

```text
你要的是：
1) 2D 房间效果图（单张图片）
2) 3D 房间任务（可返回 worldId 并可轮询）
```

仅当用户明确选择 2) 时，才进入本 skill 流程。

将图片或视频上传并创建 3D 任务，支持：

- reconstruction（重建）
- generation / spatial gen（生成）

任务完成后可返回模型文件下载地址（PLY / SPZ / SOG）。

## 前提条件（必须先确认）

只需要以下环境变量：

- `AHOLO_API_KEY` —— Aholo 开放平台分配的 API Key

如果用户没有合适的 API Key，请提醒用户前往 https://labs.aholo3d.cn/api-keys 创建。

鉴权方式：开放平台接口使用请求头 `Authorization: <API Key>`（无需 `Bearer` 前缀）。

### 未配置 `AHOLO_API_KEY` 时（Agent 行为）

- 明确告知用户：需在本机/当前运行环境配置环境变量 `AHOLO_API_KEY`（具体方式因操作系统与 Cursor 运行环境而异，可简要说明「在本终端或系统环境变量中设置」）。
- **禁止**让用户去「自己执行后面的 `python ... aholo_reconstruct.py` 命令」才能完成流程；应提示用户在配置好 API Key 后，**回到对话回复「继续」**或再次描述同一 3D 任务，由 Agent 继续代为调用脚本。
- 用户若自愿本地调试脚本，可自行执行，但技能默认流程以 **Agent 代跑** 为主。

## 证书校验策略（已默认自动绕过）

- 脚本默认关闭 SSL 证书校验，以避免公司网络/本机自签证书导致 `CERTIFICATE_VERIFY_FAILED`。
- 如需强制开启证书校验，可设置：`AHOLO_FORCE_SSL_VERIFY=1`。
- 兼容保留：`AHOLO_INSECURE_SKIP_VERIFY`。除显式设置为 `0/false/no/off` 外，默认仍按“自动绕过”执行。

## 当前服务地址

- API 网关：`https://api.aholo3d.cn`
- 上传域名：通过 `GET /world/v1/asset/token` 返回的 `globalDomain` 动态获取（OUS 直传仍走该域名，响应格式为 OUS V2 的 `c`/`m`/`d`）

## 开放平台接口（与 `openApiUrl` 一致）

| 方法 | 路径 | 成功响应体 |
|------|------|------------|
| GET | `/world/v1/asset/token` | 凭证对象：`ousToken`、`globalDomain`、`blockSize` |
| POST | `/world/v1/reconstructions` | `WorldAsyncOperation`：`{"worldId":"<加密 id>"}`（OpenAPI 名 `WorldAsyncOperation`，LRO 受理形态，与详情路径 `{worldId}` 一致） |
| POST | `/world/v1/generations` | 同上，`WorldAsyncOperation` |
| GET | `/world/v1/{worldId}` | 世界详情对象（含 `status`、`assets` 等） |
| POST | `/world/v1/list` | 分页列表对象（脚本当前未封装） |

## 响应约定（Aholo 五接口）

- **成功**：HTTP 200，响应体为业务对象直出（**非** OUS 的 `c`/`m`/`d` 封装）。
  - 创建重建/生成：**`application/json`** 的 `WorldAsyncOperation`，仅含字段 **`worldId`**（加密字符串）。网关经代理后应与 `OpenApiWorldController` 返回形态一致；若遇极旧环境曾返回裸文本 `worldId`，脚本仍做兼容解析。
- **失败**：HTTP 4xx/5xx，响应体为 `ApiError`：
  - `code`：与 HTTP 状态码一致（如未登录为 `401`）
  - `message`：错误说明
  - `status`：如 `UNAUTHENTICATED`、`INVALID_ARGUMENT`
  - `details.metaData.bizCode`：原业务码（如 `10004`）

## 支持动作（action）

| 动作 | 用途 |
|------|------|
| `create` | 统一入口，需配合 `workflow` 指定 `reconstruction` 或 `generation` |
| `create-reconstruction` | 只做重建（等价于 `create + workflow=reconstruction`） |
| `create-generation` | 只做生成（等价于 `create + workflow=generation`） |
| `poll` | 按 `worldId` 轮询直到终态（推荐） |
| `status` | 按 `worldId` 查询一次状态 |

## Agent 执行硬约束（必须严格执行）

以下规则优先级最高：

0. 若用户诉求是 2D 出图（而非 3D 任务），**禁止调用本 skill 的 create/status/poll**。
1. 若用户请求“重建”但未明确 `scene` 与 `taskQuality`，**禁止调用 create**。
2. 必须先向用户确认并拿到明确选择：
   - `scene`: `model` 或 `space`
   - `taskQuality`: `low` / `normal` / `high`
3. **禁止擅自使用默认值**（例如默认 `model` 或默认 `high`）替用户下单。
4. 仅当用户明确回复后，才允许调用 `create` / `create-reconstruction` / `create-generation`（且须遵守第 9 条「单次创建」）。
5. 拿到 `worldId` 后，**必须先询问用户是否愿意等待轮询**，根据用户选择执行：
   - **愿意等待**：同步执行 `poll`，保持会话直到返回结果
   - **不愿等待**：仅返回 `worldId` 和查看链接，告知用户如何自行查看
6. **禁止**将同步 `poll` 放到后台执行后告诉用户"完成后会通知你"——这是误导性描述，会话结束后无法通知用户。
7. 若用户未明确要 3D，必须先做“2D/3D 二选一澄清”，不得直接创建任务。
8. **图片目录处理约束**：当用户指定了包含多张图片的目录时（如“将 xxx 目录下的图片进行重建”），**必须使用 `imageDir` 参数**而不是手动列出部分图片。`imageDir` 会自动扫描并上传目录下的所有图片，确保使用完整素材进行重建。**禁止只上传部分图片（如前20张）就创建任务。
9. **创建接口单次调用（高成本，必须遵守）**：
   - 针对用户**本轮**明确提出的「这一个」3D 下单诉求（一次任务闭环），**最多只向** `POST /world/v1/reconstructions` **或** `POST /world/v1/generations` **发起 1 次 HTTP 请求**。
   - **无论**创建结果是成功、4xx/5xx、超时、网络错误、本地解析失败、未拿到 `worldId`，**禁止**在未获得用户**明确重新下单**（表达为「重新发起一笔新任务」「再建一个新项目」等**新意图**）的情况下，**再次调用** `create` / `create-reconstruction` / `create-generation`。
   - **唯一例外**：若因**上传前**失败（缺参、鉴权失败、素材未就绪等）**尚未发出**创建 POST 的，允许在用户修正前置条件后**首次**调用创建（仍须保证**整个闭环内创建 POST 仅一次**）。
   - 若怀疑服务端已受理扣费但本地未解析出 `worldId`：**引导用户在开放平台任务/世界列表核对**，或使用 `status`/`list` 等只读手段排查；**禁止**用「再创建一次」当排错手段。
   - Agent 自检：若同一会话中**已经**对创建接口发起过请求，后续若误触发 `create*`，应对脚本传入 **`forbidCreate: true`** 拦截，或**根本不发起** `create*`（仅 `status`/`poll`）。用户若确要第二个任务，须**明确新建一单**，再开始新的单次创建闭环。
10. **`projectName`（项目名）**：用户**未明确**要求命名时，**禁止**在脚本 JSON 中传入 `projectName`（由服务端使用默认展示名）。**禁止**根据目录名、文件名、时间戳等自行编造项目名。**仅当**用户明确给出名称（如「项目名叫客厅重建」）时，才传入 `projectName`。

## 必选参数交互（可直接选择）

为降低用户输入成本，确认 `scene` 与 `taskQuality` 时应遵循：

1. 在支持结构化提问工具时，优先调用 `AskQuestion` 让用户点选；不要先让用户自由输入。
2. 两个参数可一次性提问，避免来回追问。
3. 用户未选择完成前，禁止调用 `create` / `create-reconstruction`。
4. 若用户给了自由文本（如“高质量空间”），先归一化映射，再回显确认一次。

### 推荐选项

- `scene`
  - `model`：单体模型（物体）
  - `space`：空间场景（房间/展厅）
- `taskQuality`
  - `low`：低（更快）
  - `normal`：中（平衡）
  - `high`：高（推荐）

### 推荐交互模板（可直接发给用户）

**第一步：确认场景和质量**

使用 AskUserQuestion 工具时，提供全部 6 种组合选项：

| 选项 | scene | taskQuality | 说明 |
|------|-------|-------------|------|
| 1 | model | low | 单体模型，低质量（更快） |
| 2 | model | normal | 单体模型，中等质量（平衡） |
| 3 | model | high | 单体模型，高质量（推荐） |
| 4 | space | low | 空间场景，低质量（更快） |
| 5 | space | normal | 空间场景，中等质量（平衡） |
| 6 | space | high | 空间场景，高质量（推荐） |

或者直接文本提问：

```text
请先选择两个必选项（回复编号或英文值都可以）：

1) 场景 scene
   A. model（单体模型）
   B. space（空间场景）

2) 质量 taskQuality
   1. low（更快）
   2. normal（平衡）
   3. high（推荐）
```

**第二步：创建成功后询问是否等待**

```text
任务已创建，worldId: {worldId}

任务完成后查看地址: https://www.aholo3d.cn/3dgs-model/{worldId}
（注意：任务完成前该链接暂时无法查看）

重建通常需要几分钟到几十分钟，取决于质量设置和素材复杂度。

你愿意等待任务完成吗？
- 输入 "等待"或"yes"：我会保持会话直到返回结果
- 输入 "不等"或"no"：你可以稍后让我执行 poll 查询，或等待任务完成后访问上面链接在线查看
```

## 推荐执行流程（最小闭环）

1. 先确认用户目标：`reconstruction`（重建）还是 `generation`（生成）。
2. 若为重建，必须确认：
   - `scene`: `model`（模型）或 `space`（空间）
   - `taskQuality`: `low` / `normal` / `high`
3. **整个任务闭环内只调用一次** `create`（或对应专用 action）；失败后不得自动重试创建（见上文「创建接口单次调用」）。
4. 成功后记录响应中 `WorldAsyncOperation.worldId`，**必须询问用户是否愿意等待轮询完成**。
5. 根据用户选择执行：
   - **用户愿意等待**：同步执行 `poll`，保持会话直到任务完成并返回结果
   - **用户不愿等待**：仅返回 `worldId` 和查看链接，告知用户可自行查看或稍后查询
6. 后续仅调用 `poll` / `status`（或列表等只读能力），**不得**再次 `create`，除非用户明确发起**新的一笔任务**。

说明：创建接口**非幂等且成本高**，重复 POST 会产生多笔扣费/多任务；须严格遵守单次调用原则。

## 执行日志规范（默认启用）

每次执行动作都需要输出三段信息：

1. 开始做什么（例如：开始获取上传 token）
2. 结束做什么（例如：结束获取上传 token）
3. 本动作耗时多久（秒）

推荐动作粒度：

- 获取上传 token
- 上传素材（单文件/分片）
- 创建重建/创建生成
- 查询状态
- 轮询状态（每次查询）

## 轮询策略（必须明确用户意愿）

拿到 `worldId` 后，**必须先询问用户**是否愿意等待任务完成，再决定执行方式：

### 选项1：用户愿意等待（推荐）

**必须同步执行**，保持会话直到任务完成：
```bash
# 同步轮询，会话保持直到返回结果（加 -u 参数以便实时看到进度）
python -u aholo_reconstruct.py '{"action":"poll","worldId":"xxx","intervalSeconds":60,"timeoutSeconds":14400}'
```

### 选项2：用户不愿等待

仅返回信息，告知用户如何自行查看：
```text
任务已创建，worldId: xxx

任务完成后查看地址: https://www.aholo3d.cn/3dgs-model/xxx
（注意：任务完成前该链接暂时无法查看）

您可以：
1. 稍后让我执行 poll 查询状态
2. 或等待任务完成后访问上面链接在线查看
```

### 禁止行为

- **禁止**将同步轮询放到后台执行，然后告诉用户"完成后会通知你"——这是误导，因为会话结束后无法通知
- **禁止**在未询问用户的情况下自作主张放到后台轮询

### 轮询参数建议

- `intervalSeconds = 60`（每分钟检查一次）
- `timeoutSeconds = 14400`（最长等待4小时）

## 任务类型规则

### reconstruction（重建）

- 输入支持 `videoPaths` 或 `imagePaths`，二选一
- `videoPaths`：1-3 个
- `imagePaths`：不少于 20 张
- 必填：
  - `scene`: `model` / `space`
  - `taskQuality`: `low` / `normal` / `high`

### generation（spatial gen）

- 输入仅支持 `imagePaths`（最多 1 张）
- `prompt` 与 `imagePaths` 不能同时为空
- 不支持 `videoPaths`

## 参数说明

### 通用参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `action` | enum | `create` / `create-reconstruction` / `create-generation` / `status` / `poll` |
| `workflow` | enum | 仅 `create` 时生效：`reconstruction` 或 `generation` |
| `projectName` | string | 可选；对应请求体 `name`。用户未明确要求命名时**不要传** |
| `coverPath` | string | 本地封面图路径（可选） |
| `cover` | string | 已有封面 URL（可选） |
| `worldId` | string | `status` / `poll` 必填 |
| `forbidCreate` | bool | 可选，默认 `false`。为 `true` 且 `action` 为任一 `create*` 时，脚本直接拒绝执行，用于 Agent 防止同一闭环内重复下单。 |

### reconstruction 参数

| 参数 | 类型 | 规则 |
|------|------|------|
| `videoPaths` | string[] | 1-3 个视频文件路径 |
| `imagePaths` | string[] | 图片文件路径列表（至少 20 张） |
| `imageDir` | string | **推荐** 图片目录路径，自动扫描目录下所有图片（与 `imagePaths` 二选一） |
| `scene` | enum | `model` 或 `space`（必填） |
| `taskQuality` | enum | `low` / `normal` / `high`（必填） |

> **重要提示**：当用户要求重建某个目录下的所有图片时，**必须优先使用 `imageDir` 参数**，而不是手动列出部分图片。使用 `imageDir` 会自动上传目录中的所有图片（jpg/jpeg/png/bmp/webp/gif），确保重建使用完整的素材。

### generation 参数

| 参数 | 类型 | 规则 |
|------|------|------|
| `imagePaths` | string[] | 最多 1 张 |
| `prompt` | string | 可选，但与 imagePaths 不能同时为空 |

## 调用示例

**一般由 Agent 直接执行本脚本，用户无需手动复制命令**（用户仅需配置好 `AHOLO_API_KEY` 后在对话中继续即可）。以下为参考：

**注意：建议加上 `-u` 参数以便实时输出执行过程**

```bash
# 1) 重建：使用图片目录（推荐，自动上传目录下所有图片）
python -u .cursor/skills/aholo-3dgs-reconstruction/aholo_reconstruct.py '{
  "action": "create",
  "workflow": "reconstruction",
  "imageDir": "D:/images",
  "scene": "space",
  "taskQuality": "high"
}'

# 1b) 重建：手动指定图片路径列表（不推荐，仅在需要筛选特定图片时使用）
python -u .cursor/skills/aholo-3dgs-reconstruction/aholo_reconstruct.py '{
  "action": "create",
  "workflow": "reconstruction",
  "imagePaths": ["D:/images/0001.jpg", "D:/images/0002.jpg", "...至少20张..."],
  "scene": "space",
  "taskQuality": "high"
}'

# 2) 重建：视频（1-3个）
python -u .cursor/skills/aholo-3dgs-reconstruction/aholo_reconstruct.py '{
  "action": "create-reconstruction",
  "videoPaths": ["D:/videos/angle1.mp4"],
  "scene": "model",
  "taskQuality": "high"
}'

# 3) 生成（spatial gen）：最多一张图，或纯prompt
python -u .cursor/skills/aholo-3dgs-reconstruction/aholo_reconstruct.py '{
  "action": "create-generation",
  "imagePaths": ["D:/images/seed.jpg"],
  "prompt": "modern minimal interior"
}'

# 仅当用户明确要求项目名时才加 projectName，例如：
# "projectName": "客厅重建"

# 查询状态
python -u .cursor/skills/aholo-3dgs-reconstruction/aholo_reconstruct.py '{
  "action": "status",
  "worldId": "A1b2C3d4E5"
}'

# 轮询直到完成（建议使用 -u 参数以便实时看到进度）
python -u .cursor/skills/aholo-3dgs-reconstruction/aholo_reconstruct.py '{
  "action": "poll",
  "worldId": "A1b2C3d4E5",
  "intervalSeconds": 60,
  "timeoutSeconds": 14400
}'
```

## Windows PowerShell（本地调试可选）

若在 Windows 下**自行**调试，可先设置环境变量再运行脚本；**默认仍建议由 Agent 代跑**。

```powershell
$env:AHOLO_API_KEY="your_api_key"
# 再执行与上文相同的 python 调用方式
```

```powershell
# 如需强制开启证书校验（默认无需设置）
$env:AHOLO_FORCE_SSL_VERIFY="1"
```

## 终态状态值

- `SUCCEEDED`
- `FAILED`
- `CANCELED`
- `TIMEOUT`
- `REJECTED`

## 创建成功后的查看地址

`https://www.aholo3d.cn/3dgs-model/{worldId}`