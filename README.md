# BlackHawk

BlackHawk is a web application security assessment tool built on [OWASP ZAP](https://www.zaproxy.org/). It provides a web dashboard to spider targets, run active vulnerability scans, discover attack surfaces, deduplicate findings, calculate deterministic risk scores, compare scans over time, and export HTML reports.

The backend is written in Go, the frontend is built with React and Vite, and the stack runs via Docker Compose.

<p align="center">
  <img src="screenshots/screenshot1.png" alt="BlackHawk dashboard screenshot" width="100%">
</p>

## Features

### Scanner core

- Four scan modes: Quick, Fast, Deep, and Stealth, each with configured crawl depth limits
- Real-time spider and active scan progress streamed over WebSocket
- Alert filtering across High, Medium, Low, and Informational risk levels
- Scan cancellation and restart controls
- Vulnerability scanning for XSS, SQL injection, and server misconfigurations via ZAP
- Report export in JSON and HTML formats

### Security analytics (v2.0)

- Attack surface discovery: extracts every URL visited from ZAP's site tree and alert data, normalizes paths (lowercasing hosts, stripping default ports, sorting query parameters, removing fragments), and stores each entry with HTTP method, status code, content type, parameters, and discovery timestamp.
- Vulnerability correlation: groups raw alerts into findings using plugin ID, normalized path, parameter, and alert name. If an alert occurs on multiple URLs, BlackHawk presents it as a single finding listing all affected endpoints while preserving each URL's request evidence and attack payload.
- Deterministic risk scoring: computes a 0 to 100 security score using finding severity, confidence, and affected endpoint count. The calculation contains no random or time-based factors. See [Scoring methodology](#scoring-methodology).
- Security overview: displays score metrics, severity counts, vulnerability categories, top affected endpoints, and HTTP method and status distributions.
- Scan comparison: compares two completed scans to identify new, fixed, and persistent findings, endpoint additions or removals, and the net score change.
- Assessment reports: generates HTML reports containing an executive summary, target details, scan settings, risk scores, attack surface listings, and remediation steps.
- Detailed finding view: shows vulnerability descriptions, severity ratings, confidence levels, affected endpoints, parameters, attack payloads, impact descriptions, remediation advice, and CWE tags.

## Architecture

```
┌───────────────────────────────┐
│  Browser (React/Vite SPA)     │
│  Landing · Dashboard · New    │
│  Scan · Progress · Attack     │
│  Surface · Security Overview  │
│  · Compare · Report           │
└──────────────┬────────────────┘
               │ HTTP + WebSocket
┌──────────────▼────────────────────────────────┐
│  Go API server (chi router)                   │
│                                               │
│  api/        handlers, WS hub, CORS           │
│  scan/       orchestrator (scan lifecycle)    │
│              endpoints.go   normalize+dedupe  │
│              findings.go    alert correlation │
│              scoring.go     deterministic     │
│                             0-100 score       │
│              analytics.go   surface+severity  │
│                             aggregation       │
│              compare.go     scan diffing      │
│              collect.go     endpoint harvest  │
│              store.go       SQLite storage    │
│  report/     HTML assessment generator        │
│  zapclient/  ZAP JSON API client              │
│  config/     env-based configuration          │
└──────────────┬────────────────────────────────┘
               │ REST (JSON)
┌──────────────▼────────────────┐      ┌──────────────────────┐
│  OWASP ZAP daemon             │      │  SQLite scans.db     │
│  spider · passive · ascan     │      │  scans · alerts ·    │
│  alerts · site tree           │      │  endpoints           │
└───────────────────────────────┘      └──────────────────────┘
```

The orchestrator controls ZAP through `zapclient`. Discovered endpoints are collected and normalized in `collect.go`, then saved to SQLite. Raw alerts are grouped into findings in `findings.go`, evaluated by the scoring engine in `scoring.go`, and served through the HTTP and WebSocket API.

## Quick start

### Prerequisites

- Docker and Docker Compose
- Go 1.22+ and Node 20+ (optional, for local development outside containers)

### Run with Docker

```bash
git clone https://github.com/sh4dowbl4d3/BlackHawk.git
cd BlackHawk
docker compose up --build
```

The services start at the following URLs:
- Frontend: http://localhost:5174
- Backend API: http://localhost:8081
- ZAP daemon: http://localhost:8080

To override the default ZAP API key, set `ZAP_API_KEY` in a local `.env` file:

```bash
ZAP_API_KEY=changeme
```

### Local development

Terminal 1 (ZAP daemon):
```bash
docker run -p 8080:8080 ghcr.io/zaproxy/zaproxy:stable \
  zap.sh -daemon -host 0.0.0.0 -port 8080 \
  -config api.key=changeme \
  -config api.addrs.addr.name=.* \
  -config api.addrs.addr.regex=true
```

Terminal 2 (backend):
```bash
cd backend
go run ./cmd/server
```

Terminal 3 (frontend):
```bash
cd frontend
npm install
npm run dev
```

The backend reads configuration from environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `8081` | API listen port |
| `ZAP_HOST` | `http://localhost:8080` | ZAP daemon base URL |
| `ZAP_API_KEY` | `changeme` | ZAP API key |
| `CORS_ORIGIN` | `http://localhost:5174` | Allowed browser origin |
| `STORE_PATH` | `scans.db` | SQLite database path |

## API endpoints

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/scan` | Start a new scan |
| GET | `/api/scans` | List scan history |
| GET | `/api/status/:id` | Get scan status |
| POST | `/api/stop/:id` | Stop a running scan |
| DELETE | `/api/scan/:id` | Delete a scan |
| GET | `/api/report/:id` | Get raw scan report (JSON) |
| GET | `/api/report/:id/html` | Export HTML assessment |
| GET | `/api/ws/:id` | WebSocket for live progress |

### Analytics endpoints (v2.0)

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/scan/:id/endpoints` | Discovered endpoints; supports `?method=`, `?search=`, and `?minStatus=` filters |
| GET | `/api/scan/:id/surface` | Attack surface totals, method, status, and risk distributions |
| GET | `/api/scan/:id/analytics` | Overall score, risk counts, categories, and most-affected endpoints |
| GET | `/api/scan/:id/findings` | Correlated findings; supports `?risk=` filter |
| GET | `/api/scan/:id/findings/:findingId` | Single finding with request and response evidence |
| GET | `/api/compare/:baseId/:targetId` | Scan diff for two completed scans |

Endpoints that require completed scans return HTTP 400 if the scan is still running or has failed.

## Scoring methodology

The BlackHawk Security Score provides a 0 to 100 rating for prioritizing remediation. Because ZAP does not generate CVSS vectors, BlackHawk calculates this score directly from alert metadata while listing CWE and WASC identifiers separately on each finding.

```
score = clamp(100 − Σ deductions, 0, 100)

deduction(finding) = severityWeight × confidenceMultiplier × scopeFactor

severityWeight:      High 15 · Medium 7 · Low 2 · Informational 0.5
confidenceMultiplier: Confirmed/Firm 1.0 · Low 0.5 · other/empty 0.25
scopeFactor:         1 + min(affectedEndpoints−1, 4) × 0.2   (max 1.8×)
```

Properties:

- Outputs are deterministic: identical scan findings produce the exact same score, without random variance or clock dependence.
- Severity weighting accounts for most of the deduction: a High finding carries roughly twice the weight of a Medium finding.
- Lower confidence findings reduce score impact compared to confirmed alerts.
- Scope scaling increases deductions when an issue affects multiple endpoints, capped at 1.8x so a single alert cannot zero out the score.
- The calculation formula is included in every analytics API response.

## Vulnerability correlation

Raw ZAP alerts often produce duplicate records when the same issue appears across multiple URLs or scan phases. BlackHawk groups these alerts into findings using the plugin ID and alert name.

Each finding aggregates all affected URLs while retaining original alert data, including request methods, parameters, attack payloads, and evidence. Grouping alerts by vulnerability class keeps the issue list manageable. For instance, an anti-CSRF token missing across six pages is displayed as a single finding that lists all six affected endpoints.

## Scan comparison

Compare any two completed scans to review changes over time:

```
SCAN #12 → SCAN #15

Security Score: 61 → 78 (+17)
New vulnerabilities:  2
Fixed vulnerabilities: 7
Persistent vulnerabilities: 5
Endpoints: +3 new, −2 removed
```

Findings are matched across scans by plugin ID, alert name, path, and parameter. Endpoints are matched by normalized URL and HTTP method. The diff calculation is deterministic, ensuring repeated runs on the same two scans return identical results.

## Example workflow

```bash
# 1. Start the stack
docker compose up --build

# 2. Launch a deep scan of your local test target
curl -X POST localhost:8081/api/scan \
  -H 'Content-Type: application/json' \
  -d '{"target":"http://testphp.vulnweb.com","mode":"deep"}'
# → {"id":"…","status":"pending", …}

# 3. Watch progress (browser or WebSocket)
open localhost:5174/scan/<id>
# or: wscat -c ws://localhost:8081/api/ws/<id>

# 4. Review results
curl localhost:8081/api/scan/<id>/analytics   # score + severity breakdown
curl localhost:8081/api/scan/<id>/surface     # attack surface
curl "localhost:8081/api/scan/<id>/findings?risk=High"

# 5. Rescan after fixes and compare results
curl -X POST localhost:8081/api/scan -d '{"target":"http://testphp.vulnweb.com","mode":"deep"}' ...
curl localhost:8081/api/compare/<old-id>/<new-id>

# 6. View the HTML report
open localhost:8081/api/report/<new-id>/html
```

> Only scan systems you have explicit authorization to test, such as local lab environments like OWASP Juice Shop or DVWA. BlackHawk is intended for defensive security assessments and does not provide features to bypass authorization or evade detection.

## Testing

```bash
# Backend unit + integration tests (correlation, scoring, comparison, API)
cd backend && go test ./...

# Frontend production build (type-checked)
cd frontend && npm run build
```

Test suites cover endpoint normalization and deduplication, alert correlation across duplicate and partial alerts, deterministic scoring edge cases, scan comparison symmetry, and API route handling.

For end-to-end testing without a real ZAP daemon, a mock script is included:

```bash
python3 backend/tools/mockzap.py &                 # serves a fake ZAP on :8099
cd backend
PORT=18081 ZAP_HOST=http://127.0.0.1:8099 STORE_PATH=/tmp/scans.db go run ./cmd/server
# then drive scans against localhost:18081 as above
```

## Project structure

```
BlackHawk/
├── docker-compose.yml
├── backend/            # Go API + ZAP orchestration + analytics
│   ├── cmd/server/     # entrypoint
│   └── internal/
│       ├── api/        # chi handlers, WebSocket hub, CORS
│       ├── scan/       # models, orchestrator, endpoints, findings,
│       │               # scoring, analytics, comparison, store (SQLite)
│       ├── report/     # HTML assessment generator
│       ├── zapclient/  # ZAP JSON API client
│       └── config/     # env configuration
├── frontend/           # React/Vite dashboard
│   └── src/pages/      # Landing, Dashboard, NewScan, ScanProgress,
│                       # AttackSurface, SecurityOverview, Compare,
│                       # Report, Capabilities
└── screenshots/
```

## Roadmap and known limitations

- PDF export is not yet available (HTML report is print-friendly; `Ctrl+P` works well).
- Endpoint status codes depend on what ZAP's site tree reports; not-found pages may be absent.
- Comparison requires both scans to have reached `complete` status.
- Scheduled and recurring scans and multi-user authentication are planned for future releases.

## License

[GNU GPLv3](LICENSE)
