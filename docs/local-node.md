# Zamolxis Node

## Purpose

Zamolxis Node is the trusted local execution bridge installed on every workstation.

```text
Control Plane
     ^
     | authenticated outbound connection
     |
Zamolxis Node
     |
     +-- repository permissions
     +-- runtime discovery
     +-- process/session lifecycle
     +-- event normalization
     '-- local policy enforcement
```

## Security boundary

The Node, not the cloud control plane, has access to local files and agent runtimes.

Default policy should be deny-by-default with explicit repository grants.

Example:

```text
Allowed Workspaces

~/Projects/acme-platform     Read / Write / Commands
~/Projects/product-alpha    Read / Write / Commands
~/Documents                 DENIED
```

Sensitive capabilities such as deployment, destructive commands or access outside registered workspaces can require explicit approval.

## Platform strategy

The core daemon should be cross-platform TypeScript/Node where practical.

Platform packaging is separate:

- desktop/menu-bar shell where useful
- background service on server/workstation environments

Both expose the same Node protocol.

## Connection model

Nodes initiate outbound connections. Zamolxis should not require users to expose SSH, configure router port forwarding, or give a workstation a public IP.

## Manual session discovery

Where a runtime exposes sufficient local state, the Node may discover a session started outside Zamolxis.

Example UX:

```text
New local agent session detected
Product: Acme Platform

Attach to:
  - Product Alpha — Reporting Dashboard
  - Create new Work Session
  - Ignore
```
