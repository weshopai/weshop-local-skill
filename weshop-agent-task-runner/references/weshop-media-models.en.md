# WeShop CLI — Image / Video Model Input / Output Reference

> Reference version: **weshop-cli 0.2.12**
> Models covered: `z-image` (text-to-image), `qwen-image-edit` (text-to-image / image editing), `wan-ai` (image-to-video)
> Source of truth: `weshop-cli@0.2.12` `dist` source (`commands/*.js`, `run-helper.js`, `client.js`, `printer.js`, `private-run-controls.js`).

---

## 0. Overview

All AI model commands run through the same execution pipeline (`executeRun`) and share a unified request body:

```jsonc
// POST /openapi/agent/runs
{
  "agent": { "name": "<agentName>", "version": "<agentVersion>" },
  "input": { ... },     // inputs such as images, task name
  "params": { ... },    // generation parameters
  "safeGenerate": "off",     // private control, default "on"
}
```

On submit it returns `{ "executionId": "<id>" }`, then poll:

```
GET /openapi/agent/runs/<executionId>
```

Response (`data`):

```jsonc
{
  "agentName": "<agentName>",
  "agentVersion": "<agentVersion>",
  "executions": [
    { "executionId": "<id>", "status": "Success"|"Failed"|"...", "result": [ ... ] }
  ]
}
```

| Model | agentName | agentVersion | Capability |
|---|---|---|---|
| z-image | `z-image` | `v1.0` | Text-to-image (text only, no image input) |
| qwen-image-edit | `qwen-image-edit` | `v1.0` | Text-to-image / image editing (up to 5 reference images) |
| wan-ai | `wan-ai` | `v1.0` | Image-to-video (single input image + text, video model) |

> Private control parameters (not shown in `--help`, but can be appended to any command):
> - `safeGenerate=on|off` (or `--safe-generate`): content safety filter, default `off`.

---

## 1. `z-image` — Text-to-Image

### 1.1 Command Usage

```bash
export WESHOP_API_KEY=<key>

weshop z-image --prompt '<description>' [--aspect-ratio <ratio>] [--batch <n>] [--task-name <name>] [--no-wait]
```

CLI options:

| CLI option | Required | Type / values | Mapped field | Description |
|---|---|---|---|---|
| `--prompt <text>` | **Yes** | string | `params.textDescription` | Image description text |
| `--aspect-ratio <ratio>` | No | `1:1`(default) / `2:3` / `3:2` / `4:3` / `3:4` / `16:9` / `9:16` / `21:9` | `params.aspectRatio` | Output aspect ratio |
| `--batch <count>` | No | int 1–16 (default 1) | `params.batchCount` | Number of images to generate |
| `--task-name <name>` | No | string | `input.taskName` | Human-readable label for this run |
| `--no-wait` | No | boolean | — (sync control) | Return immediately after submission; check later with `weshop status <id>` |

### 1.2 Input (request body fields)

```jsonc
{
  "agent": { "name": "z-image", "version": "v1.0" },
  "params": {
    "textDescription": "A futuristic cityscape at sunset", // string, required
    "aspectRatio": "1:1",                                   // string, default "1:1"
    "batchCount": 1                                         // number, 1-16, optional
  },
  "input": {
    "taskName": "my-label"                                  // string, optional
  },
  "safeGenerate": "off"
}
```

`params` field details:

| Field | Type | Required | Description |
|---|---|---|---|
| `textDescription` | string | Yes | Description of the image to generate |
| `aspectRatio` | string | No | Enum as above, default `"1:1"` |
| `batchCount` | number | No | 1–16, default 1 |

`input` field details:

| Field | Type | Required | Description |
|---|---|---|---|
| `taskName` | string | No | Run label |

> Note: `z-image` does **not** support image input. The command has no `--image`, and `run-helper` will not write `input.images` / `input.originalImage`. Neither `opts.images` nor `opts.image` is passed at the CLI layer.

### 1.3 Output (result structure)

`data.executions.at(-1)` is the final execution; `result` is the array of images, each item:

```jsonc
{
  "executionId": "<id>",
  "status": "Success",   // Success | Failed
  "result": [
    { "status": "Success", "image": "https://..." },
    { "status": "Success", "image": "https://..." }
  ]
}
```

CLI prints the `[result]` block:

```
[result]
  agent: z-image v1.0
  executionId: <id>
  status: Success
  imageCount: 2
  image[0]:
    status: Success
    url: https://...
  image[1]:
    status: Success
    url: https://...
```

`result[]` item fields:

| Field | Type | Description |
|---|---|---|
| `status` | string | Status of this item (`Success`, etc.) |
| `image` | string | Generated image URL |

---

## 2. `qwen-image-edit` — Text-to-Image / Image Editing

