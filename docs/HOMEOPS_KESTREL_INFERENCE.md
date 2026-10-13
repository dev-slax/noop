# HomeOps / Kestrel inference context for NOOP

> **Status: design/deployment context only — not a shipped NOOP capability.**
>
> This document exists so future NOOP work does not accidentally install NOOP on Kestrel, bypass LiteLLM, or turn a private HomeOps experiment into a default NOOP dependency.

## Objective

Use the docked Pixel 8 Pro named **Kestrel** only as a private local inference worker for bounded NOOP Coach model requests.

The NOOP application itself is **never installed or executed on Kestrel**.

Target deployment path:

```text
NOOP Coach caller
  -> canonical `noop-coach` capability
  -> LiteLLM
  -> `kestrel-local`
  -> Kestrel Pixel 8 Pro / local Qwen3 worker
  -> generated completion
  -> LiteLLM
  -> NOOP Coach caller
```

LiteLLM remains the model-routing and policy boundary. Kestrel is only a physical inference worker.

## What stays in NOOP

NOOP continues to own:

- strap communication and collection;
- on-device SQLite state;
- deterministic health/recovery/strain/sleep calculations;
- Coach request construction and safety/disclaimer behavior;
- the user-visible Coach experience.

Kestrel must not become a NOOP runtime, database, scheduler, state authority, or second copy of the application.

## What may reach Kestrel

Only a bounded model-inference request selected by LiteLLM may reach Kestrel.

Prefer a compact deterministic Coach context rather than raw or unbounded health history, for example:

```json
{
  "goal": "maintain today's plan",
  "recent_pattern": "2 completed, 1 deferred",
  "energy": "medium",
  "streak": 4,
  "user_note": "Long afternoon",
  "requested_mode": "brief_check_in"
}
```

Kestrel locally generates response tokens and returns the completion to LiteLLM. NOOP should not know or care which physical worker produced the response.

## Credentials and network boundary

NOOP must never contain:

- a Kestrel endpoint;
- a Kestrel worker bearer credential;
- the LiteLLM master key;
- model-provider API keys.

The Kestrel worker credential exists only between approved HomeOps infrastructure and the worker. Kestrel itself does not receive NOOP credentials or provider credentials.

## Current NOOP scope constraint

NOOP is currently documented as offline-by-default with no NOOP-operated server or cloud dependency. The repository's present narrow network exception is a default-off one-way export to a user-owned endpoint.

A two-way in-app Coach inference request to LiteLLM therefore **must not be introduced casually as a normal shipped feature**. Before implementing such a path in NOOP itself, reconcile it explicitly with the project's scope/governance and obtain the appropriate issue/approval.

This document does not override `AGENTS.md`, `docs/SCOPE.md`, or the offline architecture contract.

For private HomeOps testing, Kestrel should be treated as external deployment infrastructure, not as part of NOOP's canonical on-device data path.

## Rollout intent

The HomeOps side is expected to prove Kestrel independently before NOOP changes production routing:

1. Kestrel local model build/device UAT passes.
2. The Pixel sustains at least 100 sequential local generations without crash/OOM or severe thermal state.
3. LiteLLM -> `kestrel-local` inference passes.
4. Background/undock/thermal conditions fail closed.
5. NOOP runs a shadow-quality comparison using at least 25 representative Coach scenarios.
6. Only after quality is accepted may HomeOps repoint the canonical `noop-coach` capability to Kestrel.

At cutover, NOOP continues to request only `noop-coach`; the application does not gain Kestrel-specific code or configuration merely because the physical inference worker changed.

## Failure and rollback

If Kestrel is unavailable, NOOP must not attempt a direct connection to it.

Routing and rollback belong to HomeOps/LiteLLM. The previously proven local route remains the rollback target until a different fallback policy is explicitly approved.

No hosted/public fallback is implied by this document.

## Non-goals

This context does **not** authorize:

- installing NOOP on Kestrel;
- syncing the NOOP database to Kestrel;
- moving health-state authority to Kestrel;
- direct NOOP -> Kestrel calls;
- direct NOOP -> model-provider calls;
- putting LiteLLM/provider secrets on the phone;
- making Kestrel a required dependency for normal NOOP operation;
- weakening NOOP's offline-by-default behavior without an explicit scope decision.

The intended boundary is simple:

> **NOOP owns the product and health truth. LiteLLM owns model routing. Kestrel only generates bounded local inference responses when selected by LiteLLM.**
