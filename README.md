# MCP Curl Server and Agent with kagent

This guide explains how to deploy the MCP Curl Server on Kubernetes using kmcp, and create an AI agent that can make HTTP requests to any URL.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Part 1: Deploy the MCP Curl Server](#part-1-deploy-the-mcp-curl-server)
- [Part 2: Deploy the Curl Agent](#part-2-deploy-the-curl-agent)
- [Part 3: Testing the Agent](#part-3-testing-the-agent)
- [Use Cases](#use-cases)

---

## Overview

The MCP Curl Server provides AI agents with the ability to make HTTP requests to any URL using a curl-like interface. This enables agents to interact with web services, APIs, and fetch data from the internet.

### Capabilities

The `curl` tool accepts the following parameters:

| Parameter | Required | Description |
|-----------|----------|-------------|
| `url` | Yes | The URL to make the request to |
| `method` | No | HTTP method (GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS). Default: GET |
| `headers` | No | Object containing HTTP headers |
| `body` | No | Request body for POST/PUT/PATCH requests |
| `timeout` | No | Request timeout in milliseconds (default: 30000, max: 300000) |

### Response Format

The tool returns structured responses with:

```json
{
  "status": 200,
  "statusText": "OK",
  "headers": {
    "content-type": "application/json",
    "server": "nginx"
  },
  "body": "{\"result\": \"success\"}"
}
```

### Architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  kagent Agent   │────▶│  MCP Curl        │────▶│  External       │
│  (HTTP Client)  │     │  Server          │     │  APIs/Services  │
└─────────────────┘     └──────────────────┘     └─────────────────┘
```

---

## Prerequisites

- Kubernetes cluster with kagent and kmcp installed
- `kubectl` configured to access your cluster

---

## Part 1: Deploy the MCP Curl Server

```bash
kubectl apply -f- <<EOF
apiVersion: kagent.dev/v1alpha1
kind: MCPServer
metadata:
  name: mcp-curl
  namespace: kagent
spec:
  deployment:
    args:
    - '-y'
    - '@mcp-get-community/server-curl'
    cmd: npx
    port: 3000
  stdioTransport: {}
  transportType: stdio
EOF
```

### Verify Deployment

```bash
# Check the MCPServer resource
kubectl get mcpserver -n kagent mcp-curl

# Check the pod is running
kubectl get pods -n kagent -l app.kubernetes.io/name=mcp-curl

# View logs
kubectl logs -n kagent -l app.kubernetes.io/name=mcp-curl
```

---

## Part 2: Deploy the Curl Agent

```bash
kubectl apply -f - <<EOF
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: curl-agent
  namespace: kagent
spec:
  description: An HTTP client agent that can make requests to any URL using curl.
  type: Declarative
  declarative:
    modelConfig: default-model-config
    systemMessage: |-
      You are an HTTP client agent that helps users make web requests and interact with APIs.
      
      # Capabilities
      You can:
      - Make GET requests to fetch data from URLs
      - Make POST/PUT/PATCH requests to send data to APIs
      - Set custom headers (authentication, content-type, etc.)
      - Handle JSON and other response formats
      
      # Instructions
      - If the user question is unclear, ask for clarification
      - Always explain what request you're about to make before executing
      - Format JSON responses in a readable way
      - Warn users about sending sensitive data over HTTP (non-HTTPS)
      - If you don't know how to help, say so clearly
      
      # Response format
      - ALWAYS format your response as Markdown
      - Display response status, headers, and body clearly
      - Parse and format JSON responses for readability
    tools:
      - type: McpServer
        mcpServer:
          name: mcp-curl
          kind: MCPServer
          toolNames:
            - curl
    a2aConfig:
      skills:
        - id: http-requests
          name: HTTP Requests
          description: Make HTTP requests to any URL with custom methods, headers, and body
          inputModes:
            - text
          outputModes:
            - text
          tags:
            - http
            - api
            - curl
            - web
          examples:
            - "Fetch the contents of https://api.github.com"
            - "POST JSON data to an API endpoint"
            - "Check if a website is up"
            - "Get the headers from a URL"
            - "Make an authenticated API request"
EOF
```

---

## Part 3: Testing the Agent

### Using kagent CLI

```bash
# Simple GET request
kagent invoke --agent curl-agent --task "Fetch https://httpbin.org/get"

# Check a website status
kagent invoke --agent curl-agent --task "Check if https://google.com is responding"

# Get JSON from an API
kagent invoke --agent curl-agent --task "Get the current Bitcoin price from a public API"
```

### Using kagent Dashboard

```bash
kagent dashboard
```

Navigate to `curl-agent` and try prompts like:
- "Make a GET request to https://api.github.com"
- "POST {'name': 'test'} to https://httpbin.org/post"
- "Fetch https://httpbin.org/headers and show me the response headers"

---

## Use Cases

| Use Case | Example Prompt |
|----------|----------------|
| API Testing | "POST a JSON payload to my API endpoint" |
| Health Checks | "Check if https://myservice.com/health returns 200" |
| Data Fetching | "Get the latest data from this REST API" |
| Webhook Testing | "Send a test webhook to this URL" |
| Header Inspection | "Show me what headers this server returns" |
| Authentication | "Make a request with Bearer token authentication" |

---

## References

- [MCP Curl Server](https://github.com/mcp-get/community-servers/tree/HEAD/src/server-curl)
- [kagent Documentation](https://kagent.dev/docs/kagent)
- [kmcp Documentation](https://kagent.dev/docs/kmcp)
