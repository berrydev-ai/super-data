# Super Data MCP: Goals & Complexity Analysis

## Project Goals (README.md & SPECIFICATION.md)
- Deliver an MCP-compliant server that lets LLM clients run jq-backed queries across curated JSON datasets via stdio and HTTP transports.
- Centralize all tools, jq functions, and execution metadata in SQLite with seed data imported from YAML on first run.
- Provide a managed jq function library with dependency resolution so tools compile into executable jq programs automatically.
- Support multi-user team workflows: user attribution, shared tool catalog, and activity/usage tracking.
- Enable distributed operation through S3 synchronization, optimistic locking, and conflict recovery for the SQLite database.
- Offer dataset metadata management: schema generation, compatibility checks, and version tracking for JSON assets.
- Expose collaborative tooling: proposing, approving, and learning new jq-based tools, including automated validation and notifications.
- Maintain performance and operability through streaming query execution, caching, metrics, and observability hooks.
- Extend through optional future capabilities (WebSocket updates, additional query engines, external data providers, RBAC, etc.).

## Complexity-Based Prioritization
The table orders goals from lower to higher implementation complexity while factoring in the user note that data versioning and approval workflows are lower priority.

| Goal | Core Value | Primary Dependencies | Relative Complexity | Recommended Priority |
| --- | --- | --- | --- | --- |
| Foundational MCP runtime (transport wiring, CLI entrypoint, logging) | Makes the server runnable for any client | Basic project scaffold, uv/pyproject setup | Medium | High: prerequisite for everything else |
| SQLite persistence with seeding (connection mgmt, migrations, WAL) | Durable storage for tools/functions | SQLAlchemy setup, seed import utilities | Medium | High: unblock tool/function features |
| jq execution engine & tool runner | Core querying capability | jq binary availability, subprocess/timeouts | Medium-High | High: immediately delivers end-user value |
| Function & tool management APIs | Turn stored definitions into callable MCP tools | Persistence layer, jq compiler | Medium-High | High: completes baseline feature set |
| Basic team features (user lookup, attribution, usage logging) | Minimal collaboration insight without complex workflows | Auth header parsing, DB models | Medium | Medium: add after core querying works |
| Dataset provider abstraction (local JSON loader) | Supplies data to queries | Filesystem access, caching | Medium | Medium: required for production data flows |
| S3 synchronization & distributed locking | Keeps multi-instance deployments consistent | boto3, locking semantics, recovery | High | Medium-Low: valuable for teams, but complex |
| Schema/version management for datasets | Detect schema drift, ensure compatibility | JSON Schema tooling, migration logic | High | Low: deprioritize per user guidance |
| Approval workflow (voting, notifications) | Governs learned tools before rollout | User mgmt, async messaging, policy engine | High | Low: defer until base collaboration is stable |
| Learning/self-training system | Auto-proposes and validates new tools | Approval workflow, metrics, advanced analytics | Very High | Low: optional future iteration |
| Advanced performance features (websockets, caching layers, optimization) | Scalability and UX polish | Stable core services, observability | High | Optional: schedule after core adoption |

## Complexity Reduction Opportunities
- **Narrow the initial transport scope**: Start with HTTP-only (or stdio-only for local dev) instead of dual-mode support; add the second transport after the base server proves stable.
- **Simplify jq compilation requirements**: Require tools to declare fully inlined function bodies during early phases, postponing topological dependency sorting until the library of shared functions grows.
- **Defer automated schema/version tooling**: Store schema metadata as static YAML and skip auto-generation + migration logic until real datasets demand it; this avoids three subsystems (schema generator, migrator, compatibility matrix) while preserving dataset documentation.
- **Replace continuous S3 sync with manual checkpoints**: Ship a `sync` command that fetches/pushes the SQLite file on demand before investing in a background locker/merger; teams still share state, but operational risk drops sharply.
- **Keep authentication header-only**: Accept `X-User-Id` and optional `X-Team-Id` without JWT interpretation or role hierarchies; this retains attribution while avoiding token parsing and role-based gating.
- **Treat approvals as offline policy initially**: Persist status fields but skip vote counting, notifications, and automatic timers; teams can coordinate externally yet keep database schema ready for a richer workflow later.
- **Limit learning system scope**: Gate self-training behind a feature flag that defaults off; focus on manual tool creation until review bandwidth exists, avoiding async validation and metric loops.
- **Prefer synchronous persistence APIs at first**: Use standard SQLAlchemy sessions (non-async) with a lightweight worker pool to dodge asyncio complexity; switch to async only if profiling shows blocking I/O.

## Additional Observations
- README.md and SPECIFICATION.md both mandate SQLite as the single source of truth—any simplification should preserve that contract even if sync/versioning features are delayed.
- Many later-phase features (websockets, alternate query engines, Redis caching) assume a solid event loop and async stack; keeping the early implementation synchronous simplifies future refactors by making dependencies explicit.
- Roadmap checkboxes in README.md indicate Phase 1 foundation is already “done”; reconcile that status with actual code progress to reset expectations before prioritizing S3 sync or approvals.
