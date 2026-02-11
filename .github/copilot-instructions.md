# 3x-ui AI Coding Agent Instructions

## Project Overview

3x-ui is an advanced web panel for managing Xray proxy servers. It's a Go backend with an embedded web UI that provides server management, inbound/outbound configuration, user management, traffic monitoring, and Telegram bot integration.

## Architecture & Key Components

### Core Structure
- **Main entry point**: [main.go](../main.go) runs two servers: `web.Server` (panel UI) and `sub.Server` (subscription endpoint)
- **Database**: SQLite via GORM at `/etc/x-ui/x-ui.db` (configured via `XUI_DB_FOLDER`)
- **Embedded assets**: Web HTML, CSS/JS, and translation files are embedded at compile time using Go's `//go:embed`
- **Language support**: 12 languages via TOML translation files in [web/translation/](../web/translation/)

### Main Servers

**Web Server** ([web/web.go](../web/web.go)):
- Gin HTTP framework with SSL/TLS support (auto-HTTPS via ACME/Let's Encrypt)
- Session management via gorilla/sessions with cookie store
- Gzip compression middleware
- Routes: auth, inbounds, users, settings, xray management, backup/restore
- Scheduled jobs using robfig/cron (traffic stats, client IP validation, xray health checks)

**Subscription Server** ([sub/sub.go](../sub/sub.go)):
- Lightweight HTTP server serving subscription links to clients
- Domain validation middleware (optional subdomain restriction)
- JSON format support for subscriptions
- Generated from [sub/subJsonService.go](../sub/subJsonService.go)

### Service Layer

Services in [web/service/](../web/service/) follow a pattern of wrapping GORM models and the Xray API:
- **InboundService**: Manages proxy inbound configurations (ports, protocols, settings)
- **XrayService**: Wraps Xray API (start/stop/reload processes, config management)
- **UserService**: Admin user authentication and CRUD
- **SettingService**: Panel settings persistence
- **Tgbot**: Telegram bot notifications for traffic reports

### Data Models

Key models in [database/model/model.go](../database/model/model.go):
- `User`: Admin panel login (bcrypt password hashing)
- `Inbound`: Xray proxy configurations with embedded client settings
- `ClientTraffic` ([xray/client_traffic.go](../xray/client_traffic.go)): Per-client traffic tracking (up/down bytes, expiry time)
- `Setting`: Key-value pairs for panel configuration
- `InboundClientIps`: IP-to-email mapping for client identification

## Developer Workflows

### Build & Run Locally

```bash
# Build binary
go build -o build/x-ui main.go

# Run with debug logging
XUI_DEBUG=true XUI_LOG_LEVEL=debug ./build/x-ui

# Environment variables (see config/config.go):
# - XUI_BIN_FOLDER: Binary directory (default: "bin")
# - XUI_DB_FOLDER: Database directory (default: "/etc/x-ui")
# - XUI_LOG_LEVEL: debug|info|notice|warn|error
```

### Docker Build

```bash
docker build -t 3x-ui:latest .
docker run --rm -v /etc/x-ui:/etc/x-ui -p 54321:54321 3x-ui
```

The Dockerfile uses a multi-stage build; xray binary is downloaded during `DockerInit.sh`.

### Database Initialization

Auto-migration happens in [database/db.go](../database/db.go#L30) for all models. Default admin:
- Username: `admin` / Password: `admin` (bcrypt hashed)

Seeders run once to track schema version history in `history_of_seeders` table.

## Code Patterns & Conventions

### Error Handling
- Services return `(result, error)` tuples; callers MUST check errors
- Prefer specific errors; see [util/common/err.go](../util/common/err.go) for common error types
- Controllers translate service errors to HTTP responses

### Controllers vs Services
- **Controllers** ([web/controller/](../web/controller/)): Parse requests, call services, return JSON
- **Services**: Business logic; testable and reusable across controllers
- Authentication: `controller.GetETag()` extracts admin token from session

### JSON Configuration
- Inbound protocol settings stored as JSON strings in `Inbound.Settings`
- Parse with `json.Unmarshal()` into `map[string][]Client` (see [web/service/inbound.go#L70](../web/service/inbound.go#L70))
- Always validate structure after unmarshaling

### Xray Integration
- [xray/](../xray/) provides low-level API wrappers (config, process, traffic)
- Services call xray via `XrayService` (don't use xray package directly)
- Traffic data flows: Xray → `ClientTraffic` table → StatsNotifyJob → Telegram notifications

### Job Scheduling
- Jobs in [web/job/](../web/job/) implement the Job interface (Run method)
- Registered with robfig/cron in [web/web.go#L200+](../web/web.go)
- Examples: traffic stats (30s), xray health check (60s), client IP tracking

## Critical Patterns to Follow

### When Adding Routes
1. Add handler in appropriate controller file
2. Register route in [web/controller/index.go](../web/controller/index.go)
3. Return JSON: `{Code: 0, Msg: "success", Obj: data}` for success (see [web/entity/entity.go](../web/entity/entity.go))

### When Modifying Models
1. Update model struct in [database/model/model.go](../database/model/model.go)
2. GORM auto-migration handles schema changes
3. Add seeder in [database/db.go](../database/db.go) if default data needed

### When Adding Translation Keys
1. Add key to all TOML files in [web/translation/](../web/translation/)
2. Access via `locale.Translate(key)` in controllers
3. Sync at least English and Russian translations

### Xray Configuration
- Stored in `Inbound` table as serialized protocol config
- Modified inbounds require XrayService.UpdateInbound() → Xray process reload
- Traffic metrics updated per-client via gRPC streaming from Xray stats API

## Dependencies to Know
- **Gin**: Web framework and routing
- **GORM**: ORM for SQLite
- **go-logging**: Structured logging
- **Xray Core**: External process managed by [xray/process.go](../xray/process.go)
- **Telego**: Telegram bot API client
- **Cron**: Scheduled job runner

## Testing Considerations
- Services are mostly I/O bound (database, Xray API calls)
- Mock database in tests; use GORM's scoped DB instances
- Xray process management tests require actual binary in PATH
- Frontend tests handled separately (Vue.js in [web/html/](../web/html/))
