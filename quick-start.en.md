# Quick Start

About 10 minutes from zero to your first managed server: start NodeGuard with Docker Compose, create the administrator and connect the first Linux agent.

German version: [quick-start.md](quick-start.md)

## Requirements

**Docker host (server)**

- Docker with the Compose plugin (`docker compose version`)
- A free port for the web app (default `80`)
- Outbound HTTPS access to `https://cdn-uem.backend.pcsmw.de` (agent packages are downloaded from there, bug reports and telemetry are sent there)

**Target machine (agent)**

- Linux `x86_64` with `systemd`
- `bash`, `curl`, `unzip`, `sha256sum` and GNU `grep`
- Root or `sudo` rights and network access to the Docker host

## 1. Start the server

```bash
mkdir -p /opt/nodeguard && cd /opt/nodeguard

curl -fsSLO https://raw.githubusercontent.com/mamphis/NodeGuard-Docs/master/docker/docker-compose.yml
curl -fsSL  https://raw.githubusercontent.com/mamphis/NodeGuard-Docs/master/docker/.env.example -o .env

# Generate a database password and write it to .env
sed -i "s/^DB_PASSWORD=.*/DB_PASSWORD=$(openssl rand -hex 24)/" .env

docker compose up -d
docker compose logs -f nodeguard
```

On start, the container runs the database migrations and then starts the web app. Without `DB_PASSWORD` set, `docker compose up` deliberately aborts with a message.

<details>
<summary>The <code>docker-compose.yml</code> to copy</summary>

```yaml
services:
  db:
    image: postgres:18
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER:-nodeguard}
      POSTGRES_PASSWORD: ${DB_PASSWORD:?set DB_PASSWORD in the .env file}
      POSTGRES_DB: ${DB_DATABASE:-nodeguard}
    volumes:
      - ./pg_data:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10

  nodeguard:
    image: docker.backend.pcsmw.de/nodeguard-server:${NODEGUARD_VERSION:-latest}
    restart: unless-stopped
    environment:
      DATABASE_URL: "postgresql://${DB_USER:-nodeguard}:${DB_PASSWORD:?set DB_PASSWORD in the .env file}@db:5432/${DB_DATABASE:-nodeguard}"
      DATA_BASE_DIR: "/app/data"
      CDN_URL: "https://cdn-uem.backend.pcsmw.de"
      # Telemetry to the CDN, set to "false" to disable – see the Telemetry section
      TELEMETRY_ENABLED: "true"
      # SESSION_SECRET: "change-me-to-a-long-random-string"
      # SESSION_COOKIE_SECURE: "auto"
      # Optional: single sign-on via OIDC (or set it up in the web app), see oidc.md
      # OIDC_ISSUER: "https://auth.example.com/application/o/nodeguard/"
      # OIDC_CLIENT_ID: "nodeguard"
      # OIDC_CLIENT_SECRET: "…"
    volumes:
      - ./ng_data:/app/data
    ports:
      - "${NODEGUARD_PORT:-80}:80"
    depends_on:
      db:
        condition: service_healthy
```

Put a `.env` file in the same directory:

```dotenv
DB_USER=nodeguard
DB_PASSWORD=<long-random-password>
DB_DATABASE=nodeguard
NODEGUARD_PORT=80
NODEGUARD_VERSION=latest
```

</details>

## 2. Create the administrator

1. Open `http://<server-address>` in your browser.
2. The **initial setup** appears automatically. Create the administrator account with your own password.
   Alternatively, open the **Single sign-on** tab, enter an OIDC provider (for example Authentik or Keycloak) and sign in there. See [Sign-in with OIDC](oidc.md).
3. Sign in.

Plain HTTP on the home network works without a certificate.

## 3. Provide the agent package

The install script downloads the agent from the NodeGuard server, so a package must be available there first.

1. Open **Settings → Downloads**. Under **Download Files**, the versions from the CDN appear shortly after start (otherwise click the refresh icon).
2. Click Download for the **newest version**. A checksum then appears in the table.

> The install script uses the highest version downloaded to the server. Download the newest version so that new agents are up to date right away.

## 4. Install the first agent

