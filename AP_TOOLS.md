## Project Overview

This is a **development environment orchestration tool** for the HMPPS (Her Majesty's Prison and Probation Service) Approved Premises system. It's designed to simplify running multiple related applications locally for development purposes.

## What It Does

The project manages a **full local development stack** that includes:

### Core Services (Local Containers)
- **PostgreSQL** (v14 with PostGIS) - Database for the API
- **Redis** (v7.2.5) - Caching for both UI and API
- **Wiremock** - Proxies and optionally mocks external service calls
- **LocalStack** - Provides SQS (Simple Queue Service) locally

### Application Components
- **Approved Premises API** - Backend service (Java/Spring Boot)
- **Multiple UI Applications** for different CAS (Community Accommodation Service) variants:
  - **CAS1** - Approved Premises UI
  - **CAS2** - Community Accommodation Tier 2 UI  
  - **CAS2v2** - Community Accommodation Tier 2 Bail UI
  - **CAS3** - Temporary Accommodation UI

## Key Architecture Points

### External Service Integration
You're correct about the dev cluster integration! The project is designed to work with the **Cloud Platform's Dev environment** for upstream services like:
- HMPPS Auth
- Prisoner API
- Probation services
- Case notes
- And other justice system APIs

### Wiremock as a Proxy
All external service calls go through Wiremock, which:
- **Proxies** requests to the actual dev cluster by default
- Allows **selective mocking** of specific endpoints for testing
- Provides visibility into all external API calls

### Flexible Deployment Options
The system supports both:
- **Docker containers** (using pre-built images from Quay.io/GHCR)
- **Local development** (running code directly from your local repositories)

## Usage Patterns

### Quick Start
```bash
ap-tools start  # Interactive prompts for configuration
```

### Direct Configuration
```bash
ap-tools server start --cas1 --local-ui --local-api
```

### Access Points
- **UI**: http://localhost:3000
- **Tilt Console**: http://localhost:10350 (for monitoring startup)
- **Wiremock Admin**: http://localhost:9004/__admin/requests

## Development Workflow

1. **Setup**: Install dependencies, configure kubectl for cloud platform access
2. **Clone**: Local repositories for UI/API you want to develop
3. **Configure**: Set environment variables pointing to local code paths
4. **Start**: Use `ap-tools` to orchestrate the entire stack
5. **Develop**: Make changes to local code, see them reflected immediately
6. **Test**: Use Wiremock to mock external services as needed

## Security & Credentials

The system uses the same authentication as the dev environment, with a script (`sync-secrets.sh`) to manage secrets between Kubernetes and local development.

This is a sophisticated development environment that bridges local development with the actual justice system infrastructure, allowing developers to work on isolated components while still integrating with real external services when needed.





# Technical Breakdown: HMPPS Approved Premises Tools

## Architecture Overview

This is a **multi-layered orchestration system** that creates a complete local development environment for the HMPPS justice system applications. Here's how it works:

## Core Technologies & Tools

### 1. **Tilt** - The Orchestration Engine
- **Purpose**: Manages the entire development environment lifecycle
- **Role**: Acts as the central coordinator, handling both Docker containers and local development processes
- **Configuration**: Uses `tiltfile` to define resource dependencies and startup sequences
- **Features**: 
  - Health checks for services
  - Resource dependency management
  - Hot reloading for local development
  - Web UI at `http://localhost:10350` for monitoring

### 2. **Docker Compose** - Container Orchestration
- **Purpose**: Manages all containerized services
- **Services**:
  - **PostgreSQL** (v14 with PostGIS) - Persistent database
  - **Redis** (v7.2.5) - Caching layer
  - **Wiremock** - API proxy/mocking service
  - **LocalStack** - AWS SQS emulation
  - **Application containers** (API + UI variants)

### 3. **Wiremock** - API Gateway & Mocking Layer
- **Purpose**: Intercepts all external API calls
- **Architecture**:
  ```
  Local Apps → Wiremock → Dev Cluster APIs
  ```
- **Configuration**:
  - **Proxies** (priority 100): Forward requests to actual dev environment
  - **Overrides** (priority < 100): Mock specific endpoints for testing
- **External Services Proxied**:
  - Prison API
  - Probation services
  - Case notes
  - HMPPS Tier services
  - User management
  - And more...

## Detailed Workflow

### 1. **Startup Sequence** (`bin/start-server`)

```bash
# 1. Parse command line arguments
# 2. Resolve secrets from Kubernetes
# 3. Configure environment files
# 4. Launch Tilt with appropriate flags
# 5. Wait for health checks
```

#### Secret Resolution Process:
```bash
# For each service (API, CAS1-UI, CAS2-UI, etc.)
kubectl get secrets <secret-name> --namespace hmpps-community-accommodation-dev
# → Extract base64 decoded values
# → Substitute into .env.template files
# → Generate .env files for each service
```

### 2. **Environment Configuration**

The system uses a **template-based configuration** approach:

```
.env.api.template → .env.api (with secrets resolved)
.env.cas1-ui.template → .env.cas1-ui (with secrets resolved)
```

**Key Environment Variables**:
- Database connections
- Redis configuration
- External service URLs (all pointing to Wiremock)
- Authentication tokens
- Feature flags

### 3. **Service Dependencies & Health Checks**

```yaml
# Docker Compose Dependencies
api:
  depends_on: [postgres, redis, wiremock, localstack]

# Tilt Health Checks
readiness_probe:
  period_secs: 15
  http_get:
    port: 8080
    path: /health
```

### 4. **Local vs Docker Development**

The system supports **hybrid development**:

#### Docker Mode:
```yaml
# Uses pre-built images from Quay.io/GHCR
image: quay.io/hmpps/hmpps-approved-premises-api:latest
```

#### Local Mode:
```bash
# Runs local code with hot reloading
serve_cmd: "./gradlew bootRunDebug --stacktrace"  # API
serve_cmd: "npm run start:dev"                    # UI
```

## Network Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Local Apps    │    │    Wiremock     │    │   Dev Cluster   │
│                 │    │                 │    │                 │
│ API: localhost  │───▶│ localhost:9004  │───▶│ External APIs   │
│ UI: localhost   │    │                 │    │                 │
│ 3000            │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │
         │              ┌─────────────────┐
         └─────────────▶│   LocalStack    │
                        │ localhost:4566  │
                        │ (SQS Emulation) │
                        └─────────────────┘
```

## Data Flow

### 1. **Application Requests**
```
UI/API → Wiremock → Dev Cluster APIs
```

### 2. **Database Operations**
```
API → PostgreSQL (localhost:5431)
```

### 3. **Caching**
```
UI/API → Redis (localhost:6379)
```

### 4. **Message Queuing**
```
API → LocalStack SQS (localhost:4566)
```

## Security & Authentication

### 1. **Secret Management**
- **Source**: Kubernetes secrets in `hmpps-community-accommodation-dev` namespace
- **Resolution**: `resolve_secrets.sh` script extracts and substitutes values
- **Storage**: Local `.env` files (gitignored)

### 2. **Authentication Flow**
- Uses same credentials as dev environment
- HMPPS Auth service handles authentication
- Session management via Redis

## Development Workflow Integration

### 1. **Local Development Setup**
```bash
# Clone local repositories
export LOCAL_CAS_API_PATH=/path/to/api
export LOCAL_CAS1_UI_PATH=/path/to/ui

# Start with local code
ap-tools server start --cas1 --local-ui --local-api
```

### 2. **Hot Reloading**
- **API**: Gradle continuous build
- **UI**: Node.js development server
- **Tilt**: Monitors file changes and restarts services

### 3. **Testing & Mocking**
- Wiremock allows selective endpoint mocking
- Override files in `wiremock/mappings/overrides/`
- Priority system controls which responses are used

## Resource Management

### 1. **Container Lifecycle**
- **Start**: Docker Compose + Tilt orchestration
- **Stop**: `tilt down` + cleanup
- **Database Reset**: Volume removal option

### 2. **Port Management**
- **3000**: UI applications
- **8080**: API
- **5431**: PostgreSQL
- **6379**: Redis
- **9004**: Wiremock
- **4566**: LocalStack
- **10350**: Tilt UI

### 3. **Volume Management**
- **PostgreSQL**: Docker managed volume for performance
- **Wiremock**: Mapped volume for configuration
- **LocalStack**: Temporary directory mapping

## Error Handling & Resilience

### 1. **Health Checks**
- Service readiness probes
- Database connectivity checks
- API endpoint health monitoring

### 2. **Graceful Shutdown**
- Proper container cleanup
- Database volume management
- Process termination

### 3. **Dependency Resolution**
- Sequential startup based on dependencies
- Retry logic for external service connections
- Fallback mechanisms for failed services

This system essentially creates a **miniature production environment** locally, with all the complexity of a distributed system but running on a single developer's machine. It bridges the gap between local development and the actual justice system infrastructure, providing a realistic testing environment while maintaining developer productivity.
