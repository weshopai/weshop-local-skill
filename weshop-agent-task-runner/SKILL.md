---
name: weshop-agent-task-runner
description: Build and run a localhost-only Web form that discovers a WeShop CLI agent schema and submits a synchronous WeShop task. Use when a user needs a local browser form for a WeShop agent.
---

# WeShop Agent Task Runner

Read [the full operational specification](references/operational-spec.md) before implementing or running a local WeShop form service. Its delivery and validation requirements are mandatory.

Run the service only on a loopback address. Read `WESHOP_API_KEY` only from the service process environment; never expose or log its value. Discover the chosen agent's real CLI contract before rendering editable fields and invoke the CLI with structured arguments, never a shell string.

The service is not delivered merely because source files, a PID file, or a startup log exist. After every start, restart, or port change—and immediately before reporting a URL—perform this mandatory port-liveness gate in order:

1. Verify the selected `127.0.0.1:<port>` is actively in `LISTEN` state using a system-level listener check.
2. Send an actual HTTP `GET /api/health` with a short timeout; require HTTP 2xx and a non-secret ready response.
3. Send an actual HTTP `GET /` with a short timeout; require HTTP 200 and an HTML form page, not JSON or a redirect placeholder.
4. Recheck that the service process remains alive and its listener remains present. Record the final port and the four verification outcomes without secrets.

If any condition fails, including `ERR_CONNECTION_REFUSED`, the service is not delivered. Re-establish it with a persistent process and an available loopback port, then rerun all four checks. Do not tell the user merely to refresh or rely on a previous address.

Never test by submitting an actual paid generation unless the user explicitly asks. Show results only after the CLI reports a terminal state and each expected media item has a non-empty URL or valid Base64 payload.
