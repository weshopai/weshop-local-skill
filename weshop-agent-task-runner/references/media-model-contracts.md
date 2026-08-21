# WeShop CLI Media Model Contracts

Reference snapshot: `weshop-cli` 0.2.12. Consult this only after live CLI discovery; its parameter names, model availability, defaults, and result shapes may be stale.

## Shared execution shape

The three known media commands submit an agent run and eventually produce a terminal execution with `executionId`, `status`, and `result[]`. A local service should retain the execution ID, display only sanitized errors, and consider an item usable only when its media payload is non-empty.

## Known agents

| Agent | Purpose | Required inputs | Optional inputs | Expected result item |
| --- | --- | --- | --- | --- |
| `z-image` (`v1.0`) | Text-to-image | prompt | aspect ratio; batch; task name | `{ status, image }` |
| `qwen-image-edit` (`v1.0`) | Text-to-image or image editing | prompt | 0–5 images; batch; task name | `{ status, image }` |
| `wan-ai` (`v1.0`) | Image-to-video | one image and prompt | duration; batch; task name | `{ status, video, videoPoster? }` |

### z-image

Known command shape:

```bash
weshop z-image --prompt '<description>' [--aspect-ratio <ratio>] [--batch <n>] [--task-name <name>] [--no-wait]
```

The 0.2.12 reference supports aspect ratios `1:1`, `2:3`, `3:2`, `4:3`, `3:4`, `16:9`, `9:16`, and `21:9`; default is `1:1`. Batch is 1–16. It does not accept image input. Successful result items use `image` URLs.

### qwen-image-edit

Known command shape:

```bash
weshop qwen-image-edit --prompt '<instruction>' [--image <path-or-url> ...] [--batch <n>] [--task-name <name>] [--no-wait]
```

It accepts prompt-only generation or up to five reference images. The first image is the base image; further images are references. Local paths are uploaded by the CLI. In the 0.2.12 request mapping, resolved references go to `params.images` and the first one is mirrored to `input.originalImage`; do not assume `input.images` exists. Successful result items use `image` URLs.

### wan-ai

Known command shape:

```bash
weshop wan-ai --image <path-or-url> --prompt '<description>' [--duration <time>] [--batch <n>] [--task-name <name>] [--no-wait]
```

It needs exactly one input image. The snapshot lists `3s` through `8s` duration choices and default `5s`; batch is 1–16. The mapped request uses a single-item `params.images` array plus `input.originalImage`. Successful result items use `video` and may include `videoPoster`.

## Uploading and status

The snapshot lists `weshop upload <file>` for image upload and `weshop status <execution-id>` for later status checks. Prefer the installed CLI's documented commands. If a synchronous command has no terminal result, query only its recorded execution ID; never submit again automatically.

## Rendering rules

- Render `image` as an image and `video` as a video; use `videoPoster` when available.
- Display an accessible link alongside each preview.
- If the CLI returns Base64 rather than a URL, validate the media type and construct a data URL for preview; do not expose raw payload as the only result.
- A mixed terminal result is partial success, not success.
