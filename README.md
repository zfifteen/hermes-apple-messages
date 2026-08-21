![Hermes Apple Messages](docs/assets/hero.png)

# Hermes Apple Messages

Bring Apple Messages into Hermes Agent with local-first conversation search, reading, analysis, reply drafting, and approval-gated sending on macOS.

## Vision

Hermes Apple Messages gives Hermes a focused set of tools for working with the conversations already available on your Mac:

- Search messages and conversations.
- Read recent context from direct and group chats.
- Summarize conversations and surface requested insights.
- Draft replies grounded in the selected conversation.
- Send messages after explicit user approval.
- Keep message access local to the Mac.

## Project status

The project is entering its design and implementation phase. The first release will focus on a secure, testable Python integration for Hermes Agent on macOS.

## Safety principles

- Read access uses the local Apple Messages database with the narrowest practical permissions.
- Sending requires explicit user approval and precise recipient resolution.
- Tool responses expose only the message content requested for the active task.
- Logs and diagnostics keep private conversation content out of persistent output.

## Platform

- macOS
- Hermes Agent
- Python 3.11+

## License

MIT
