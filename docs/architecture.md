# Architecture

## System boundary

Hermes Apple Messages is a standalone, opt-in Hermes Agent plugin for macOS. Hermes owns model interaction, tool dispatch, approvals, and conversation reasoning. The plugin owns deterministic access to the local Messages database and delivery through Messages.app.

The plugin registers one `apple_messages` toolset containing:

- `messages_check_access`
- `messages_list_chats`
- `messages_search`
- `messages_read_chat`
- `messages_send`

The tool handlers return JSON strings for success and failure. Domain services return typed records and domain-specific failures before the handler boundary serializes them.

## Hermes plugin contract

The repository root contains `plugin.yaml` and `__init__.py`. Hermes clones the repository into the active profile's plugin directory, reads the root manifest, imports the root module, and calls `register(ctx)`.

Registration uses:

- `ctx.register_tool(...)` for the five public tools.
- `ctx.register_hook("pre_tool_call", ...)` for outbound-message approval.
- A plugin-local `check_fn` that exposes the tools on compatible macOS systems and lets `messages_check_access` explain permission state.

The manifest declares `kind: standalone`, the five tool names, the approval hook, MIT licensing, the project homepage, and discoverability tags. Installation and enablement use the profile-aware Hermes commands:

```text
hermes plugins install zfifteen/hermes-apple-messages --enable
```

A Hermes restart loads newly enabled plugin code.

### Approval invariant

Every `messages_send` invocation returns an `approve` directive from the plugin's `pre_tool_call` hook. The directive uses a freshly generated rule key for that one invocation. Session and permanent approval choices therefore apply to the presented send only. A denial, timeout, absent human surface, or approval-system error stops delivery before the handler executes.

The approval description identifies the exact target and includes a bounded text preview. It keeps full message content out of logs while giving the operator enough context to decide.

## Read architecture

### Database location and connection

The default database is:

```text
~/Library/Messages/chat.db
```

A plugin-relative setting may provide another database path for compatibility testing and fixtures.

The database layer:

1. Expands and resolves the configured path.
2. Opens SQLite with URI `mode=ro`.
3. Enables `PRAGMA query_only = ON`.
4. Sets a short busy timeout.
5. Uses short-lived connections for bounded operations.
6. Reads schema metadata before selecting a query variant.

`immutable=1` stays outside the live database URI because Messages may have current WAL state that belongs in a coherent read.

The plugin never executes write statements or mutation pragmas against Messages data.

### Schema capability detection

The schema detector records available tables and columns for the narrow set used by the plugin:

- `message`
- `chat`
- `handle`
- `chat_message_join`
- `chat_handle_join`
- `attachment`
- `message_attachment_join`

Queries select columns through explicit capability branches. A missing required table or join produces a compatibility failure that names the missing capability and requests a schema-only compatibility report.

### Query boundaries

All values enter SQL through parameters. Sort direction and selected column expressions come from internal constants. Public limits use conservative minimums and maximums. Search supports optional exact chat, exact sender, and date scopes.

Tool results expose stable chat and message identifiers, timestamps, direction, service, sender metadata, text, and bounded attachment metadata. Diagnostic and log output contains schema names, permission state, counts, and error categories while omitting message bodies and participant handles.

## Message-body decoding

### Plain text

`message.text` is the preferred body whenever it contains a string.

### Modern attributed bodies

Modern Messages versions may place text in `message.attributedBody` as an archived `NSAttributedString`. The selected decoder uses macOS Foundation through JavaScript for Automation:

1. Python base64-encodes only the bounded blobs that require decoding.
2. Python sends one JSON batch to `/usr/bin/osascript -l JavaScript` through stdin.
3. The bundled JXA source reads stdin through `NSFileHandle.fileHandleWithStandardInput`.
4. Foundation decodes each archive through `NSUnarchiver`.
5. The adapter returns each decoded attributed string's `.string` value as JSON on stdout.
6. Python validates output cardinality and size before mapping bodies back to records.

This transport keeps archived bodies out of process arguments, shell source, and temporary files. It uses macOS frameworks already present on the supported platform and adds no Python runtime dependency or compiled helper.

