![Hermes Apple Messages](docs/assets/hero.png)

# Hermes Apple Messages

Evaluate and strengthen Apple Messages integrations for Hermes Agent through compatibility testing, documented gaps, and upstream contributions.

## Vision

Hermes Apple Messages gives Hermes a focused set of tools for working with the conversations already available on your Mac:

- Search messages and conversations.
- Read recent context from direct and group chats.
- Summarize conversations and surface requested insights.
- Draft replies grounded in the selected conversation.
- Send messages after explicit user approval.
- Keep message access local to the Mac.

## Project status

Implementation is paused following an ecosystem review. Hermes already ships an official [`imessage` skill](https://hermes-agent.nousresearch.com/docs/user-guide/skills/bundled/apple/apple-imessage), BlueBubbles support, and Photon support, while community projects provide several local `imsg` platform plugins and a matching local MCP design.

This repository remains an evaluation workspace while we test the official path and leading community integrations. Verified gaps will become upstream contributions to the strongest existing project. This approach concentrates maintenance, review, and discoverability where the community already gathers.

Current references:

- [Official Hermes iMessage skill](https://hermes-agent.nousresearch.com/docs/user-guide/skills/bundled/apple/apple-imessage)
- [promptclickrun/hermes-imsg-cli](https://github.com/promptclickrun/hermes-imsg-cli)
- [MaudeCode/hermes-imsg-platform](https://github.com/MaudeCode/hermes-imsg-platform)
- [jeffhuber/hermes-imessage-plugin](https://github.com/jeffhuber/hermes-imessage-plugin)
- [BlueBubbles setup](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/bluebubbles)
- [Photon setup](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/photon)

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
