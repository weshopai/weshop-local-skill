---
name: weshop-agent-task-runner
version: 2.0.0
description: Start a local web form service that dynamically wraps WeShop CLI agent parameters, runs tasks synchronously, and displays media results. For running WeShop AI tasks on the local machine through a browser.
compatibility: platform-neutral
---

# WeShop Local Form Service & CLI Executor

## End Goal

Deliver and start a web form service for local use only. The user fills in the Agent/model parameters in the browser, the server always executes the task synchronously via the latest WeShop CLI available from the official source, and returns the final media results to the page in the same request.

The completion criteria for this Skill are not generating request JSON, project source code, or a "copy command" button. Instead: the service process is already running on the user's machine, the local address is reachable, the page can be clicked to submit, the WeShop CLI is actually invoked, and verified final results are displayed. Delivering only files without starting the service counts as incomplete.

```text
Browser form
    ↓ HTTP (local only)
Local form service
    ↓ subprocess, WESHOP_API_KEY injected from environment
Latest WeShop CLI (synchronously waits for the result)
    ↓
Final task status, media URLs, or error
    ↓
Local form service returns and displays
```

## Service Boundaries

- The service listens only on a loopback address (`127.0.0.1` or `localhost`) by default, and must not be exposed to the LAN or public network by default.
- The browser never holds, displays, transmits, or saves the API key. The service process reads the credential from the `WESHOP_API_KEY` environment variable and checks for its presence at startup.
- The service must not write the API key, auth headers, cookies, or signatures into logs, HTML, JavaScript, request previews, task records, or result objects.
- Form submission must execute the CLI on the server side; the browser must not run shell commands directly.
- A static request preview must not replace the submit capability. If the CLI, the Agent schema, or the local service is unavailable, the page must show a clear reason why it cannot run and disable submission.

## Local Run Prerequisites

| Config | Required | Description |
| --- | --- | --- |
| `WESHOP_API_KEY` | Yes | Read only from the environment by the local service process. |
| `WESHOP_CLI_COMMAND` | Yes | The WeShop CLI executable command. |
| `WESHOP_LOCAL_SERVICE_URL` | Only when the CLI supports a local service | A trusted local service address, e.g. `http://127.0.0.1:<port>`. |
| `HOST` / `PORT`            | No                        | Local form service listen address; defaults to a loopback address and an available port. |

Before starting or handling the first request:

1. Before each service start and before each task execution, check that the CLI exists, its current version, and the latest version available from the official install source.
2. Check the current weshop CLI version; if the CLI is missing or the current version is not the latest, it must be installed/upgraded to the latest from the official source (`npm i weshop-cli`); then re-confirm the installed version and help output. Do not reuse an old CLI, build the form from old help output, or silently fall back to a lower version because an upgrade failed.
3. Verify that `WESHOP_API_KEY` was injected from the environment, but never print its value.
4. If using a local WeShop service, verify `WESHOP_LOCAL_SERVICE_URL` is a trusted local address and configure it in the way the CLI actually supports.
5. Provide a health check after starting the form service; the "Submit generation task" action is enabled on the page only when all the above conditions are met.

## Mandatory Startup & Delivery

After implementation is complete, the Agent must actually start the service on the user's machine itself, rather than only writing README start commands, asking the user to run commands, or requiring the user to troubleshoot on their own. The Agent is responsible for installing/upgrading to the latest WeShop CLI from the official source, installing required dependencies, and starting and keeping the service process alive until the user can access and use the page.

After startup, confirm in order:

1. The service has successfully bound a loopback address. If the default port is occupied, permission is denied, or startup fails, automatically probe and try other free loopback ports in sequence until the service starts successfully; after each switch, re-run the health check and report the final actual address. Report startup failure only if no port is available.
2. The health check returns CLI readiness without leaking any secrets.
3. Confirm from the user's browser or an accessibility check that `http://127.0.0.1:<actual port>` is reachable; an `ERR_CONNECTION_REFUSED` means the service was not fully delivered.
4. The page is not a static JSON preview, and the "Submit generation task" button is enabled when the configuration is ready.

The final delivery must include: a running local service, a local URL the Agent opened or can be opened directly, and a verified health check result. If a port is unavailable, the Agent must automatically switch ports and continue. If the current execution environment lacks the permission to run, listen on a port, or install dependencies, the Agent must request the needed permission and complete the work itself once granted; it must not hand commands, README, or manual startup steps to the user as the final delivery.

## CLI-Driven Dynamic Form

The user provides the model/Agent intent and the parameter items needed, e.g.: "use the qwen image edit model, parameters include prompt, number of images, and image". The user does not need to know the CLI parameter names.

After the user selects a model, the service must obtain the real contract from the currently installed WeShop CLI:

