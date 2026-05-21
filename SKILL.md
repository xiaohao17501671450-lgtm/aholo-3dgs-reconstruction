---
name: aholo-3dgs-reconstruction
description: "仅用于 Aholo OpenAPI v1 的 3D 任务（reconstruction/generation）：素材上传后创建任务（WorldAsyncOperation / worldId），随后 status/poll 与结果（PLY/SPZ/SOG）。默认单笔诉求每轮一次 create；用户明确多视频「各建」时可按视频数多次 create。不适用于 2D 出图。"
---

# Aholo 3D 重建技能（OpenAPI v1）

> **Aholo 国内开放平台**（`api.aholo3d.cn`）3D 任务技能，非 2D 文生图。默认由 **Agent 代跑** `aholo_reconstruct.py`，用户仅需配置 `AHOLO_API_KEY`。

## 1. 何时使用

**适用：** 3D 重建（reconstruction）、3D 生成（generation）、查询/轮询 `worldId`、获取 PLY/SPZ/SOG。

**不适用：** 2D 效果图/文生图；未要求 3D 结果（worldId、模型文件、在线预览）。

**歧义必先澄清（2D vs 3D）：** 例如「按参考图生成房间」→ 问用户要 2D 单图还是 3D 任务（worldId 可轮询）；仅选 3D 时进入本 skill。

## 2. 前提与接口要点