1. Open **Settings → Agents**, enter a label under **Agent invites** (for example `Homelab`) and click **Create invite**.
2. Click **Get command** and check that **Server URL** shows the address the target machine uses to reach the server (not `localhost`).
3. Run the command on the target machine as root or with `sudo`:

   ```bash
   curl -fsSL http://<server-address>/api/agents/install.sh | bash -s -- \
     --invite-code <token> --server-url http://<server-address> --server-fingerprint <sha256>
   ```

   The output ends with `NodeGuard Agent installation complete.`
4. In the web app, go to **Settings → Agents → Approval queue** and click **Approve**.

The agent now appears under **Connected agents** and is online within a minute.

## 5. Try it out

- Under **Hardware** in the agent details, check CPU, RAM and drives.
- Under **Script execution**, run a short Bash script, for example `uptime`.
- Create tags under **Settings → Tags** (name and color) and assign them in **Agents** via **Assign Tag**. The agent list can be filtered by tag, operating system, version and status.
- Create a script under **Automation → Script library** and schedule it under **Automation → Schedules** with **New schedule**, using cron, for a tag or individual agents. Results appear under **Automation → Script runs**.

Good to know:

- A script runs for at most **60 minutes** by default. After that, the agent terminates it together with all processes it started and reports it as failed (exit code `-1`).
- At most **1 MB** of output per stream (stdout and stderr) is stored. The rest is discarded and marked as truncated.
- A script from the script library can only be deleted once no schedule uses it anymore. Existing script runs are kept.
- Deleting a tag also removes it from schedules. Schedules left without any target are disabled.
- The web app can also be used on a smartphone; tables are shown as cards there.

## Telemetry

The bundled `docker-compose.yml` enables telemetry. The server then reports to `https://cdn-uem.backend.pcsmw.de/telemetry`:

- **Slow database queries** (from 1 second) – SQL statement (without parameter values), number of rows and duration.
- **Server errors** (HTTP 5xx) – error message and stack trace.

Each report contains the anonymous instance ID from `ng_data/instance.json` and the server version. User, agent or inventory data is not sent.

| Variable | Meaning |
|---|---|
| `TELEMETRY_ENABLED` | `true` (or `1`, `yes`, `on`) enables telemetry. If the variable is missing or set to `false`, nothing is sent. |
| `CDN_URL` | Target for telemetry, bug reports and agent packages. Keep `https://cdn-uem.backend.pcsmw.de`. |

To disable: set `TELEMETRY_ENABLED: "false"` in `docker-compose.yml` and run `docker compose up -d`.

## Important: backup

Back up the `ng_data/` directory. It contains the **CA key for the agent certificates** (`agent-ca/ca-key.pem`). If it is lost, all agents have to be installed again. The database is stored in `pg_data/`.

## Update

```bash
cd /opt/nodeguard
docker compose pull && docker compose up -d
```

## Common problems

| Symptom | Solution |
| --- | --- |
| `set DB_PASSWORD in the .env file` | Enter `DB_PASSWORD` in the `.env` file. |
| `curl: (22) … 404` during installation | No agent package has been downloaded yet → step 3. |
| Agent registered but not visible | Not approved yet → **Settings → Agents → Approval queue**. |
| No versions under **Settings → Downloads** | The Docker host cannot reach the CDN: `curl https://cdn-uem.backend.pcsmw.de/versions` |
| Agent stays offline | The server URL is not reachable from the target machine (firewall, port, `localhost`). |
| Script stops after 60 minutes | Maximum runtime reached. Split long tasks or start them in the background (for example as a systemd service). |
| Script cannot be deleted | A schedule still uses it. Delete the schedule under **Automation → Schedules** or switch it to another script. |
| Password forgotten | `docker compose exec nodeguard reset-password`, then run the initial setup again. |
| Single sign-on problems | See [OIDC troubleshooting](oidc.md#5-troubleshooting). |

## Feedback

Something broken or missing? Report it with the **bug report button** in the web app, or open an [issue](https://github.com/mamphis/NodeGuard-Docs/issues) here. Bug reports are only sent when you submit them yourself.
