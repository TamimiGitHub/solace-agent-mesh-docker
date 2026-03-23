# Solace Agent Mesh with Docker

## Table of Contents

- [Overview](#overview)
  - [What is Solace Agent Mesh?](#what-is-solace-agent-mesh)
  - [Architecture](#architecture)
- [Quick Start](#quick-start)
  - [Prerequisites](#prerequisites)
  - [Setup Instructions](#setup-instructions)
- [Docker Services](#docker-services)
- [Configuration Files](#configuration-files)
- [Usage Examples](#usage-examples)
- [Development](#development)
  - [Viewing Logs](#viewing-logs)
  - [Stopping Services](#stopping-services)
  - [Rebuilding](#rebuilding)
- [Environment Variables Reference](#environment-variables-reference)
- [Additional Resources](#additional-resources)

## Overview

This project provides a containerized deployment of **[Solace Agent Mesh](https://github.com/SolaceLabs/solace-agent-mesh)** - an AI agentic system that coordinates multiple specialized agents to handle complex tasks through intelligent orchestration and workflow management.

### What is Solace Agent Mesh?

Solace Agent Mesh is an enterprise-grade AI agent framework that:
- **Orchestrates AI Agents**: Coordinates multiple specialized agents to break down and execute complex tasks
- **Event-Driven Architecture**: Uses Solace Broker for reliable, scalable inter-agent communication
- **Web Gateway**: Provides a FastAPI-based HTTP/SSE interface for user interactions
- **Artifact Management**: Creates, manages, and transforms various data artifacts (documents, reports, files)
- **Extensible**: Supports custom tools, data analysis, and multi-agent workflows

### Architecture

Solace Agent Mesh uses an event-driven architecture where multiple gateways can trigger agents, agents can communicate with each other via the Agent-to-Agent (A2A) protocol, and external agents can be integrated through proxy components.

```
                        External Systems
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
  ┌──────────┐        ┌──────────┐        ┌──────────┐
  │ Web UI   │        │  Slack   │        │   REST   │
  │ Gateway  │        │ Gateway  │        │ Gateway  │
  │ (8000)   │        │          │        │          │
  └─────┬────┘        └─────┬────┘        └─────┬────┘
        │                   │                    │
        │      A2A Protocol over Solace Broker   │
        └───────────────────┼────────────────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │   Solace Broker       │
                │   (Event Mesh)        │
                └───────────┬───────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
  │Orchestrator │    │ Specialized │    │   A2A       │
  │   Agent     │◄──►│   Agents    │    │   Proxy     │
  │             │    │ (SQL, RAG,  │    │             │
  │             │    │  MongoDB)   │    │             │
  └─────────────┘    └─────────────┘    └──────┬──────┘
                                               │
                                               │ HTTPS
                                               │
                                               ▼
                                    ┌──────────────────┐
                                    │  External A2A    │
                                    │     Agents       │
                                    │ (Remote Services)│
                                    └──────────────────┘

Key Components:
- Gateways: Multiple entry points (WebUI, Slack, REST) for external systems
- Solace Broker: Central event mesh routing A2A protocol messages
- Orchestrator Agent: Coordinates complex multi-agent workflows
- Specialized Agents: Domain-specific agents that can delegate to each other
- A2A Proxy: Bridges external HTTPS-based agents into the mesh
```
## Quick Start

### Prerequisites

- **Docker Desktop** installed and running
- **Python 3.12+** for local setup
- **LLM Service Access** (e.g., LiteLLM endpoint with API key)

### Setup Instructions

#### 1. Create Docker Network

First, create the external Docker network that the containers will use:

```bash
docker network create sam-network
```

#### 2. Set Up Python Environment

Activate a Python virtual environment:

```bash
# On macOS/Linux
python3 -m venv .venv
source .venv/bin/activate

# On Windows
python3 -m venv .venv
.venv\Scripts\activate
```

#### 3. Install Dependencies

```bash
pip install -r requirements.txt
```

#### 4. Initialize Solace Agent Mesh

```bash
sam init --skip
```
This creates the necessary Solace Agent Mesh configuration structure.


#### 5. Configure Environment Variables

Create or update your `.env` file with the following required variables:

```bash
# LLM Service Configuration
LLM_SERVICE_ENDPOINT="https://lite-llm.mymaas.net"
LLM_SERVICE_API_KEY="<your_api_key_here>"
LLM_SERVICE_PLANNING_MODEL_NAME="openai/vertex-claude-4-5-sonnet"
LLM_SERVICE_GENERAL_MODEL_NAME="openai/vertex-claude-4-5-sonnet"

# Agent Namespace
NAMESPACE="demo/"

# Docker-specific settings
SOLACE_BROKER_URL="ws://solace-broker:8008"
FASTAPI_HOST="0.0.0.0"
```

> **Important Docker Configuration Notes:**
> - `SOLACE_BROKER_URL` must point to the container name `solace-broker` (not localhost)
> - `FASTAPI_HOST` must be `0.0.0.0` to expose the API outside the container and be accessible by the host
> - Container names and network settings are defined in `docker-compose.yaml`

#### 6. Start the Services

```bash
docker compose up
```

Or run in detached mode:

```bash
docker compose up -d
```

#### 7. Access the Web Interface

Once the containers are running, access the Solace Agent Mesh web interface at:

```
http://localhost:8000
```

## Docker Services

The `docker-compose.yaml` defines two services:

### 1. solace-broker
- **Image**: `solace/solace-pubsub-standard:latest`
- **Purpose**: Event broker for agent communication
- **Key Ports**:
  - `8008` - Web transport for messaging
  - `8080` - SEMP/Management console
  - `55554` - SMF messaging
  - `2222` - SSH CLI access

### 2. solace-agent-mesh
- **Image**: `solace-agent-mesh-enterprise:1.95.2`
- **Purpose**: Runs the Solace Agent Mesh gateway and orchestrator agent
- **Port**: `8000` - Web UI and API
- **Dependencies**: Waits for `solace-broker` to be ready
- **Volumes**:
  - `./configs` - Agent and gateway configurations
  - `./src` - Custom source code
  - `./.env` - Environment variables
  - `./artifacts` - Generated artifacts



### This Repository Deploys

```
┌─────────────────────────────────────────────────────┐
│         Docker Compose Deployment                   │
│                                                     │
│  ┌────────────────────────────────────────────────┐ │
│  │  Container: solace-broker                      │ │
│  │  Image: solace/solace-pubsub-standard:latest   │ │
│  │  Ports: 8008, 8080, 55554, 2222                │ │
│  └───────────────────┬────────────────────────────┘ │
│                      │                              │
│                      │ A2A Protocol                 │
│                      │                              │
│  ┌───────────────────▼────────────────────────────┐ │
│  │  Container: sam-ent                            │ │
│  │  Image: solace-agent-mesh:latest               │ │
│  │  Port: 8000                                    │ │
│  │                                                │ │
│  │  Components:                                   │ │
│  │  ├─ Web UI Gateway (port 8000)                 │ │
│  │  └─ Orchestrator Agent                         │ │
│  └────────────────────────────────────────────────┘ │
│                                                     │
│  Access: http://localhost:8000                      │
└─────────────────────────────────────────────────────┘
```

## Configuration Files

All configuration files are located in the `configs/` directory. By default, `sam init` generates configurations for the Orchestrator Agent and Web UI Gateway. This directory contains all Solace Agent Mesh components organized by type:

### Directory Structure

- **`configs/agents/`** - Agent configurations
  - `main_orchestrator.yaml` - Orchestrator Agent that analyzes tasks, delegates to specialized agents, coordinates multi-agent workflows, and manages artifacts
  
- **`configs/gateways/`** - Gateway configurations
  - `webui.yaml` - Web UI Gateway with HTTP/SSE API endpoints, session management, authentication, speech features, and task logging
  
- **`configs/services/`** - Service configurations
  - `platform.yaml` - Platform service configuration

- **`configs/shared_config.yaml`** - Shared configurations used across all components
- **`configs/logging_config.yaml`** - Logging settings for all Solace Agent Mesh components

## Development

### Viewing Logs

```bash
# View all logs
docker compose logs -f

# View specific service logs
docker compose logs -f solace-agent-mesh
docker compose logs -f solace-broker
```

### Stopping Services

```bash
# Stop containers
docker compose down

# Stop and remove volumes
docker compose down -v
```

### Rebuilding

If you make changes to configurations:

```bash
docker compose restart
```

## Environment Variables Reference

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `LLM_SERVICE_ENDPOINT` | LLM service URL | - | Yes |
| `LLM_SERVICE_API_KEY` | API key for LLM service | - | Yes |
| `LLM_SERVICE_PLANNING_MODEL_NAME` | Model for planning tasks | - | Yes |
| `LLM_SERVICE_GENERAL_MODEL_NAME` | Model for general tasks | - | Yes |
| `NAMESPACE` | Agent namespace prefix | `demo/` | Yes |
| `SOLACE_BROKER_URL` | Solace broker WebSocket URL | `ws://solace-broker:8008` | Yes |
| `FASTAPI_HOST` | FastAPI bind address | `0.0.0.0` | Yes |
| `FASTAPI_PORT` | FastAPI port | `8000` | No |
| `WEBUI_GATEWAY_ID` | Gateway identifier | auto-generated | No |

## Additional Resources

- **[Solace Agent Mesh Documentation](https://solacelabs.github.io/solace-agent-mesh/docs/documentation/getting-started)** - Official getting started guide
- **[Solace Agent Mesh GitHub Repository](https://github.com/SolaceLabs/solace-agent-mesh)** - Source code and examples
- [Solace Broker Documentation](https://docs.solace.com/Cloud/cloud.htm) - Event broker documentation
- [Docker Compose Documentation](https://docs.docker.com/compose/) - Container orchestration guide
