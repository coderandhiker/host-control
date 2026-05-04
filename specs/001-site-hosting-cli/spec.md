# Feature Specification: Site Hosting CLI

**Feature Branch**: `001-site-hosting-cli`  
**Created**: 2026-05-04  
**Status**: Draft  
**Input**: User description: "Create a tool that makes it simple to configure Caddy on an Ubuntu Linux server, along with the sites that are hosted. Each site will have either a Dockerfile or a Docker Compose file. The Registry will live on the server on which this tool runs."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Initialize Hosting Root (Priority: P1)

An operator runs the CLI for the first time on a server and initializes the hosting root directory (e.g., `/apps/hosting/web/`). The tool creates the required directory structure, sets up a Caddy configuration with the import-based pattern, creates a Docker Compose file for the Caddy container, and generates a manifest file that tracks all enrolled sites. The entire resulting directory is designed to be committed to version control.

**Why this priority**: Nothing else works without the hosting root. This establishes the foundation — the Caddy container, the manifest, and the directory conventions.

**Independent Test**: Can be fully tested by running `hostctl init /apps/hosting/web/` on a clean server and verifying the directory structure, Caddy container startup, and manifest file exist.

**Acceptance Scenarios**:

1. **Given** a server with Docker and a local registry running, **When** the operator runs `hostctl init /apps/hosting/web/`, **Then** the tool creates the hosting root with a `caddy/` directory containing a Caddyfile (with `import sites/*.Caddyfile`), a `docker-compose.yml` for Caddy, a `sites/` directory for site configs, an `excluded-ports.txt` file with commented instructions and examples, and a `manifest.yml` tracking all sites.
2. **Given** the hosting root already exists, **When** the operator runs `init` again, **Then** the tool reports that it is already initialized and does not overwrite existing files.
3. **Given** the hosting root is initialized, **When** the operator inspects the directory, **Then** every file is suitable for version control (no secrets, no generated state that should be excluded).

---

### User Story 2 - Add a Simple Site (Priority: P1)

An operator adds a new site by specifying a Docker image and one or more hostnames. The tool first validates that all hostnames resolve to the current server via `dig`. Then it allocates the next available port, auto-generates a `docker-compose.yml` from the image, generates a Caddyfile snippet, updates the manifest, pulls the image, runs `docker compose up -d`, reloads Caddy, waits for the container to become healthy, then curls each hostname to verify routing and TLS are working. One command, fully deployed and verified.

If the operator provides `--image`, the tool auto-generates a `docker-compose.yml`. If the operator provides `--compose`, the tool uses the supplied file instead. Either way, the site ends up with a `docker-compose.yml` in its directory.

**Why this priority**: This is the core value proposition — adding a site should be one command that ends with a verified, live site.

**Independent Test**: After `init`, run `hostctl add --image frost:5000/myapp:latest --hostnames myapp.com,www.myapp.com` and verify the site is live — the CLI itself reports the verification result.

**Acceptance Scenarios**:

1. **Given** an initialized hosting root, **When** the operator runs `hostctl add --image frost:5000/myapp:latest --hostnames myapp.com,www.myapp.com`, **Then** the tool creates a site directory, auto-generates a `docker-compose.yml`, generates a Caddyfile, updates the manifest, pulls the image, starts the container, reloads Caddy, waits for readiness, curls each hostname over HTTPS, and reports success or failure with details.
2. **Given** an initialized hosting root, **When** the operator adds a site with a hostname already claimed by another site, **Then** the tool rejects the add and reports the conflict before generating any files.
3. **Given** an initialized hosting root with existing port allocations, **When** the operator adds a new site, **Then** the tool allocates the next available port that is not a well-known port (below 1024), not currently in use on the host, not listed in `excluded-ports.txt`, and not already allocated in the manifest.
4. **Given** a site that needs environment variables, **When** the operator runs `hostctl add --image frost:5000/myapp:latest --hostnames myapp.com --env SECRET_KEY,DB_URL`, **Then** the tool generates a `.env.example` file listing those variables as placeholders, halts before starting the container, and prompts the operator to create a `.env` file with actual values before proceeding.
5. **Given** an initialized hosting root, **When** the operator adds a site whose hostnames do not resolve to the current server (verified via `dig`), **Then** the tool fails before generating any files or starting containers, and reports which hostnames have incorrect or missing DNS records.
6. **Given** the container starts and DNS is correct but the HTTPS curl check fails (e.g., TLS not yet provisioned, application error), **When** the verification step runs, **Then** the tool reports which hostnames failed, the HTTP status or error received, and leaves the site enrolled but flagged as unverified in the manifest.

