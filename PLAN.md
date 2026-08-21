# Hermes Apple Messages — Implementation Plan

## Goal

Build a community-installable Hermes Agent plugin for macOS that exposes local Apple Messages conversations through focused, privacy-conscious tools. The first release will support access diagnostics, chat discovery, message search, conversation reading, and approval-gated sending.

## Acceptance criteria

1. `hermes plugins install zfifteen/hermes-apple-messages --enable` installs the plugin from GitHub.
2. The plugin registers an `apple_messages` toolset in Hermes CLI, Desktop, and gateway-backed sessions running on the Mac.
3. Read operations open `~/Library/Messages/chat.db` in SQLite read-only/query-only mode.
4. Read operations support modern `text` and `attributedBody` storage through a verified decoding strategy.
5. Every send enters Hermes' human approval gate through `pre_tool_call` before Messages.app receives the command.
6. Each send receives a unique approval rule key, preserving approval for every individual outbound message.
7. Recipient selection uses an exact chat identifier or an exact resolved participant handle and returns an ambiguity error whenever several targets match.
8. Tool results use bounded limits, structured JSON, stable identifiers, and explicit error remediation.
9. Logs, exceptions, fixtures, and diagnostics remain free of real message content and contact details.
10. Unit, integration, type, lint, packaging, and plugin-discovery checks pass locally and in CI.

## Proposed public tools

| Tool | Purpose | Main safety boundary |
|---|---|---|
| `messages_check_access` | Report macOS, database, and Messages automation readiness | Returns capability status and remediation only |
| `messages_list_chats` | List recent direct and group chats with stable chat identifiers | Bounded results and optional participant metadata |
| `messages_search` | Search message text within optional chat, sender, and date scopes | Read-only query, bounded result count |
| `messages_read_chat` | Read a bounded recent window from one exact chat | Requires the stable chat identifier |
| `messages_send` | Send text to one exact chat or exact participant | Unique human approval gate for every call |

Hermes itself will summarize, analyze, and draft replies from the structured read results. The plugin will keep model-facing tools focused on deterministic message access and delivery.

## Planned repository structure

- `plugin.yaml` — Hermes manifest and provided-tool declaration.
- `__init__.py` — plugin registration and send-approval hook.
- `pyproject.toml` — Python, Ruff, mypy, pytest, and packaging configuration.
- `hermes_apple_messages/errors.py` — domain-specific failures and remediation details.
- `hermes_apple_messages/models.py` — immutable chat, participant, message, and capability records.
- `hermes_apple_messages/apple_time.py` — Apple epoch conversion.
- `hermes_apple_messages/attributed_body.py` — modern message-body decoding adapter.
- `hermes_apple_messages/database.py` — read-only SQLite connection and schema capability detection.
- `hermes_apple_messages/queries.py` — bounded, parameterized chat and message queries.
- `hermes_apple_messages/messages_app.py` — exact target resolution and Messages.app send adapter.
- `hermes_apple_messages/service.py` — application-level orchestration.
- `hermes_apple_messages/schemas.py` — model-facing JSON schemas.
- `hermes_apple_messages/tools.py` — JSON-returning Hermes handlers.
- `tests/fixtures/` — synthetic Messages-compatible SQLite fixtures.
- `tests/unit/` and `tests/integration/` — deterministic automated coverage.
- `scripts/doctor.py` — local permission and compatibility diagnostics.
- `.github/workflows/ci.yml` — macOS-focused CI quality gates.
- `docs/` — installation, permissions, privacy, architecture, and troubleshooting guidance.

## Execution procedure

### 1. Architecture validation

**Work**

- Validate the standalone plugin install layout against the current Hermes plugin loader.
- Inspect the local Messages scripting dictionary for chat and participant send contracts.
- Evaluate modern `attributedBody` decoding approaches using synthetic data and public schema references.
- Select the smallest reliable decoding dependency or native adapter.
- Record the database, decoding, recipient-resolution, and approval invariants in `docs/architecture.md`.

**Verification**

- A minimal discovery probe loads the manifest from a temporary Hermes home.
- A decoding spike demonstrates plain text and attributed-body behavior without reading personal message rows.
- The architecture document maps every privileged action to its macOS and Hermes permission gate.

**Operator review point**

- Review the chosen body-decoding strategy if it introduces a runtime dependency or compiled helper.

### 2. Phase 1 — scaffolding

**Work**

- Create every planned module with complete signatures, types, docstrings, invariants, control-flow comments, and explicit failure contracts.
- Create test modules with descriptive test names and fixture contracts.
- Keep function bodies structural and free of executable implementation logic.

**Verification**

- Python compilation confirms a syntactically valid skeleton.
- A manual skeleton review confirms tool boundaries, data flow, and ownership.

**Commit**

- Commit the reviewed structural scaffold as one design increment.

### 3. Phase 2 — skeleton review

**Work**

- Review the complete skeleton for responsibility boundaries, query safety, permission flow, recipient ambiguity, approval enforcement, concurrency, schema variation, and privacy.
- Revise signatures and comments before implementation.
- Record the review in `docs/design-review.md`.

**Verification**

- Every acceptance criterion maps to one or more signatures and named tests.
- Every privileged operation has a fail-closed path.

**Commit**

- Commit the revised skeleton and design review.

### 4. Phase 3 — incremental implementation

Each increment implements exactly one function or method, adds its immediate tests, runs the focused quality gates, and creates a granular conventional commit before the next implementation begins.