A synthetic architecture probe on macOS 26.6.1 archived and recovered `synthetic message body` through this exact Foundation path. Automated fixtures will generate or store synthetic archives containing public test text only.

Each malformed or unsupported archive produces an item-level decoding failure. The surrounding query remains usable and marks the affected body as unavailable with a structured reason.

## Send architecture

### Target model

The send service accepts one exact target:

- A stable chat identifier returned by `messages_list_chats`, `messages_search`, or `messages_read_chat`.
- An exact participant handle for starting or addressing a direct conversation.

Target resolution returns one chat or participant. Zero matches produce a targeted remediation. Several matches produce an ambiguity error with bounded, non-sensitive candidate metadata. Display names serve discovery and presentation while stable identifiers authorize the final send.

### Messages.app adapter

The adapter invokes `/usr/bin/osascript -l JavaScript` with bundled, constant JXA source. A JSON payload containing the target and message text travels through stdin. User-controlled text never enters generated source code, command-line arguments, or shell interpolation.

The JXA program:

1. Reads and validates the JSON payload.
2. Connects to `Application("Messages")`.
3. Resolves the exact chat or participant.
4. Confirms a single target.
5. Calls the Messages `send` command.
6. Returns structured JSON describing the accepted target and operation.

Python supplies a bounded timeout, captures stdout and stderr, and maps Automation denial, missing account, missing target, ambiguity, Messages launch, and timeout failures to domain errors with remediation.

The installed Messages scripting dictionary on macOS 26.6.1 declares `send` for a participant or chat, along with readable chat and participant identifiers.

## Privacy and security

- Full Disk Access gates local history through macOS privacy controls.
- Automation permission gates Messages.app delivery.
- Hermes' human approval gate authorizes every outbound message.
- Stable identifiers carry authority; display names support discovery.
- SQL values remain parameterized.
- Message text travels to native helpers through stdin.
- Subprocess arguments, generated source, logs, fixtures, diagnostics, and exception traces remain free of personal message content.
- Read results and outbound text remain bounded.
- Database connections remain read-only and short-lived.
- Tool failures fail closed with explicit remediation.

## Testing strategy

### Unit tests

- Domain error and model serialization.
- Apple epoch conversion at known boundaries.
- Path resolution and compatibility errors.
- Schema capability branches.
- Query parameterization and limit enforcement.
- Plain and attributed body selection.
- Foundation adapter input/output validation with mocked subprocesses.
- Target ambiguity and exact-match behavior.
- Send adapter payload construction and failure mapping.
- Tool JSON contracts.
- Approval directive uniqueness and send-only scope.

### Integration tests

Synthetic SQLite databases model supported schema variants, plain text, attributed bodies, direct chats, group chats, attachments, edited messages, and empty values. A macOS-only Foundation test decodes synthetic attributed archives. Messages.app delivery tests use a fake subprocess boundary and exercise no live send.

### Hermes integration tests

An isolated temporary `HERMES_HOME` installs or discovers the local plugin, enables it, loads the plugin manager, and verifies all five tools plus the approval hook. The test leaves the operator's active profile unchanged.

### Live smoke tests

A doctor command reports actual Full Disk Access and Automation readiness. Operator-selected scopes drive live reads. One operator-specified target and message drive the final send smoke test through the real Hermes approval prompt.

## Compatibility policy

The first release targets:

- macOS with Messages.app and JXA/AppKit Foundation bridges.
- Hermes Agent's current native plugin API.
- Python 3.11 and newer versions supported by Hermes.

Schema and subprocess capability detection produce actionable compatibility reports. CI uses deterministic fixtures, while a manual macOS matrix records verified operating-system releases.

## References

- Hermes plugin guide: https://hermes-agent.nousresearch.com/docs/developer-guide/plugins
- Hermes plugin user guide: https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins
- Hermes hook reference: https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks
- Apple `NSAttributedString`: https://developer.apple.com/documentation/foundation/nsattributedstring
- Python typedstream reference implementation: https://github.com/dgelessus/python-typedstream
- iMessage exporter typedstream research: https://chrissardegna.com/blog/reverse-engineering-apples-typedstream-format/