---

### User Story 3 - Add a Multi-Container Site (Priority: P2)

An operator adds a site that requires multiple containers (e.g., API + admin + public frontend, like TournamentMaker). The operator provides a Docker Compose file and hostname-to-service mappings. Each service with a hostname mapping receives its own port allocation from the available pool. The tool copies the compose file into the site directory, generates per-service Caddyfile entries (each pointing to the service's allocated port), updates the manifest, starts all containers, reloads Caddy, and verifies each hostname.

**Why this priority**: Several existing sites (tournamentmaker, rdce-web) already use multiple containers. This is needed to model real workloads but can follow the simpler single-container story.

**Independent Test**: Run `hostctl add --compose ./my-compose.yml --name tournamentmaker --hostnames api.mysite.com:api,admin.mysite.com:admin,mysite.com:public` and verify the tool brings all containers up and curls each hostname successfully.

**Acceptance Scenarios**:

1. **Given** an initialized hosting root, **When** the operator runs `hostctl add --compose ./project-compose.yml --name mysite --hostnames api.mysite.com:api,admin.mysite.com:admin,mysite.com:public`, **Then** the tool copies the compose file into the site directory, generates Caddyfile entries, updates the manifest, runs `docker compose up -d`, reloads Caddy, and curls each hostname to verify.
2. **Given** a multi-container site is enrolled, **When** the operator lists sites, **Then** the multi-container site shows all its services and hostname mappings.

---

### User Story 4 - List and Inspect Sites (Priority: P2)

An operator lists all enrolled sites and inspects the details of a specific site — its hostnames, image, port, status, and volume mappings.

**Why this priority**: Operators need visibility into what is deployed. Essential for day-to-day management but not blocking initial setup.

**Independent Test**: After adding two sites, run `hostctl list` and `hostctl inspect myapp` and verify the output matches the manifest and running containers.

**Acceptance Scenarios**:

1. **Given** a hosting root with 3 enrolled sites, **When** the operator runs `hostctl list`, **Then** the tool displays a table showing each site's name, hostnames, image, port, and running status.
2. **Given** an enrolled site, **When** the operator runs `hostctl inspect myapp`, **Then** the tool displays the site's full configuration — hostnames, image, port mapping, environment variables (names only, not values), volume mounts, and whether the container is currently running.

---

### User Story 5 - Remove a Site (Priority: P3)

An operator removes a site from the hosting root. The tool removes the Caddyfile snippet, removes the site from the manifest, and stops/removes the container. The site directory can optionally be deleted or archived.

**Why this priority**: Needed for lifecycle management but less frequent than adding sites.

**Independent Test**: After adding a site, run `hostctl remove myapp` and verify the Caddyfile snippet is gone, the manifest is updated, and the container is stopped.

**Acceptance Scenarios**:

1. **Given** an enrolled site named "myapp", **When** the operator runs `hostctl remove myapp`, **Then** the tool stops the container, removes the Caddyfile snippet, removes the site from the manifest, and prompts whether to delete or archive the site directory.
2. **Given** an enrolled site, **When** the operator removes it and Caddy is reloaded, **Then** the hostnames previously routed to that site return a Caddy error page (not routed elsewhere).

---

### User Story 6 - Deploy / Update a Site (Priority: P2)

An operator updates a site to a new image tag. For multi-container sites, the operator can target a specific service with `--service`, or omit it to update all services. The tool validates DNS resolution for all affected hostnames (via `dig`) before proceeding, pulls the new image, recreates the container(s), reloads Caddy if the Caddyfile changed, waits for readiness, and curls the hostnames to verify the site is still serving correctly.

**Why this priority**: Deploying new versions is a frequent operation for live sites. Core to the GitOps workflow.

**Independent Test**: Run `hostctl deploy myapp --tag v2.0`, verify the container restarts with the new image and the tool reports successful hostname verification.

**Acceptance Scenarios**:

1. **Given** an enrolled single-container site running `frost:5000/myapp:v1`, **When** the operator runs `hostctl deploy myapp --tag v2`, **Then** the tool validates DNS for all hostnames via `dig`, updates the image tag in the site's compose file, pulls the new image, recreates the container, waits for it to become healthy, curls each hostname, and reports success.
2. **Given** an enrolled multi-container site, **When** the operator runs `hostctl deploy mysite --service api --tag v2`, **Then** the tool updates only the specified service's image tag, pulls the new image, recreates that service's container, and verifies the hostnames mapped to that service.
3. **Given** an enrolled multi-container site, **When** the operator runs `hostctl deploy mysite --tag v2` without `--service`, **Then** the tool updates all services in the site to the new tag, recreates all containers, and verifies all hostnames.
4. **Given** a deployment where DNS does not resolve to the current server, **When** the operator runs `hostctl deploy`, **Then** the tool fails before pulling images or restarting containers and reports the DNS mismatch.
5. **Given** a deployment in progress, **When** the new container fails its health check or hostname verification, **Then** the tool reports the failure with details (health check output, HTTP status codes) and leaves the previous state documented.

---

### User Story 7 - Apply All / Sync from Manifest (Priority: P3)

An operator clones the hosting root repo onto a fresh server (or after a git pull) and runs a single command to bring the entire platform up — all sites, Caddy, and networking — matching the version-controlled manifest. The tool handles everything: starts Caddy, pulls images, starts containers, reloads Caddy, and verifies every hostname.

**Why this priority**: This is the GitOps payoff — declarative server state from version control. Important but builds on all prior stories.

**Independent Test**: Clone the repo to a second server, run `hostctl up`, verify all sites come up and the tool reports verification results for every hostname.

**Acceptance Scenarios**:

1. **Given** a hosting root repo cloned to a new server with Docker and a local registry, **When** the operator runs `hostctl up`, **Then** the tool starts Caddy, pulls all images, runs `docker compose up -d` for each site, reloads Caddy, curls every hostname, and reports a summary table of results (site, hostname, status, TLS valid).
2. **Given** a hosting root repo cloned to a new server where sites require environment variables, **When** the operator runs `hostctl up` and `.env` files are missing, **Then** the tool emits a `.env` file for each affected site pre-populated from `.env.example` with placeholder values, reports exactly which sites are blocked and why, and skips those sites. The operator can fill in the values and re-run `hostctl up` to bring up the remaining sites.
3. **Given** a running platform where the manifest has been updated (site added via git), **When** the operator runs `hostctl up`, **Then** new sites are started, removed sites are stopped, changed sites are updated, and all hostnames are re-verified.

---

### Edge Cases

- What happens when Docker is not installed or the Docker daemon is not running? The tool MUST detect this at startup and report a clear error.
- What happens when the local registry is unreachable? The tool MUST report which sites cannot be pulled and continue with others.
- What happens when two sites try to claim the same port? The port allocator MUST prevent collisions; the manifest is the source of truth for port assignments. Additionally, the allocator MUST check that a candidate port is not in active use on the host and not listed in `excluded-ports.txt`.
- What happens when a site's `.env` file is missing required variables? The tool MUST emit a `.env` file from `.env.example` with placeholder values, report the issue clearly, and skip that site until the operator fills in the values and re-runs.
- What happens when DNS for a hostname does not resolve to the current server? The tool MUST check DNS via `dig` before deploying and fail with a clear message identifying which hostnames are not pointed correctly.
- What happens when the Caddyfile has a syntax error after modification? The tool MUST validate the Caddyfile (via `caddy validate`) before reloading.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST provide a CLI with subcommands: `init`, `add`, `remove`, `list`, `inspect`, `deploy`, and `up`.
- **FR-002**: System MUST generate a `manifest.yml` at the hosting root that declaratively describes all enrolled sites — their names, images, hostnames, ports, environment variable names, and volume mounts.
- **FR-003**: System MUST auto-allocate ports per service (each service in a site gets its own port). Allocated ports MUST NOT be well-known ports (below 1024), MUST NOT be currently in use on the host (verified before allocation), and MUST NOT appear in the operator-managed `excluded-ports.txt` file. The `init` command creates `excluded-ports.txt` with comment-based instructions (lines starting with `#` are ignored). All allocations are tracked in the manifest to prevent conflicts.
- **FR-004**: System MUST generate per-site Caddyfile snippets and a root Caddyfile that imports them, following the `import sites/*.Caddyfile` pattern already in use.
- **FR-005**: System MUST auto-generate a `docker-compose.yml` when the operator provides `--image` (single-container sites), or accept a supplied compose file via `--compose` (multi-container sites). Either path results in a `docker-compose.yml` in the site directory.
- **FR-006**: System MUST generate `.env.example` files for sites that require environment variables. When a `.env` file is missing at deploy time, the tool MUST emit a `.env` file pre-populated from `.env.example` with placeholder values and halt that site's deployment until the operator fills in actual values. Actual `.env` files (containing secrets) are gitignored.
- **FR-007**: System MUST validate the Caddyfile before applying changes (using `caddy validate` or equivalent).
- **FR-008**: System MUST support sites with multiple containers, each with independent hostname-to-port mappings.
- **FR-009**: System MUST produce a directory structure suitable for version control — no secrets, no transient Docker state, clear `.gitignore` patterns.
- **FR-010**: System MUST support a convention for volume mounts, where persistent data lives under a predictable path (e.g., `sites/<name>/data/`) and is declared in the manifest.
- **FR-011**: System MUST reload Caddy after adding, removing, or modifying site configurations.
- **FR-012**: System MUST detect and report hostname conflicts when adding a site.
- **FR-013**: System MUST work with a local Docker registry (default: `localhost:5000`).
- **FR-014**: System MUST run `docker compose up -d` (and `docker compose down` for removals) — the operator never runs Docker Compose commands manually.
- **FR-015**: System MUST perform post-deploy verification: wait for container readiness (configurable timeout, default 30 seconds), then curl each hostname over HTTPS to confirm routing and TLS are working. Report per-hostname results (HTTP status, TLS validity, response time).
- **FR-016**: System MUST track verification status per site in the manifest (verified/unverified/failed) so the operator can see at a glance which sites are confirmed working.
- **FR-017**: System MUST validate DNS resolution for all hostnames (via `dig`) before deploying or adding a site. If any hostname does not resolve to the current server's IP address, the tool MUST fail with a clear error and not proceed with deployment.

### Key Entities

- **Hosting Root**: The top-level directory (e.g., `/apps/hosting/web/`) containing Caddy config, site directories, and the manifest. One per server.
- **Manifest**: A version-controlled file (`manifest.yml`) at the hosting root that describes the desired state of all sites — the single source of truth.
- **Site**: A named unit of deployment with one or more containers, hostname mappings, port allocations, environment variables, and optional volume mounts.
- **Service**: A single container within a site. Simple sites have one service; complex sites (like TournamentMaker) have multiple.
- **Port Allocation**: A mapping from a service to a host port, managed by the tool to prevent conflicts.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: An operator can add a new single-container site (from zero to verified HTTPS traffic) with 1 command (`add`), completing in under 5 minutes including verification.
- **SC-002**: An operator can bring up the entire platform from a clean server with a cloned repo in under 10 minutes using 1 command (`up`), with verification results for every hostname.
- **SC-003**: The tool prevents 100% of port and hostname conflicts at enrollment time.
- **SC-004**: All configuration files (Caddyfile, docker-compose.yml, manifest.yml) pass validation before being applied — zero silent misconfigurations.
- **SC-005**: The hosting root directory can be committed to git and cloned to a second server to reproduce the full platform without manual file editing.

## Assumptions

- The server runs Ubuntu Linux with Docker and Docker Compose (v2) already installed.
- A local Docker registry is already running on `localhost:5000` (or a configurable address).
- Caddy will run as a Docker container with `network_mode: host`, consistent with the current setup.
- The operator has SSH access and runs the CLI directly on the server (not remotely).
- DNS records for site hostnames are managed separately (outside this tool's scope), but MUST be pointed to the server before using `add` or `deploy`.
- Caddy handles TLS certificate provisioning automatically via Let's Encrypt (default Caddy behavior).
- The tool is built as a Go application, distributed as a static binary (e.g., via GitHub Releases). It is not a daemon or long-running service.
- Image building and pushing to the local registry are handled separately (e.g., by CI runners or manual `docker build && docker push`). This tool manages deployment, not builds.
- The `.env` files containing actual secrets are never committed to version control.