| 项 | 说明 |
|----|------|
| 环境变量 | `AHOLO_API_KEY`（[api-keys](https://labs.aholo3d.cn/api-keys)） |
| 鉴权 | `Authorization: <API Key>`，无 `Bearer` |
| 创建请求头 | `x-source: skills` → 平台 `OPEN_API_SKILL` |
| 网关 | `https://api.aholo3d.cn`；路径 `/world/v1/*` |
| 查看链接 | `https://studio.aholo3d.cn/3dgs-model/{worldId}` |
| 脚本动作 | `create` / `create-reconstruction` / `create-generation` / `status` / `poll` |
| OUS 上传 | token 响应为 `c/m/d`；OpenAPI 成功为业务对象直出 |
| 创建成功 | `WorldAsyncOperation` 仅含 `worldId` |
| 积分不足 | bizCode `11003` 或 `insufficient credits` → 告知积分不足，引导 [labs.aholo3d.cn/quickstart](https://labs.aholo3d.cn/quickstart)，禁止编造链接；**禁止**对**同一视频/同一笔意图**重复 `create*` 重试 |

**缺 `AHOLO_API_KEY`：** 说明如何配置环境变量，请用户回复「继续」后由 Agent 代跑；**禁止**把「用户自己执行 python 命令」作为主路径。

**TLS：** 脚本默认跳过证书校验；需校验时设 `AHOLO_FORCE_SSL_VERIFY=1`。

## 3. Agent 硬约束（必须严格执行）

| # | 约束 |
|---|------|
| 0 | 2D 出图诉求 → **禁止**本 skill 的 create/status/poll |
| 7 | 未明确要 3D → 先做 2D/3D 二选一（与 §1 一致），再创建 |
| 1–3 | **仅 reconstruction**：未确认 `scene`（`model`/`space`）与 `taskQuality`（`low`/`normal`/`high`）→ **禁止** create；**禁止**默认 `model`/`high` 等替用户下单 |
| 4 | **generation**：**不问** `scene`/`taskQuality`；`prompt` 与 `imagePaths` 就绪后 create-generation |
| 8 | 图片目录 → **必须** `imageDir` 上传全部图片；**禁止**只传部分（如前 20 张） |
| 11 | **2–3 个视频**：创建前**必须**问用户（**禁止**代选）：**A** 合并为一个 3DGS（一次 create，`videoPaths` 含全部）；**B** 每个视频各一个 3DGS（见 #9 多笔例外）。仅 1 个视频不必问 |
| 9 | **创建 POST（高成本）** — 默认：用户**单笔** 3D 诉求（一个 worldId 目标）在本轮对话中 **最多 1 次** create；成功/失败/超时/未解析 `worldId` 均**禁止**在同意图下再次 create，除非用户**明确重新下单**。**上传前**失败（未发出 POST）→ 修正后可发**该笔**的首次 create。**疑似已扣费无 worldId** → 引导任务列表或 status/list，禁止用再 create 排错。**多视频选 B**：用户已明确「各建」时，按视频数量依次 create（每视频 `videoPaths` 仅 1 个，共 N 次，N≤3）；开始前告知 N 笔任务/扣费；**禁止**对**同一视频**重复 create；单次失败不得重试该视频，可对**尚未创建**的视频继续。误防重复：仅在对**已完成的那一笔**误触时再传 `forbidCreate: true` |
| 10 | `projectName`：用户未明确要求 → **禁止**传入；禁止用目录名/时间戳编造 |
| 5–6 | 每个 `worldId` 创建后 → **必须先问**是否等待；愿等则**同步** `poll` 至终态；不愿等则返回 worldId + studio 链接。**禁止**后台 poll 并承诺「完成后通知」；**禁止**未问就 poll |

### `taskQuality` 展示名（传参仍用枚举）

| 枚举 | 展示 |
|------|------|
| `low` | 极速预览 |
| `normal` | 标准 |
| `high` | 专业（推荐） |

## 4. 标准执行流程

1. 判定适用性 + 2D/3D（§1）。
2. 确认 `reconstruction` 或 `generation`。
3. **重建**：`AskQuestion` 优先，一次确认 `scene` × `taskQuality`（6 组合，见 §5）；自由文本先映射再确认。
4. **多视频（2–3）**：确认 A 合并 / B 各建（§5 模板）。
5. `token` → 上传素材 → **create**（遵守 §3 #9、#11）。
6. 记录 `worldId`；**询问是否等待**（§5 模板）。
7. 等待 → 同步 `poll`（建议 `intervalSeconds=60`，`timeoutSeconds=14400`）；不等 → 仅返回链接，后续可 status/poll。
8. 同轮默认不再 create，除非用户**新下单**或多视频 B 中**下一视频**（§3 #9）。

## 5. 用户交互模板

**重建 — 场景与质量（6 选一或分题）：**

| # | scene | taskQuality |
|---|-------|-------------|
| 1 | model | low |
| 2 | model | normal |
| 3 | model | high |
| 4 | space | low |
| 5 | space | normal |
| 6 | space | high |

**多视频（2–3 个）：**

```text
你提供了 N 个视频，请确认（不要替我选）：
A) 合并为一个 3DGS — 一个 worldId
B) 每个视频单独一个 3DGS — N 个 worldId（将创建 N 笔任务，按视频逐个处理）
```

**创建成功后：**

```text
任务已创建，worldId: {worldId}
完成后查看: https://studio.aholo3d.cn/3dgs-model/{worldId}
（完成前链接可能不可用）

是否等待任务完成？
- 等待 / yes — 本会话内同步 poll 直到结束
- 不等 / no — 稍后 poll 或自行打开链接
```

## 6. 任务规则与脚本参数

### reconstruction

- `videoPaths`（1–3）**或** `imagePaths` / `imageDir`（≥20 张），二选一
- 必填：`scene`、`taskQuality`
- `imageDir` 扫描 jpg/jpeg/png/bmp/webp/gif

### generation

- 最多 1 张图；`prompt` 与图不能都空；无 `videoPaths`；无 `scene`/`taskQuality`

### 常用参数

| 参数 | 说明 |
|------|------|
| `action` / `workflow` | 见 §2 |
| `imageDir` | 目录重建首选 |
| `videoPaths` | 1–3；B 模式每次仅 1 个 |
| `worldId` | status / poll |
| `forbidCreate` | 防止**同一笔**误重复 create；多视频 B 的**下一视频**不要误开 |

### 调用示例（Agent 代跑，`python -u`）

```bash
# 重建（图片目录）
python -u .cursor/skills/aholo-3dgs-reconstruction/aholo_reconstruct.py '{"action":"create","workflow":"reconstruction","imageDir":"D:/images","scene":"space","taskQuality":"high"}'

# 生成
python -u .cursor/skills/aholo-3dgs-reconstruction/aholo_reconstruct.py '{"action":"create-generation","imagePaths":["D:/seed.jpg"],"prompt":"modern minimal interior"}'

# 轮询
python -u .cursor/skills/aholo-3dgs-reconstruction/aholo_reconstruct.py '{"action":"poll","worldId":"xxx","intervalSeconds":60,"timeoutSeconds":14400}'
```

## 7. 附录

**OpenAPI 路径：** `GET /world/v1/asset/token` · `POST /world/v1/reconstructions` · `POST /world/v1/generations` · `GET /world/v1/{worldId}`（`list` 脚本未封装）

**终态：** `SUCCEEDED` · `FAILED` · `CANCELED` · `TIMEOUT` · `REJECTED`
