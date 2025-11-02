# Architecture Overview

## Component Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                           │
│                                                                       │
│  ┌─────────────┐                                                     │
│  │   n8n Pod   │                                                     │
│  │ (namespace: │                                                     │
│  │    n8n)     │                                                     │
│  └──────┬──────┘                                                     │
│         │                                                             │
│         │ HTTP Request                                                │
│         │                                                             │
│         ▼                                                             │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │        kgateway-system namespace                        │        │
│  │                                                         │        │
│  │  ┌──────────────┐         ┌─────────────────┐         │        │
│  │  │  kgateway    │ manages │  agentgateway   │         │        │
│  │  │ (controller) │────────►│   (dataplane)   │         │        │
│  │  └──────────────┘         └────────┬────────┘         │        │
│  │         │                           │                   │        │
│  │         │ manages                   │                   │        │
│  │         ▼                           │                   │        │
│  │  ┌──────────────┐                  │                   │        │
│  │  │   Gateway    │                  │                   │        │
│  │  │(agentgateway)│◄─────────────────┘                   │        │
│  │  └──────┬───────┘                                       │        │
│  │         │ parentRef                                     │        │
│  │         ▼                                               │        │
│  │  ┌──────────────┐                                       │        │
│  │  │  HTTPRoute   │                                       │        │
│  │  │ (mcp-route)  │                                       │        │
│  │  └──────┬───────┘                                       │        │
│  │         │ backendRef                                    │        │
│  │         ▼                                               │        │
│  │  ┌──────────────┐                                       │        │
│  │  │   Backend    │                                       │        │
│  │  │ (mcp-backend)│                                       │        │
│  │  └──────────────┘                                       │        │
│  └────────────────────────────────────┼───────────────────┘        │
│                                        │                             │
│                                        │ forwards to                 │
│                                        │                             │
│                                        ▼                             │
│  ┌──────────────────────────────────────────────────────┐          │
│  │         default namespace                             │          │
│  │                                                       │          │
│  │  ┌─────────────────────────────────────────────┐    │          │
│  │  │  MCP Server (mcp-website-fetcher)          │    │          │
│  │  │  - Deployment & Service                     │    │          │
│  │  │  - Protocol: SSE (Server-Sent Events)       │    │          │
│  │  │  - Port: 80 → 8000                          │    │          │
│  │  └─────────────────────────────────────────────┘    │          │
│  └──────────────────────────────────────────────────────┘          │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

## Request Flow

1. **n8n workflow execution**
   - User triggers workflow in n8n
   - Workflow sends HTTP POST request to agentgateway

2. **Gateway receives request**
   - Request arrives at Gateway resource (agentgateway) listening on port 8080
   - Gateway is managed by kgateway control plane

3. **Route matching**
   - Gateway evaluates HTTPRoute (mcp-route) attached via parentRef
   - HTTPRoute matches the incoming request
   - Routes to Backend resource (mcp-backend) via backendRef

4. **Backend forwarding**
   - Backend configuration specifies:
     - Target: mcp-website-fetcher.default.svc.cluster.local
     - Port: 80
     - Protocol: SSE (Server-Sent Events)
   - agentgateway forwards request to MCP server

5. **MCP server processing**
   - MCP server receives request
   - Processes the website fetching operation
   - Returns response via SSE protocol

6. **Response flow**
   - Response flows back through agentgateway
   - agentgateway returns response to n8n
   - n8n workflow processes the result

## Key Components

### kgateway (Control Plane)
- Manages Gateway API resources
- Configures routing rules
- Monitors and updates agentgateway configuration
- Namespace: `kgateway-system`

### agentgateway (Data Plane)
- Handles actual request routing
- Proxies requests to MCP servers
- Implements Gateway API specifications
- Supports SSE protocol for MCP communication
- Namespace: `kgateway-system`

### Gateway Resource
- Kubernetes Gateway API resource defining the entry point
- Specifies listeners (protocol, port)
- Managed by kgateway controller
- HTTPRoutes attach to Gateway via parentRef
- Name: `agentgateway`
- Namespace: `kgateway-system`
- Note: Often created automatically by Helm installation

### Backend Resource
- Custom resource defining MCP server targets
- Specifies connection details:
  - Host
  - Port
  - Protocol (SSE)
- Namespace: `kgateway-system`

### HTTPRoute Resource
- Defines routing rules
- Connects Gateway to Backend
- Matches incoming requests
- Namespace: `kgateway-system`

### MCP Server (website-fetcher)
- Implements Model Context Protocol
- Provides website fetching capabilities
- Exposes SSE endpoint on port 8000
- Namespace: `default`

### n8n
- Workflow automation platform
- Client that initiates requests
- Processes MCP server responses
- Namespace: `n8n`

## Protocol: Server-Sent Events (SSE)

The MCP protocol uses SSE for communication:
- Unidirectional communication from server to client
- Maintains persistent connection
- Allows server to push updates
- Standard HTTP protocol with `text/event-stream` content type

## Networking

### Service Communication
- **n8n → agentgateway**: `agentgateway.kgateway-system.svc.cluster.local`
- **agentgateway → MCP Server**: `mcp-website-fetcher.default.svc.cluster.local:80`

### Port Mappings
- n8n Service: 5678 → 5678
- agentgateway Service: 8080 → 8080
- MCP Server Service: 80 → 8000 (with appProtocol: kgateway.dev/mcp)

## Configuration Flow

```
Helm Chart Installation
         │
         ▼
   CRDs Installed
         │
         ├─► Gateway API CRDs
         └─► kgateway CRDs
         │
         ▼
  kgateway Deployed
         │
         ├─► Controller pods
         └─► agentgateway pods
         │
         ▼
  Backend & HTTPRoute Created
         │
         └─► Routing rules configured
         │
         ▼
    MCP Server Deployed
         │
         └─► Ready to receive requests
```

## Observability

### Monitoring Points
1. **n8n workflow execution logs**
   - View in n8n UI
   - Track request/response

2. **agentgateway logs**
   ```bash
   kubectl logs -n kgateway-system -l app=agentgateway
   ```

3. **MCP server logs**
   ```bash
   kubectl logs -n default -l app=mcp-website-fetcher
   ```

4. **Backend status**
   ```bash
   kubectl describe backend mcp-backend -n kgateway-system
   ```

5. **HTTPRoute status**
   ```bash
   kubectl describe httproute mcp-route -n kgateway-system
   ```

## Security Considerations

- All communication within cluster uses internal DNS
- n8n uses basic authentication (configurable)
- Services are not exposed externally by default
- Use Ingress/LoadBalancer for external access if needed
- Consider using NetworkPolicies for additional security
