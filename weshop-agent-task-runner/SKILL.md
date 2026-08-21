---
name: weshop-agent-task-runner
description: Build or operate a localhost-only browser form for WeShop CLI media agents, with live schema discovery, safe synchronous submission, and image/video result display.
---

# WeShop Agent Task Runner

Use this skill when a user needs a browser-based local runner for WeShop image or video agents. It packages two concerns: operating a safe local form service and handling the known media-agent contracts.

## Authority and boundaries

The installed CLI's current help/schema is authoritative for executable parameters, defaults, and supported agents. The media reference is a design and parsing aid—not permission to hard-code fields or to bypass live discovery.

Do not expose the service beyond loopback by default. Keep `WESHOP_API_KEY` exclusively in the service process environment; never place it in browser code, arguments, persisted task data, or logs. Invoke the CLI with an argument array, never a shell command string. Treat timeouts or malformed submission responses as outcome-unknown: do not automatically resubmit a potentially billable generation.

Starting a service, installing/upgrading software, and generating paid media are external actions. Do them only when the user's request authorizes them. When delivery does include a running service, validate listener, `/api/health`, `/`, and process liveness before reporting its address.

## Workflow

1. Read [the operational specification](references/operational-spec.md) before designing, implementing, or operating the service.
2. Discover the requested agent from the installed CLI and parse its current help/schema. Render only supported editable fields, with client and server validation.
3. For `z-image`, `qwen-image-edit`, or `wan-ai`, read [the media model contracts](references/media-model-contracts.md) to correctly route media inputs and parse image/video outputs. Confirm any disagreement against live CLI output.
4. Submit synchronously through the local service, record any execution ID, and display sanitized terminal results. Render non-empty URLs directly and support valid Base64 only when the CLI actually returns it.
5. Before declaring a running local service delivered, complete the liveness gates described in the operational specification. Never use a real generation as a smoke test without explicit user authorization.

## Result acceptance

Do not report success merely because a process started or a request was accepted. Require a terminal CLI outcome and a usable media result for every expected item; otherwise show partial success or a sanitized failure state.
