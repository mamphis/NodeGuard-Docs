# NodeGuard

A lightweight self-hosted UEM for local networks

Manage the servers in your home network centrally through a web app: status and hardware at a glance, run and schedule scripts, view logs and files, update agents.

German version: [README.md](README.md)

## Highlights

**Agents at a glance:** status, certificate, operating system, tags and agent version of all servers in one view.

![List of connected agents](images/agents.png)

**Availability and reports:** uptime from heartbeats and success rate of executed actions.

![Availability and action statistics](images/availability.png)

**Schedules:** run scripts automatically on servers via cron, selected by tag or individually.

![Schedules for scripts](images/schedules.png)

## Typical use cases

The examples are Bash scripts for Linux agents (`x86_64`) using `apt`. Windows agents are not supported yet.

### Plan updates

Updates run in NodeGuard as scripts that you execute on a schedule:

1. Create a script under **Automation → Script library**, for example:

   ```bash
   export DEBIAN_FRONTEND=noninteractive
   apt-get update && apt-get -y upgrade
   ```

2. Under **Automation → Schedules**, create a cron schedule with **New schedule**, for example `0 3 * * 0` (Sundays at 03:00 UTC). Select a tag (for example `Homelab`) or individual agents as the target.
3. The result per server is shown under **Automation → Script runs**.

> Schedules only run scripts. You trigger updates of the NodeGuard agent itself manually, and there is no automatic detection of missing security updates yet.

### Install software

There is no software management in NodeGuard yet. Install packages with the package manager inside a script, for example:

```bash
DEBIAN_FRONTEND=noninteractive apt-get install -y htop
```

How to run the script:

- **Once:** create the script in the script library and assign it to an agent or tag. The execution overview shows the result of all agents on one page.
- **Regularly:** create the script as a schedule, for example to install packages on new servers.
- **For testing:** run it on a single agent under **Script execution** in the agent details.

Tip: `dpkg -s htop >/dev/null 2>&1 || apt-get install -y htop` installs only what is missing and is safe to repeat.

### Remove software

Removal also runs through a script. Execution and checking the result work the same way as for installation:

```bash
DEBIAN_FRONTEND=noninteractive apt-get remove -y htop
```

`apt-get purge` also removes configuration files. A software inventory with an uninstall action in the UI is still planned.

## Getting started

- **[Quick Start](quick-start.md)** – start the server with Docker Compose and connect the first agent
- **[Sign-in with OIDC](oidc.md)** – set up single sign-on with Authentik, Keycloak, Authelia and others
- [`docker/docker-compose.yml`](docker/docker-compose.yml) and [`docker/.env.example`](docker/.env.example) – example setup