1. Match the user's wording against the CLI's agent list, global help, or equivalent capability queries to confirm the actual Agent/command and version. (Do not expose other Agents.)
2. Read that Agent's `--help` or the schema/describe output provided by the CLI.
3. Extract the real fields: parameter names, types, required flags, enums, defaults, numeric ranges, repeatable-value rules, and the media input requirements for files or URLs.
4. Generate the form dynamically. The form language may follow the local machine's environment language; if it cannot be determined, use English, but submission must use the parameter names the CLI actually supports.
5. Fields required by the CLI but not mentioned by the user must be shown and filled in by the user; fields not declared by the CLI must not be presented as submittable parameters.

For example: only when the current CLI explicitly exposes `--batch` can "number of images" be shown and `--batch` be submitted; only when the CLI explicitly requires an image can a file picker or URL input be offered accordingly. Do not guess fields from screenshots, historical forms, or common sense.

### Fixed Runtime Fields

The following fields are managed by the local form service and are not derived from CLI parameter discovery. The CLI request must include `--safeGenerate='off'`.

- `safeGenerate`: the form must provide an `off` / `on` option. Pass it only when the local service/CLI adapter layer explicitly supports it; otherwise keep it in the local request metadata and note that it was not passed to the Agent.

Apart from the above fields, all Agent-editable parameters must come from the verified output of that CLI run.

## Local Service Interface Responsibilities

The interface paths and tech stack are an implementation choice, but the service must provide equivalent capabilities:

| Capability | Service behavior |
| --- | --- |
| Health check | Returns CLI readiness, version, and secret-free failure reasons. |
| Agent discovery | Returns the Agent/model candidates available in the current CLI. |
| Parameter schema | Given an Agent/model, returns field definitions from the CLI usable to build the form. |
| Synchronous task submission | Validates form data, safely invokes the CLI, and waits for the final result within the request. Records the returned executionId. |
| Page UI | Renders the dynamic form, in-progress state, task errors, and accessible successful media. |
| Query task | Uses the executionId to query and get the status of this execution. |
| Upload image | Uploads a local image to obtain a URL. |

The submit interface may only accept fields allowed by the current schema. To prevent shell injection: invoke the CLI with a structured argument array, never concatenate user text, URLs, or JSON directly into a shell string.

## Synchronous Task Execution & Result Display

1. Re-validate the submitted values on the server against the current CLI schema.
2. If the Agent supports image input, provide an upload entry point; get a URL via the CLI upload interface and pass it in on submission.
3. Run the CLI with the `WESHOP_API_KEY` environment variable; never pass it as `--api-key`, `--apiKey`, or any ordinary command argument.
4. If a synchronous submission returns no final result after more than 5 minutes, it may be interrupted. Use the recorded executionId and the query interface to get the current task status. If it is still running, poll every 2 seconds to wait for the result.
5. Execute the task using the local service config, Agent, media input, and parameters the CLI actually supports, and wait synchronously for the CLI to exit and return the final result.
6. Parse the CLI output and display the final status, sanitized errors, and each media result on the page. Each successful media item is returned as a URL:
   - After validating the URL is non-empty, render the image/video preview directly and provide a clickable link.
7. Show task success only when the CLI has returned a terminal success and every expected media item succeeded with a non-empty URL.

If the CLI process times out, is cancelled, or exits abnormally, return the corresponding status and sanitized error; do not automatically resubmit, to avoid duplicate generation and billing.

## Page States

| State | Page behavior |
| --- | --- |
| `initializing` | Shows that the CLI and local config are being checked; cannot submit. |
| `ready` | Agent schema obtained; can submit. |
| `validation_error` | Highlights field errors; does not run the CLI. |
| `submitting` / `running` | Shows that the CLI synchronous return is being awaited; avoids duplicate clicks. |
| `success` | Displays all successful media results (URLs). |
| `partial_success` | Displays successful media (URLs) and failed items. |
| `failed` / `timeout` | Shows a sanitized error and safe next steps. |

## Verification & Delivery

Before reporting completion, verify the following observable results:

1. The service listens only on a local address, and the browser source, network requests, and service logs contain no API key.
2. The service process is running, the actual local URL is reachable, and the health check succeeds; a README or unstarted project files cannot substitute.
3. The page shows Agent fields obtained from the current CLI, not hard-coded `z-image` or historical JSON.
4. The form button calls the local service's synchronous submit interface, rather than merely copying JSON.
5. Do not actually generate via the CLI during testing.
6. Use a supported Agent to complete the full loop from filling in, synchronous execution, to media result display.
7. Check each media result's terminal success and non-empty URL; do not claim completion based only on "CLI started".
8. Verify that a URL result can be previewed as an image/video on the page and provides a clickable link.

When the CLI does not provide the model or parameter the user specified, the service must clearly state the current CLI's support and block submission; it must not fabricate fields, silently drop parameters, or degrade to a static preview.