### 2.1 Command Usage

```bash
export WESHOP_API_KEY=<key>

weshop qwen-image-edit --prompt '<instruction>' \
    [--image <path|url> ...] \
    [--batch <n>] [--task-name <name>] [--no-wait]
```

CLI options:

| CLI option | Required | Type / values | Mapped field | Description |
|---|---|---|---|---|
| `--prompt <text>` | **Yes** | string | `params.textDescription` | Edit / generation instruction |
| `--image <path\|url...>` | No | path/url, **max 5** (repeatable) | `params.images`, `input.originalImage` | Reference images: local path or URL |
| `--batch <count>` | No | int 1–16 (default 1) | `params.batchCount` | Number of images to generate |
| `--task-name <name>` | No | string | `input.taskName` | Run label |
| `--no-wait` | No | boolean | — (sync control) | Return immediately after submission; check later with `weshop status <id>` |

> - Local paths are uploaded automatically (see §3) and cached by path+size+mtime in `~/.weshop-cli/upload-cache.json`.
> - With no image, the model generates from the prompt alone; with images, image 1 is the base and images 2..N are style/reference.
> - More than 5 `--image` args cause the command to error and exit.

### 2.2 Input (request body fields)

```jsonc
{
  "agent": { "name": "qwen-image-edit", "version": "v1.0" },
  "params": {
    "textDescription": "Make the sky more dramatic",   // string, required
    "batchCount": 1,                                   // number, 1-16, optional
    "images": ["https://.../a.png", "https://.../b.png"] // string[], optional, ≤5 URLs
  },
  "input": {
    "originalImage": "https://.../a.png",   // string, first image URL (set when images present)
    "taskName": "my-label"                  // string, optional
  },
  "safeGenerate": "off"
}
```

`params` field details:

| Field | Type | Required | Description |
|---|---|---|---|
| `textDescription` | string | Yes | Edit / generation instruction |
| `batchCount` | number | No | 1–16, default 1 |
| `images` | string[] | No | Reference image URL array, ≤5 (URLs resolved/uploaded from `--image`) |

`input` field details:

| Field | Type | Required | Description |
|---|---|---|---|
| `originalImage` | string | No | First reference image URL (auto-set to the first image by `run-helper` when `images` is present) |
| `taskName` | string | No | Run label |

> Mapping logic (from the `run-helper.js` multi-image path):
> - Each `--image` is resolved to a URL (local → upload, URL → passthrough) and collected into `params.images`.
> - If `input.originalImage` is not explicitly set, `input.originalImage = params.images[0]`.
> - Note: the `run-helper` **multi-image path only writes `params.images`, not `input.images`**.

### 2.3 Output (result structure)

Same as `z-image` (see §1.3):

```jsonc
{
  "executionId": "<id>",
  "status": "Success",
  "result": [
    { "status": "Success", "image": "https://..." }
  ]
}
```

CLI prints the `[result]` block (agent name corresponds to `qwen-image-edit v1.0`).

---

## 3. `wan-ai` — Image-to-Video

> Video model; output is `video[]`, unlike the image models above (whose output is `image`).

### 3.1 Command Usage

```bash
export WESHOP_API_KEY=<key>

weshop wan-ai --image <path|url> --prompt '<description>' [--duration <time>] [--batch <n>] [--task-name <name>] [--no-wait]
```

CLI options:

| CLI option | Required | Type / values | Mapped field | Description |
|---|---|---|---|---|
| `--image <path\|url>` | **Yes** | path/url | `params.images`, `input.originalImage` | Input image (local path auto-uploaded to URL) |
| `--prompt <text>` | **Yes** | string | `params.textDescription` | Video scene description |
| `--duration <time>` | No | `3s` / `4s` / `5s`(default) / `6s` / `7s` / `8s` | `params.duration` | Video duration |
| `--batch <count>` | No | int 1–16 (default 1) | `params.batchCount` | Number of videos to generate |
| `--task-name <name>` | No | string | `input.taskName` | Human-readable label for this run |
| `--no-wait` | No | boolean | — (sync control) | Return immediately after submission; check later with `weshop status <id>` |

### 3.2 Input (request body fields)

```jsonc
{
  "agent": { "name": "wan-ai", "version": "v1.0" },
  "params": {
    "textDescription": "Ocean waves crashing on rocks", // string, required
    "duration": "5s",                                   // string, default "5s"
    "batchCount": 1,                                    // number, 1-16, optional
    "images": ["https://.../scene.png"]                 // string[], single input image URL
  },
  "input": {
    "originalImage": "https://.../scene.png",   // string, input image URL (from --image)
    "taskName": "my-label"                      // string, optional
  },
  "safeGenerate": "off"
}
```

