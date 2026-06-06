![logo_ironhack_blue 7](https://user-images.githubusercontent.com/23629340/40541063-a07a0a8a-601a-11e8-91b5-2f13e4e6b441.png)

# Lab | FastAPI for ML Serving

## Overview

You will not write a serving framework today. You will write the **contract** that any serving framework would have to implement: a complete OpenAPI 3.1 specification for a model-serving API, plus example payloads that validate against it.

This is a 90-minute design lab. You'll produce YAML and JSON. No Python, no notebooks. The deliverable is a spec you could hand to a frontend team and a backend team and they'd both know exactly what to build.

## Learning Goals

By the end of this lab you should be able to:

- Author an OpenAPI 3.1 spec with explicit schemas, error responses, and security schemes
- Design batch and async endpoints alongside the synchronous default
- Embed observability hooks (request ID, model version) in the contract itself

## Setup

Fork and clone the lab repo. You need:

- A YAML/JSON editor (any IDE)
- A working OpenAPI validator. Pick **one**:
  - [Swagger Editor (web)](https://editor.swagger.io/) — paste your spec, it validates live
  - `redocly lint openapi.yaml` (Node CLI, no Python)
  - `spectral lint openapi.yaml` (Node CLI)

Bring something that says "spec is valid" green. Submissions with broken specs are auto-rejected.

## The Scenario

You are designing the public API for **Vision Moderation as a Service** — a SaaS that classifies uploaded images for prohibited content. Partner customers integrate it into their upload flows.

Operational facts you must respect:

- Inputs are images up to **5 MB** (JPEG/PNG/WebP)
- The model returns the top **K** labels with calibrated probabilities, K configurable per request (1–10, default 5)
- p95 latency target: **300 ms** for synchronous calls; longer images may take seconds (async path)
- Peak traffic: 200 RPS; batch endpoint must accept up to 32 images per call
- Authentication: API key in a header
- Multi-tenant — every request belongs to a tenant; tenant is derived from the API key, not sent in the body
- Model version is part of every response

## Tasks

### Task 1 — Author `openapi.yaml`

Write a single `openapi.yaml` file at the repo root using OpenAPI **3.1**. It must include:

**Endpoints**

- `POST /v1/predict` — synchronous, single image
- `POST /v1/predict-batch` — synchronous, up to 32 images
- `POST /v1/predict-async` — accepts a single image, returns a job ID immediately (202)
- `GET /v1/predictions/{job_id}` — returns the async result

**Schemas**

- `PredictRequest`, `PredictResponse`
- `BatchPredictRequest`, `BatchPredictResponse` (each item keyed by a caller-supplied `id`)
- `AsyncSubmitResponse`, `AsyncResult`
- A reusable `Error` schema with `code`, `message`, `request_id`

**Responses**

- Document **at least** these status codes per endpoint where they apply: `200`, `202`, `400`, `401`, `413`, `422`, `429`, `503`
- Each error response uses the shared `Error` schema

**Headers**

- `X-Request-ID` on every response
- `X-Model-Version` on every successful prediction response

**Security**

- `securitySchemes` with `ApiKeyAuth` (header `X-API-Key`); applied globally

Your spec must lint clean. Include a `make lint` target or a one-line README command that runs the validator.

### Task 2 — Example payloads

In `examples/` produce at least these JSON files:

```
examples/predict-request.json
examples/predict-response.json
examples/predict-error-413.json
examples/batch-request.json
examples/batch-response.json
```

Each file must be a **valid instance of the corresponding schema in your spec** — not aspirational free-form JSON. Include realistic values (real-looking labels, IDs, request IDs, model version strings). The base64-encoded image field can use a placeholder string for size — but document the size assumption in a comment-free README note.

### Task 3 — Decision write-up

Write `DECISIONS.md` (one page max) with three short sections:

1. **Versioning** — Why did you pick path-based (`/v1/`) over header-based versioning? Two sentences.
2. **Batch ordering and partial failures** — When `predict-batch` is called with 32 items and one image is corrupt, what does the response look like? (Pick a behavior and justify it.) Three sentences.
3. **Async lifecycle** — What states does a job go through, and what's the retention policy? (e.g. `queued → running → done`/`failed`; results kept for N hours.) Three sentences.

These are decisions a real API team has to make. Make them explicit.

## Submission

Open a PR to the lab repository with these files at the repo root:

```
openapi.yaml
examples/predict-request.json
examples/predict-response.json
examples/predict-error-413.json
examples/batch-request.json
examples/batch-response.json
DECISIONS.md
```

Paste the PR link as your deliverable. The PR description must include the lint command that proves the spec is valid.

## Quality bar

You will be reviewed on:

- **Does the spec lint clean?** A spec with errors fails automatically.
- **Are batch responses keyed by caller-supplied IDs**, or do they rely on input order?
- **Are error responses structured and consistent** across endpoints?
- **Is the model version in the response header**, or did you bury it in the body?
- **Do your example payloads validate against your own schemas?**

Design-first thinking. No frameworks, just contracts.
