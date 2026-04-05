# :material-book-open-variant: Tutorials & Documentation

!!! abstract "In-depth technical documentation from across my projects — architecture decisions, implementation guides, and deep dives."

=== ":material-dumbbell: Gorilla Coach"

    | Tutorial | Description |
    |----------|-------------|
    | [Architecture](gorilla-coach/ARCHITECTURE.md) | Full ecosystem architecture — services, data flows, deployment topology |
    | [Rust Tutorial](gorilla-coach/RUST_TUTORIAL.md) | Building a production web app in Rust from first principles |
    | [Frontend (Dioxus)](gorilla-coach/FRONTEND_TUTORIAL.md) | Dioxus Wasm PWA frontend — components, signals, routing |
    | [Database](gorilla-coach/DATABASE_TUTORIAL.md) | Schema design to production queries with PostgreSQL |
    | [LLM Agents](gorilla-coach/LLM_AGENTS_TUTORIAL.md) | Building tool-calling AI agents in Rust with Claude |
    | [Garmin Auth](gorilla-coach/GARMIN_AUTH_TUTORIAL.md) | Reverse-engineering Garmin Connect OAuth/SSO authentication |
    | [Google Sheets](gorilla-coach/GOOGLE_SHEETS_TUTORIAL.md) | Service accounts, JWT auth, and two-way sync |
    | [WASM & PWA](gorilla-coach/WASM_PWA_DIOXUS_TUTORIAL.md) | WebAssembly, Progressive Web Apps, and Dioxus deep dive |
    | [Scaling](gorilla-coach/SCALING_TUTORIAL.md) | Scaling from 100 to 1,000,000 users — architecture evolution |
    | [Architecture Patterns](gorilla-coach/ARCHITECTURE_PATTERNS_TUTORIAL.md) | Monolith vs microservices vs modular monolith tradeoffs |
    | [Spec-Driven Dev](gorilla-coach/SDD_TUTORIAL.md) | Building software from a blueprint — spec-first workflow |
    | [Vault Architecture](gorilla-coach/VAULT_ARCHITECTURE.md) | Local ChaCha20 vs HashiCorp Vault — encryption at rest decisions |

=== ":material-robot: Gorilla MCP"

    | Doc | Description |
    |-----|-------------|
    | [Architecture](gorilla-mcp/architecture.md) | MCP server design, transport layers, and data flow |
    | [Tools Reference](gorilla-mcp/tools-reference.md) | All 14 MCP tools — parameters, responses, examples |
    | [Chatbot](gorilla-mcp/chatbot.md) | Chat UI, Claude CLI integration, and LLM gateway |
    | [Configuration](gorilla-mcp/configuration.md) | Environment variables and setup guide |
    | [Deployment](gorilla-mcp/deployment.md) | Docker + Tailscale deployment with sidecar networking |
    | [Development](gorilla-mcp/development.md) | Local development, testing, and contributing |

=== ":material-cellphone: Life Manager"

    | Doc | Description |
    |-----|-------------|
    | [Architecture](lifemanager/01-architecture.md) | System overview — modules, data flow, auth model |
    | [Dioxus Fullstack](lifemanager/02-dioxus-fullstack.md) | Dioxus fullstack patterns — server functions, hydration |
    | [Data Model](lifemanager/03-data-model.md) | SQLite schema, migrations, and query patterns |
    | [Components](lifemanager/04-components.md) | Reusable UI components — swipe, chips, modals |
    | [Security](lifemanager/05-security.md) | Tailscale auth, CSRF, CSP headers |
    | [OCR Pipeline](lifemanager/06-ocr-pipeline.md) | Tesseract OCR for Shopee screenshot parsing |
    | [Deployment](lifemanager/07-deployment.md) | Docker deployment with Tailscale networking |
    | [Developer Guide](lifemanager/08-developer-guide.md) | Contributing, local setup, and project conventions |
    | [Dioxus Guide](lifemanager/dioxus-guide.md) | Dioxus framework patterns and best practices |
    | [Tesseract OCR](lifemanager/tesseract-ocr.md) | OCR setup, training data, tuning for Traditional Chinese |
    | [PWA Deployment](lifemanager/pwa-deployment.md) | Service worker, caching strategies, offline support |

=== ":material-gamepad-variant: Retro Arcade"

    | Doc | Description |
    |-----|-------------|
    | [Architecture](retrogames/architecture.md) | Game engine design — ECS-like patterns for Canvas games |
    | [Game Engine Patterns](retrogames/game-engine-patterns.md) | Sprite systems, collision, particles, audio |
    | [Miyoo Porting](retrogames/miyoo-porting-guide.md) | Cross-compiling Rust/Macroquad for ARM handheld |
    | [Mobile Optimization](retrogames/mobile-optimization.md) | Touch joystick, viewport scaling, performance |
    | [Adding Games](retrogames/adding-games.md) | How to add new games to the arcade |
    | [Deployment](retrogames/deployment-guide.md) | Docker + Tailscale deployment with nginx |

=== ":material-eye: cc-watcher.nvim"

    | Doc | Description |
    |-----|-------------|
    | [Tutorial](cc-watcher/tutorial.md) | Hands-on guide — setup, sidebar, diffs, integrations |
    | [Lua Plugin Guide](cc-watcher/lua-plugin-guide.md) | Building Neovim plugins in Lua from scratch |

=== ":material-bell: tmux_cc_attention"

    | Doc | Description |
    |-----|-------------|
    | [Architecture](tmux-cc-attention/architecture.md) | Plugin design — hooks, state machine, cross-session awareness |