**Implementation order**

1. Domain error serialization.
2. Immutable model serialization.
3. Apple timestamp conversion.
4. Database-path resolution.
5. Read-only SQLite connection setup.
6. Schema capability detection.
7. Chat listing query.
8. Message search query.
9. Conversation-window query.
10. Plain-text extraction.
11. Attributed-body extraction.
12. Participant and chat target resolution.
13. Messages.app script construction with argument-safe transport.
14. Messages.app send execution.
15. Capability diagnostic service.
16. Chat listing service.
17. Search service.
18. Conversation reading service.
19. Sending service.
20. JSON tool handlers.
21. Tool schemas.
22. Hermes registration.
23. Per-send approval hook with a unique rule key.
24. Doctor command.
25. Installation and live smoke-test harness.

**Immediate verification per increment**

- Focused pytest selection for the implemented unit.
- Ruff formatting and linting for touched files.
- mypy strict checking for touched modules.
- A commit containing the implementation, tests, and updated comments.

### 5. Local integration and permissions

**Work**

- Install the plugin into an isolated Hermes profile first.
- Run the doctor command before requesting macOS permissions.
- Guide the operator through Full Disk Access for message history and Automation access for Messages.app.
- Restart Hermes after plugin enablement.
- Load the `apple_messages` toolset and exercise diagnostics, listing, search, and reading with explicit operator-chosen scopes.
- Exercise one live send only after the operator supplies the exact recipient/chat, exact message text, and approves the Hermes send prompt.

**Verification**

- Hermes lists all five tools.
- The access diagnostic reports the actual permission state.
- Read results match the operator-selected chat and bounds.
- The approved test message appears in the exact target conversation.

**Operator review point**

- Grant macOS permissions.
- Choose the live read scope.
- Choose and approve the live send target and message.

### 6. Documentation, CI, and community release readiness

**Work**

- Expand the README with installation, enablement, permission, usage, privacy, and uninstall guidance.
- Add architecture, troubleshooting, security, contribution, and release documentation.
- Add a macOS CI workflow for Python versions supported by Hermes.
- Add issue templates for compatibility reports that request schema metadata while excluding message content.

**Verification**

- Follow the installation guide from a temporary Hermes home.
- Run link, packaging, lint, type, and test checks.
- Confirm repository metadata and topics remain accurate.

### 7. Phase 4 — full self-review

Apply every item in the canonical code-review checklist:

- Conversational prose style in names and control flow.
- Structure and design alignment.
- Test coverage for happy paths and important failures.
- Empty, malformed, ambiguous, permission-denied, and schema-variant cases.
- Logical fidelity to the reviewed skeleton.
- Zero Ruff, mypy, formatting, or packaging findings.
- Bounded query and resource behavior.
- SQL parameterization, AppleScript argument safety, approval enforcement, and privacy.
- Accurate docs and comments.
- Full adherence to the four-phase authoring procedure.

Record the final checklist and fixes in `docs/code-review.md`, run the complete verification suite, and publish the verified release candidate to the existing GitHub repository.

## Commands and quality gates

The exact environment commands will follow the selected dependency manager. The target top-level interface is:

```text
make format
make lint
make typecheck
make test
make integration
make verify
```

Hermes integration checks will use an isolated temporary `HERMES_HOME` and the installed local Hermes source. Live permission checks will run only on macOS and will remain separate from deterministic CI.

## Risks and controls

| Risk | Control |
|---|---|
| Messages schema varies by macOS release | Capability detection, synthetic schema fixtures, explicit compatibility errors |
| Modern bodies live in `attributedBody` | Architecture spike followed by fixture-backed decoder tests |
| SQLite WAL changes during reads | Read-only connections, short bounded queries, busy timeout, no mutation pragmas |
| Recipient names match several chats or handles | Stable identifiers and ambiguity errors |
| AppleScript receives untrusted message text | Pass values as process arguments and keep content out of generated script source |
| Automated sending bypasses intent | `pre_tool_call` human approval with a unique rule key on every send |
| Permission prompts differ by launch surface | Doctor command and surface-specific setup docs |
| Personal content enters logs or fixtures | Synthetic fixtures, metadata-only diagnostics, content-free logging |
| Plugin changes outpace Hermes APIs | CI against supported Hermes releases plus a compatibility module |

## Current discovery results

- Hermes' current plugin API supports custom tools through `ctx.register_tool(...)`.
- Hermes' `pre_tool_call` hook supports `{"action": "approve", ...}` and routes the call through the host-owned human approval gate before execution.
- Messages.app's local scripting dictionary exposes `send` for chats and participants.
- The local `chat.db` schema probe currently receives macOS `authorization denied`, confirming Full Disk Access as an installation prerequisite.

## Approval requested

Approval of this plan authorizes architecture validation and the four-phase implementation procedure. Live personal-message reads and the live send test retain their dedicated operator review points.

## Approval and execution log

- **Approved by the operator:** 2026-08-20
- **Architecture validation:** In progress
- Confirmed the current Hermes root-manifest plugin layout and human-approval hook contract from the installed Hermes source.
- Confirmed the local Messages scripting dictionary exposes delivery to chats and participants.
- Confirmed macOS privacy currently denies the live schema-only database probe, establishing Full Disk Access as a live-test prerequisite.
- Validated dependency-free Foundation decoding with a synthetic `NSAttributedString` and a batched stdin-only JXA transport.
