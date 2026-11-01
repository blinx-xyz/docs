# Service status

**Route:** `/tenant/healthcheck`

Service status is a quick connectivity check for the services the dashboard depends on. Each check shows an **operational** state, a version and the time it was last checked.

| Check | What it reports |
|---|---|
| **Backend API** | Whether the Blinx backend is reachable and ready, with its version and date |
| **Web app** | A ping of the public web app, when configured |
| **Public site** | A ping of the public `www` site, when configured |

Use it to confirm the platform is up before raising an issue. It needs no special permissions and is pinned in the sidebar's utility group, below **Settings**.
