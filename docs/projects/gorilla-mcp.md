# gorilla_mcp

MCP (Model Context Protocol) server and chat interface for strength training coaching. Bridges Claude AI to training data via gorilla_coach and Garmin biometrics via gapi.

## Architecture

```
                      ┌──────────────┐
                      │  Claude Code │
                      │  or Claude AI│
                      └──────┬───────┘
                             │
                        MCP Protocol
                     (stdio or HTTP/SSE)
                             │
                      ┌──────┴───────┐
                      │ gorilla_mcp  │
                      │ (MCP Server) │
                      └──┬────────┬──┘
                         │        │
               ┌─────────┘        └────────┐
               │                           │
        ┌──────┴──────┐            ┌───────┴──────┐
        │gorilla_coach│            │  garmin_api  │
        │    API      │            │   service    │
        └─────────────┘            └──────────────┘
        Training data              Garmin biometrics
        5/3/1 auto-reg             HRV, sleep, stress
```

## What It Does

Gives Claude access to your training and biometric data through:

- **14 tools** — query training logs, check readiness, log sets, analyze recovery
- **3 resources** — training plans, exercise library, user profiles
- **4 prompts** — SITREP, AAR, DEBRIEF, coaching conversation

## Binaries

| Binary | Purpose |
|--------|---------|
| **gorilla_mcp** | MCP server — tools, resources, prompts. Stdio or HTTP/SSE transport. |
| **gorilla_chatbot** | Web chat UI that spawns Claude CLI with MCP tools. Also serves as LLM gateway for gorilla_coach. |

## Stack

Rust (2024 edition), Axum, Claude AI via MCP protocol.

<div class="project-banner gorilla" style="margin-top: 3rem;">

<a href="https://github.com/elmomk/gorilla_mcp" class="md-button">View on GitHub</a>

</div>
