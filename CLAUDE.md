# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

### Local Development
```bash
cd src/app
pip install -r requirements.txt
# Set required environment variables (see Environment Variables section)
chainlit run app.py -w
```

### Databricks Deployment
```bash
# Validate configuration
databricks bundle validate --profile <PROFILE>

# Deploy the bundle (creates resources)
databricks bundle deploy --profile <PROFILE>

# Run the setup job first (creates Lakebase tables)
databricks bundle run setup_lakebase --profile <PROFILE>

# Then run the app
databricks bundle run --profile <PROFILE>
```

### Testing & Quality
```bash
# No specific test commands found - add tests to requirements if needed
# For debugging, enable:
export CHAINLIT_DEBUG=true
```

## Architecture Overview

This is a **Databricks App** built with **Chainlit** that provides an AI-powered Business Intelligence chat interface. The application integrates with:

- **Multi-Agent Supervisor (MAS)** for AI reasoning and query routing
- **Lakebase (PostgreSQL)** for persistent chat history and session management
- **Databricks Unity Catalog** for data security and permissions
- **Databricks Model Serving** endpoints for AI inference

### Key Components

**Core Application (`src/app/`)**
- `app.py` - Main Chainlit application entry point with chat handlers
- `routes.py` - Chat message processing and history management
- `config.py` - Pydantic settings with environment variable handling

**Authentication (`src/app/auth/`)**
- Supports two modes: OBO (On-Behalf-Of) tokens for Databricks Apps, and PAT (Personal Access Token) for local development
- `identity.py` - Core identity management and token handling
- `header.py` / `password_auth.py` - Authentication method implementations

**Data Layer (`src/app/data/`)**
- `lakebase.py` - Chainlit data layer implementation using PostgreSQL
- Handles chat history persistence and session management
- Uses service principal authentication with auto-refreshing database credentials

**Services (`src/app/services/`)**
- `mas_client.py` - HTTP client for MAS endpoints with dual transport (OpenAI SDK for PAT, raw SSE for OBO)
- `mas_normalizer.py` - Event stream normalization from MAS responses
- `renderer.py` - Chainlit-specific rendering for streaming responses and tool calls

**Configuration (`src/app/.chainlit/`)**
- `config.toml` - Chainlit UI settings, branding, and feature toggles
- `public/` - Static assets including logos and custom CSS

### Authentication Flows

| Context | MAS Auth | Transport | Lakebase Auth |
|---------|----------|-----------|---------------|
| Databricks App | OBO tokens | SSE to `/invocations` | Service Principal |
| Local Development | PAT | OpenAI Async client | Service Principal |

**Important**: OpenAI SDK with OBO tokens can cause 403 errors due to scope issues. The app bypasses this by using direct REST calls for OBO authentication.

## Environment Variables

### Required for Local Development
```bash
# Authentication
DATABRICKS_TOKEN=<your_pat_token>
DATABRICKS_HOST=<workspace_host>

# Database
PGHOST=<lakebase_host>
PGPORT=5432
PGUSER=<service_principal>
PGDATABASE=databricks_postgres
DATABASE_INSTANCE=<lakebase_instance_name>

# Model Serving
SERVING_ENDPOINT=<mas_endpoint_name>

# Auth Mode (choose one)
ENABLE_PASSWORD_AUTH=true  # for local dev
ENABLE_HEADER_AUTH=false   # for Databricks Apps
```

### Required for Databricks Apps
Set in `src/app/app.yaml`:
```yaml
ENABLE_HEADER_AUTH: true
ENABLE_PASSWORD_AUTH: false
DATABRICKS_HOST: ${workspace.host}
SERVING_ENDPOINT: ${resources.serving_endpoints.agent-endpoint.name}
DATABASE_INSTANCE: ${var.lakebase_instance_name}
# ... other variables
```

## Key Configuration Files

**`databricks.yml`** - Bundle configuration defining:
- Target environments (dev/staging/prod)
- Model serving endpoint references
- Lakebase instance settings
- App permissions and resources

**`src/app/config.py`** - Application settings including:
- Chat history limits (turns and character counts)
- Starter message configuration
- Database connection parameters
- Authentication mode selection

**`src/app/.chainlit/config.toml`** - UI configuration:
- Session timeouts and persistence
- File upload restrictions (CSV, Excel, JSON)
- Branding and custom CSS
- Feature toggles for editing, threading, etc.

## Development Patterns

### Chat Message Flow
1. User message → `on_message()` in `routes.py`
2. Build message history with token/turn limits via `_build_messages_with_history()`
3. Stream to MAS via `mas_client.stream_raw()`
4. Normalize events via `mas_normalizer.normalize()`
5. Render to Chainlit via `ChainlitStream` renderer

### Error Handling
- 401/403 errors typically indicate OBO token scope/expiry issues
- Database connection errors auto-refresh credentials via service principal
- MAS streaming errors are caught and displayed to users

### History Management
- Keeps earliest system message + last N turns (configurable)
- Enforces character budget to prevent token overflow
- Persists via Chainlit data layer to Lakebase

## Troubleshooting

**Authentication Issues**
- OBO scope errors: Check endpoint ACLs and token expiry
- Database auth failures: Verify service principal permissions

**Streaming Issues**
- Ensure OBO path uses `Accept: text/event-stream` header
- Payload must be `input=[...]` not `messages=[...]` for MAS

**Performance**
- Monitor chat history limits in `config.py`
- Check Lakebase connection pooling under load