`params` field details:

| Field | Type | Required | Description |
|---|---|---|---|
| `textDescription` | string | Yes | Video scene description |
| `duration` | string | No | Enum as above, default `"5s"` |
| `batchCount` | number | No | 1–16, default 1 |
| `images` | string[] | Yes | Input image URL array (resolved/uploaded from `--image`, single element) |

`input` field details:

| Field | Type | Required | Description |
|---|---|---|---|
| `originalImage` | string | Yes | Input image URL (set by the `run-helper` single-primary-image path) |
| `taskName` | string | No | Run label |

> Mapping logic (from the `run-helper.js` single-primary-image path): the `--image` URL is written to
> both `input.originalImage` and `params.images = [imageUrl]`.

### 3.3 Output (result structure)

`data.executions.at(-1)` is the final execution; `result` is the array of videos, each item:

```jsonc
{
  "executionId": "<id>",
  "status": "Success",   // Success | Failed
  "result": [
    { "status": "Success", "video": "https://...mp4", "videoPoster": "https://...jpg" }
  ]
}
```

CLI prints the `[result]` block:

```
[result]
  agent: wan-ai v1.0
  executionId: <id>
  status: Success
  videoCount: 1
  video[0]:
    status: Success
    url: https://...mp4
    poster: https://...jpg
```

`result[]` item fields:

| Field | Type | Description |
|---|---|---|
| `status` | string | Status of this item (`Success`, etc.) |
| `video` | string | Generated video URL |
| `videoPoster` | string | Video poster/thumbnail image URL (optional) |

---

## 4. Common Helper Endpoints

### 4.1 Upload Image

```
POST /openapi/agent/assets/images   (multipart/form-data, field "image")
→ data: { "image": "<url>" }
```

CLI: `weshop upload <file>`.

### 4.2 Submit a Run

```
POST /openapi/agent/runs   (body per §0)
→ data: { "executionId": "<id>" }
```

### 4.3 Query Run Status / Result

```
GET /openapi/agent/runs/<executionId>
→ data.executions.at(-1): { executionId, status, result:[...] }
```

### 4.4 Environment Variables / Auth

| Variable | Description |
|---|---|
| `WESHOP_API_KEY` | Required. Request header `Authorization: <key>` |
| `WESHOP_BASE_URL` | Optional, default `https://openapi.weshop.ai/openapi` |

---

## 5. Model Input / Output Cheat Sheet

### z-image (`v1.0`)

- Input: `params.textDescription`(req), `params.aspectRatio`, `params.batchCount`; `input.taskName`.
- Image input: **none**.
- Output: `result[]` items `{ status, image }`.

### qwen-image-edit (`v1.0`)

- Input: `params.textDescription`(req), `params.batchCount`, `params.images`(≤5); `input.originalImage`, `input.taskName`.
- Image input: supports up to 5 reference images (local paths auto-uploaded to URLs).
- Output: `result[]` items `{ status, image }`.

### wan-ai (`v1.0`)

- Input: `params.textDescription`(req), `params.duration`, `params.batchCount`, `params.images`(single); `input.originalImage`, `input.taskName`.
- Image input: single input image (local path auto-uploaded to URL).
- Output: `result[]` items `{ status, video, videoPoster }` (video model).

> Image models (`z-image`, `qwen-image-edit`) output `image`; the video model (`wan-ai`) outputs `video` + `videoPoster`.

---

## 6. Common Use Cases

```bash
# z-image: default 1:1
weshop z-image --prompt 'A futuristic cityscape at sunset'

# z-image: custom ratio + 2 images
weshop z-image --prompt 'Product photo of sneakers on white background' --aspect-ratio 3:4 --batch 2

# z-image: vertical 9:16, async
weshop z-image --prompt 'Asian model in streetwear' --aspect-ratio 9:16 --no-wait

# qwen-image-edit: text-only generation
weshop qwen-image-edit --prompt 'A cat sitting on a rainbow'

# qwen-image-edit: single-image editing
weshop qwen-image-edit --image ./photo.png --prompt 'Make the sky more dramatic'

# qwen-image-edit: multi-image blending (image 1 as base, image 2 as style reference)
weshop qwen-image-edit --image ./a.png --image ./b.png \
  --prompt 'Combine these two scenes using image 1 as the base and image 2 as the style reference' --batch 2

# wan-ai: default 5s
weshop wan-ai --image ./scene.png --prompt 'Ocean waves crashing on rocks'

# wan-ai: 8s duration
weshop wan-ai --image ./photo.png --prompt 'Person dancing in slow motion' --duration 8s

# wan-ai: async submit
weshop wan-ai --image ./landscape.png --prompt 'Sunrise over mountains' --no-wait
```
