# Repos & Code Locations

## Active Repos

| Repo | What | Branch |
|------|------|--------|
| `ThrustNThunder/thundergate-dev` | ThunderGate runtime scaffold | master |
| `ThrustNThunder/thunderagent-ios` | ThunderCommo iOS code drops for Mack | build-20-redesign |
| `ThrustNThunder/tnt-workspace` | Design docs, memory, principles | master |
| `ThrustNThunder/aa-pilot-shared` | AA automation scripts for Rex | master |
| `ThrustNThunder/burt-jon-shared` | ThunderCommo build queue, iOS bugs | master |
| `ThrustNThunder/cli-jon-context` | This repo — your briefing | main |

## Key File Locations on ThunderBase

```
/home/ubuntu/thundergate-dev/          ThunderGate TypeScript scaffold
/home/ubuntu/.openclaw/workspace/      Main workspace + memory
/home/ubuntu/thundergate/              Live ThunderCommo bridge + relay
  extensions/thundercomm/
    bridge.mjs                         WebSocket bridge (port 8765)
    relay.mjs                          Federation relay (port 8767)
    inbox.mjs                          Message inbox + auth API (port 8769)
    run-inbox.mjs                      Inbox service entry point
```

## Live Services

| Service | Port | What |
|---------|------|------|
| OpenClaw gateway | 18789 | Jon's brain |
| ThunderCommo bridge | 8765 | WebSocket for iOS/web |
| ThunderCommo relay | 8767 | Federation |
| ThunderCommo inbox | 8769 | Message inbox + auth API |
| ThunderCommo web | 8766 | Web UI |

## API Endpoints (all at thunderai.us)

```
POST /api/auth/signup
POST /api/auth/signin
GET  /api/auth/me
POST /api/auth/refresh
GET  /api/inbox?since=<ts>
POST /api/inbox/ack
POST /api/messages
POST /api/devices/token
GET  /api/agent/identity
GET  /api/inbox/status
```

## GitHub PAT
[see credentials/github.json on ThunderBase]
