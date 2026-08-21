# Operational Specification: Local WeShop CLI Form Service

This reference defines a local browser service that discovers an installed WeShop CLI contract, validates a selected agent's inputs, runs the CLI server-side, and displays terminal media results. The service is a local convenience layer; it must never impersonate a CLI feature or fabricate a model field.

## Boundaries and authorization

- Bind only to `127.0.0.1` or `localhost` unless the user explicitly authorizes a different exposure.
- Read `WESHOP_API_KEY` only from the service process environment. Do not include it in browser assets, requests, saved records, command arguments, logs, or error messages.
- Use a structured subprocess argument array. Never concatenate user input into a shell command.
- Verify the CLI command, version, and help/schema before starting and before task execution. Check the official source for an upgrade only when that is relevant; installing or upgrading requires user authorization.
- Validate a configured local service URL as loopback and use it only if the current CLI supports it.
- A static request preview is not a usable runner. If readiness, CLI discovery, or schema discovery fails, explain the non-secret reason and disable submission.

## Required service capabilities

| Capability | Required behavior |
| --- | --- |
| Health check | Return readiness, CLI version, and non-secret failure reasons. |
| Agent discovery | Return only agents available from the installed CLI. |
| Schema discovery | Parse current CLI help/schema into names, types, requiredness, enums, defaults, numeric bounds, repeated values, and media requirements. |
| Form UI | Build the selected agent's form from discovered fields; show any CLI-required field even if the user did not mention it. |
| Submit | Revalidate allowed fields server-side, invoke CLI synchronously, and retain the returned execution ID. |
| Query | Query a recorded execution ID to obtain a final result; never create a new run during a query. |
| Upload | When current CLI supports file upload, accept local media and obtain a URL through that CLI path. |
| Results | Render sanitized errors and accessible image/video previews and links. |

Runtime-only controls such as safety and preferred result encoding may be represented in the UI only when the CLI or adapter explicitly supports them. Do not pass undocumented flags merely because a historical contract mentions them.

## Submission lifecycle

1. Discover the selected agent's real contract and create a fresh form schema.
2. On submit, validate values against that schema again on the server. Reject undeclared fields.
3. Resolve media through the supported upload path when necessary, then invoke the CLI with `WESHOP_API_KEY` in the process environment.
4. Await the terminal CLI response. If an interrupted or timed-out command produced an execution ID, use that exact ID for status polling; do not automatically resubmit because the outcome may be billable.
5. Parse each result item based on the actual CLI output. A non-empty URL is previewable; valid Base64 requires an identified MIME type and conversion to a browser data URL.
6. Mark a run successful only if the CLI reports a terminal success and every expected media item is usable. Otherwise report `partial_success`, `failed`, or `timeout` with sanitized context.

Use page states `initializing`, `ready`, `validation_error`, `submitting`/`running`, `success`, `partial_success`, `failed`, and `timeout`. Disable duplicate clicks while the run is in progress.

## Delivery checks for a running service

When a user asks for an operational local service, it is not delivered until all checks pass after every start, restart, or port change:

1. Confirm the chosen loopback address is in `LISTEN` state using a system-level listener check.
2. Send `GET /api/health` with a short timeout and require a 2xx non-secret readiness response.
3. Send `GET /` with a short timeout and require a 200 HTML form page—not JSON or a redirect placeholder.
4. Recheck that the process is still alive and the listener is still present.

If a port is occupied, select another available loopback port and repeat every check. Do not report a stale or refused URL. Never use an actual paid generation merely as a test unless the user explicitly requests it; a mocked or non-billable end-to-end path is preferable.
