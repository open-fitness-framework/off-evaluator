# Briefing: OFF Evaluator (`off-evaluator`, OVA)

**Status:** draft for `/init` · **Specification:** [Open Fitness Framework (OFF) 0.3 draft](https://github.com/open-fitness-framework/off-specification/blob/main/spec/0.3/open-fitness-framework-spec.md) · **Builds on:** [`off-core`](https://github.com/open-fitness-framework/off-core)

## 1 Context

The Open Fitness Framework (OFF) lets a consumer decide whether a data asset snapshot is fit for a declared purpose. It does so by evaluating signed fitness reports against a fitness profile, under the consumer's own trust policy, at an explicit evaluation time. The outcome is a fitness decision (FIT, NOT_FIT or NOT_ASSESSABLE) with a trace of which evidence was accepted or excluded and why (Sections 10, 11).

The reference implementation has three components on top of the shared library `off-core`:

| Component | Spec roles | Purpose |
| --- | --- | --- |
| OFF Engine (OFE) | Assessor, report issuer; snapshot binding | Runs fitness checks and issues signed reports. |
| OFF Catalogue (OFC) | Repository, discovery; profile and definition publication | Makes reports, profiles and definitions findable and retrievable. |
| **OFF Evaluator (OVA)**, this project | Evaluator; trust policy store; request building | Builds evaluation requests and produces fitness decisions. |

The deterministic evaluation procedure is implemented in `off-core` as a pure function. The OVA is the service and tool around it: it manages trust policies, gathers and pins the inputs, calls the evaluator and returns the result.

## 2 Users

- **Agents** that need a fit-for-purpose decision before using a data asset, for example to select training data or to decide whether to submit or list a dataset.
- **Data consumers and their applications**, as a service or in CI pipelines.
- **Policy owners**, who maintain the organisation's trust policies.

## 3 Responsibilities and scope of the first version

The first version implements the OFF core and no extensions. A profile, policy or request that declares an extension yields an evaluation error with UNSUPPORTED_EXTENSION (Section 2.2).

### 3.1 Trust policy store (Section 8.2)

- **Versioned policies:** the OVA keeps the organisation's trust policies as versioned, immutable snapshots, each identified by id + digest.
- **Admin-only changes:** only policy administrators may add a new version. Each change records who made it and when.
- **Independent authorisation:** "the application MUST authorize the pinned policy independently of report content" (8.2). Policies are never taken from the OFC, from reports or from request content. A request can only pin a policy that is in the store.
- **Updates and key removal** create a new version; old versions stay available so that earlier decisions can be reproduced (8.2).
- **Building policies:**
  - import of issuer keys from a JWKS URL, as a draft;
  - validation of the policy with `off-core`;
  - a diff against the previous version before an administrator publishes it.

  The OVA never trusts a key automatically because it was fetched.

### 3.2 Evaluating supplied requests (Sections 9, 10)

- **Input:** a complete `EvaluationRequest` with every pinned artifact either supplied with the call or resolvable. Artifacts are resolved through `off-core`'s resolver interface, backed by the call's supplied files, the OFC and configured HTTPS locations, and digests are always verified.
- **Retrieval** follows Section 13: allowed schemes and destinations, redirect, size and timeout limits, and all fetched content is treated as untrusted input.
- **Evaluation:** the OVA calls `off-core`'s evaluator with the request, the resolved artifacts, the policy and the configured definition plug-ins.
- **Declared definition support:** the OVA states which definitions it supports through the plug-ins installed in it (7.1). It exposes this list so that callers can tell UNSUPPORTED_DEFINITION apart from missing evidence.

### 3.3 Request building (outside the core)

- **Given** a subject binding, a profile (by id + digest, or a purpose to look up), a policy from the store and an evaluation time (default: now), the OVA builds the request itself:
  1. it queries the OFC, and optionally other configured sources, for every report about the snapshot;
  2. it fetches them and pins their exact bytes in `reportResources`;
  3. it evaluates as in [Section 3.2](#32-evaluating-supplied-requests-sections-9-10).
- **No filtering:** all reports found are pinned, including negative ones. The builder never filters by result (5.3).
- **Stated discovery scope:** the decision's `limitations` say which sources were searched, instead of "submitted-only evidence set". An evaluator that discovered nothing must not imply an exhaustive search (Section 11).
- **Returned request:** the built request is returned with the decision, so that the caller can re-run exactly the same evaluation later.

### 3.4 Results and retention (Section 11)

- **The OVA returns the decision and keeps nothing.** Each call returns:
  - the `FitnessDecision`;
  - the pinned `EvaluationRequest`;
  - the **evaluation record**, which accounts for every input report as accepted, excluded, duplicate or irrelevant, as Section 11 requires.
- **Retention** is the caller's responsibility, and the documentation says so. Nothing is stored, so there are no decision histories in the first version.
- **Decision signing** (Section 11, MAY) is out of scope for the first version.

### 3.5 Interfaces

- **HTTP API** with an OpenAPI description:
  - `POST /evaluations` for supplied requests;
  - `POST /evaluations:build` for request building;
  - `GET /definitions/supported`;
  - policy store administration, separately authenticated.
- **CLI with the same functions:** `evaluate`, `build-and-evaluate`, `policy` (import, validate, diff, publish).
- **Callers:** both the API and the CLI are suitable for agents and CI pipelines.
- **Delivery:** as a container image and a runnable JAR.

## 4 Out of scope for the first version

- Extensions E1 to E5. E4 (status checks, `statusChecks` in decisions) and E2 (evidence retrieval) come first, then E1, E3 and E5.
- Storing decisions or requests, and decision signing.
- Scheduling, monitoring and automatic re-evaluation when new evidence, status changes or policy versions appear (Section 12).
- Ordering assessments from the OFE when evidence is missing. An agent may do this using the decision.
- Selecting a profile from natural-language purposes. Agents may do this, but the OVA requires an exact profile or purpose id. "Natural-language inference MUST NOT substitute for a missing assertion" (Section 2).

## 5 Constraints

- **Language:** Java, the same version and build tool as `off-core` (to be confirmed at `/init`).
- **Determinism:** for a given request and policy, the decision is the same on every run and every machine. The OVA adds no clock: `evaluatedAt` is explicit in every request, and the builder only sets it from the clock if the caller omits it.
- **Data protection:** the OVA does not process data assets, only reports, profiles and policies. Policy administrators' identities are recorded with policy changes.
- **Factory runtime:** the Software Factory image ships Node and Python only, so a `.factory/Dockerfile` with a JDK is needed.

## 6 Success criteria

- Evaluating the core-only scenarios of the published guides (tree-vitality orthoimagery, INSPIRE road network) reproduces every documented decision, both from supplied requests and through the request builder against an OFC that holds the same artifacts.
- The OVA passes the `off-conformance` evaluator class through its CLI.
- A request that pins a policy not in the store is an evaluation error. A policy can never be supplied through report or catalogue content.
- The request builder pins negative reports and states its discovery scope in `limitations`.

## 7 Candidate feature map

| # | Feature | Depends on |
| --- | --- | --- |
| 1 | Trust policy store: versions, administrators, validation, JWKS import, diff | off-core |
| 2 | Resolvers: supplied files, OFC, configured HTTPS with Section 13 controls | off-core |
| 3 | Evaluation of supplied requests, with plug-in configuration and supported-definition listing | 1, 2 |
| 4 | Request building through OFC discovery | 2, 3 |
| 5 | HTTP API | 3, 4 |
| 6 | CLI | 3, 4 |

## 8 Open questions for `/init`

- Authentication of API callers and policy administrators.
- Which definition plug-ins ship by default: the OFE's first definitions (schema conformity, attribute completeness) at least.
- License, as for the other OFF repositories.
