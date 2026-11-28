# off-evaluator

The OFF Evaluator (OVA) decides whether a data asset snapshot is fit for a declared purpose. It belongs to the [Open Fitness Framework (OFF)](https://github.com/open-fitness-framework/off-specification) reference implementation and is built on the shared library [off-core](https://github.com/open-fitness-framework/off-core), which contains the deterministic evaluation procedure.

The OVA:

- **manages trust policies:** versioned, immutable policy snapshots that only administrators can change. They are never taken from reports or from the catalogue;
- **evaluates requests:** it checks signed fitness reports against a fitness profile, under a pinned trust policy, at an explicit evaluation time. The result is FIT, NOT_FIT or NOT_ASSESSABLE, with traces of accepted and excluded evidence;
- **builds requests:** given a snapshot binding, profile and policy, it finds all reports in the [off-catalogue](https://github.com/open-fitness-framework/off-catalogue), negative ones included, pins their bytes, and states which sources it searched;
- **returns everything needed to repeat the evaluation:** the decision, the pinned request and the evaluation record. It keeps nothing itself.

The first version covers the OFF core, with an HTTP API and a CLI that offer the same functions.

**Status:** not yet implemented. See the [briefing](docs/briefing.md) for the planned scope.
