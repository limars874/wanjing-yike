# 万镜一刻视频与图片生成调用说明

## 1. 基础配置

服务地址由部署方提供，请替换为实际的 New API 地址：

```text
https://<YOUR_NEW_API_BASE_URL>
```

调用方使用 **New API 下游 Token**，不是万镜一刻供应商 API Key：

```http
Authorization: Bearer <NEW_API_KEY>
Content-Type: application/json
```

示例环境变量：

```bash
export BASE_URL='https://<YOUR_NEW_API_BASE_URL>'
export NEW_API_KEY='<YOUR_NEW_API_KEY>'
```

生产环境请使用 HTTPS；如果部署方提供的是 HTTP 地址，Token 和请求内容未加密，只适合短期测试。

## 2. 视频生成

### 2.1 支持模型

```text
Wonder-Ultra
Wonder-Pro
Wonder-Standard
wan3.0-video
happyhorse-1.0
happyhorse-1.1
```

当前首版只支持**文生视频**，输出异步任务，通常为 MP4。暂不支持图片输入、图生视频、首尾帧和参考视频。

### 2.2 提交任务

接口：

```http
POST /v1/videos
```

示例：

```bash
curl --request POST "$BASE_URL/v1/videos" \
  --header "Authorization: Bearer $NEW_API_KEY" \
  --header 'Content-Type: application/json' \
  --data '{
    "model": "happyhorse-1.0",
    "prompt": "一只橘猫在阳光下的草地上散步，写实风格",
    "seconds": 3,
    "size": "1280x720",
    "n": 1
  }'
```

主要参数：

| 参数 | 必填 | 说明 |
|---|---:|---|
| `model` | 是 | 视频模型 ID，必须使用上面的名称。 |
| `prompt` | 是 | 文生视频提示词。 |
| `seconds` | 否 | 视频时长；`happyhorse-1.0` 已验证支持 3～15 秒。 |
| `size` | 否 | 如 `1280x720`（720P）、`1920x1080`（1080P）。 |
| `resolution` | 否 | 也可直接填写 `720P` 或 `1080P`。 |
| `n` | 否 | 输出数量，默认 1；建议先使用 1。 |
| `aspect_ratio` | 否 | 如 `16:9`、`9:16`、`1:1`。 |
| `scene` | 否 | 场景类型，默认 `general`。 |

提交成功返回本地任务 ID：

```json
{
  "id": "task_xxx",
  "object": "video",
  "status": "queued",
  "model": "happyhorse-1.0"
}
```

### 2.3 查询任务

```http
GET /v1/videos/:task_id
```

```bash
curl "$BASE_URL/v1/videos/<TASK_ID>" \
  --header "Authorization: Bearer $NEW_API_KEY"
```

状态通常为：

```text
queued → in_progress → completed
```

失败时状态为 `failed`，同时查看返回的 `error.message`。

### 2.4 下载视频

任务完成后：

```bash
curl --location \
  "$BASE_URL/v1/videos/<TASK_ID>/content" \
  --header "Authorization: Bearer $NEW_API_KEY" \
  --output result.mp4
```

## 3. 图片生成

### 3.1 支持模型

```text
qwen-image-3.0
qwen-image-2.0
Wonder-Image-2
Wonder-Image-Pro
```

当前首版只支持**文生图片**，输出异步任务，通常为 PNG。暂不支持图片输入、图生图和 `ImportMedia`。

图片接口使用 Plugin 声明的原生路由，不是标准 OpenAI 的 `/v1/images`：

```http
POST /wanjing/v1/images
```

### 3.2 提交任务

示例：

```bash
curl --request POST "$BASE_URL/wanjing/v1/images" \
  --header "Authorization: Bearer $NEW_API_KEY" \
  --header 'Content-Type: application/json' \
  --data '{
    "model": "qwen-image-3.0",
    "prompt": "一只戴着宇航员头盔的橘猫，漂浮在星云中，电影级光影",
    "resolution": "1K",
    "n": 1
  }'
```

主要参数：

| 参数 | 必填 | 说明 |
|---|---:|---|
| `model` | 是 | 图片模型 ID，必须使用上面的名称。 |
| `prompt` | 是 | 文生图片提示词。 |
| `resolution` | 否 | `1K`、`2K` 或 `4K`，默认 `1K`。 |
| `size` | 否 | 也可使用尺寸表达，例如 `1024x1024`。 |
| `n` | 否 | 输出数量，建议使用 1；供应商文档范围为 1～4。 |
| `aspect_ratio` | 否 | 如 `16:9`、`9:16`、`1:1`。 |
| `scene` | 否 | 场景类型，默认 `general`。 |

提交成功返回本地任务 ID：

```json
{
  "id": "task_xxx",
  "object": "image",
  "status": "queued",
  "model": "qwen-image-3.0"
}
```

### 3.3 查询任务

```http
GET /wanjing/v1/images/:task_id
```

```bash
curl "$BASE_URL/wanjing/v1/images/<TASK_ID>" \
  --header "Authorization: Bearer $NEW_API_KEY"
```

状态通常为：

```text
queued/running → completed
```

### 3.4 获取图片

图片结果建议使用通用 artifact 接口：

```bash
curl "$BASE_URL/v1/tasks/<TASK_ID>/artifacts" \
  --header "Authorization: Bearer $NEW_API_KEY"
```

返回 `artifacts[0].content_url` 后，使用其中的路径和 `access` 参数下载：

```bash
CONTENT_URL=$(curl --silent \
  "$BASE_URL/v1/tasks/<TASK_ID>/artifacts" \
  --header "Authorization: Bearer $NEW_API_KEY" | jq -r '.artifacts[0].content_url')

CONTENT_PATH=$(printf '%s' "$CONTENT_URL" | sed -E 's#^https?://[^/]+##')

curl --location "$BASE_URL$CONTENT_PATH" \
  --output result.png
```

`content_url` 中的 `access` 是短期访问凭证，不要公开或长期保存。

当前 Plugin `0.1.3` 已修复原生图片查询响应类型；完成任务后返回 `object=image` 和 `data[].url`。图片下载仍建议使用 artifact 接口，以获得统一的内容代理和短期访问 URL。

## 4. 常见错误

| 错误 | 原因 |
|---|---|
| `401 Unauthorized` | New API Token 无效、缺少 `Bearer` 或 Token 已禁用。 |
| `model_price_error` | 后台没有为该模型配置完整价格。 |
| `prompt is required` | 未填写非空 `prompt`。 |
| `media input is not enabled` | 当前 Plugin 首版不接受图片、视频或 `ImportMedia` 输入。 |
| `status=failed` | 查看响应中的 `error.message`，通常是模型权限、参数或供应商任务失败。 |

## 5. 调用流程摘要

```text
New API Token
    ↓ Authorization: Bearer <NEW_API_KEY>
POST /v1/videos 或 POST /wanjing/v1/images
    ↓
获得 task_id
    ↓
轮询对应查询接口
    ↓
completed
    ↓
视频：GET /v1/videos/:task_id/content
图片：GET /v1/tasks/:task_id/artifacts，再访问 image artifact content_url
```
