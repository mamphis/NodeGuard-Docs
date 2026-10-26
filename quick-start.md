# Quick Start

In etwa 10 Minuten von null zum ersten verwalteten Server: NodeGuard per Docker Compose starten, Administrator anlegen und den ersten Linux-Agent einbinden.

## Voraussetzungen

**Docker-Host (Server)**

- Docker mit Compose-Plugin (`docker compose version`)
- Freier Port für die Web-App (Standard `80`)
- Ausgehender HTTPS-Zugriff auf `https://cdn-uem.backend.pcsmw.de` (dort liegen die Agent-Pakete)

**Zielrechner (Agent)**

- Linux `x86_64` mit `systemd`
- `bash`, `curl`, `unzip`, `sha256sum` und GNU `grep`
- Root- oder `sudo`-Rechte und Netzwerkzugriff auf den Docker-Host

## 1. Server starten

```bash
mkdir -p /opt/nodeguard && cd /opt/nodeguard

curl -fsSLO https://raw.githubusercontent.com/mamphis/NodeGuard-Docs/master/docker/docker-compose.yml
curl -fsSL  https://raw.githubusercontent.com/mamphis/NodeGuard-Docs/master/docker/.env.example -o .env

# Datenbank-Passwort erzeugen und eintragen
sed -i "s/^DB_PASSWORD=.*/DB_PASSWORD=$(openssl rand -hex 24)/" .env

docker compose up -d
docker compose logs -f nodeguard
```

Beim Start führt der Container die Datenbank-Migrationen aus und startet danach die Web-App. Ohne gesetztes `DB_PASSWORD` bricht `docker compose up` bewusst mit einer Meldung ab.

<details>
<summary>Die <code>docker-compose.yml</code> zum Kopieren</summary>

```yaml
services:
  db:
    image: postgres:18
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USER:-nodeguard}
      POSTGRES_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD in der .env setzen}
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
      DATABASE_URL: "postgresql://${DB_USER:-nodeguard}:${DB_PASSWORD:?DB_PASSWORD in der .env setzen}@db:5432/${DB_DATABASE:-nodeguard}"
      DATA_BASE_DIR: "/app/data"
      CDN_URL: "https://cdn-uem.backend.pcsmw.de"
      # SESSION_SECRET: "change-me-to-a-long-random-string"
      # SESSION_COOKIE_SECURE: "auto"
      # Optional: Single Sign-on per OIDC (alternativ in der Web-App einrichten), siehe oidc.md
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

Dazu eine `.env` im selben Verzeichnis:

```dotenv
DB_USER=nodeguard
DB_PASSWORD=<langes-zufälliges-passwort>
DB_DATABASE=nodeguard
NODEGUARD_PORT=80
NODEGUARD_VERSION=latest
```

</details>

## 2. Administrator anlegen

1. `http://<server-adresse>` im Browser öffnen.
2. Die **Ersteinrichtung** erscheint automatisch – Administratorkonto mit eigenem Passwort anlegen.
   Alternativ im Reiter **Single sign-on** einen OIDC-Provider (z. B. Authentik, Keycloak) eintragen und sich dort anmelden – siehe [Anmeldung per OIDC](oidc.md).
3. Anmelden.

Reines HTTP im Heimnetz funktioniert ohne Zertifikat.

## 3. Agent-Paket bereitstellen

Das Installationsskript lädt den Agent vom NodeGuard-Server, deshalb muss dort zuerst ein Paket liegen.

1. **Settings → Download Files** öffnen. Kurz nach dem Start erscheinen die Versionen aus dem CDN (sonst Aktualisieren klicken).
2. Bei der **neuesten Version** auf Download klicken. Danach steht eine Prüfsumme in der Tabelle.

> Immer die neueste Version herunterladen – das Skript nimmt den zuletzt angelegten Eintrag.

## 4. Ersten Agent installieren

1. **Settings → Agent invites** → Label vergeben (z. B. `Homelab`) → **Create invite**.
2. **Get command** klicken und prüfen, dass unter **Server URL** die Adresse steht, unter der der Zielrechner den Server erreicht (nicht `localhost`).
3. Den Befehl auf dem Zielrechner als root bzw. mit `sudo` ausführen:

   ```bash
   curl -fsSL http://<server-adresse>/api/agents/install.sh | bash -s -- \
     --invite-code <token> --server-url http://<server-adresse> --server-fingerprint <sha256>
   ```

   Die Ausgabe endet mit `NodeGuard Agent installation complete.`
4. In der Web-App **Settings → Approval queue** → **Approve**.

Der Agent erscheint jetzt unter **Connected agents** und ist innerhalb einer Minute online.

## 5. Ausprobieren

- Im Agent-Detail unter **Hardware** CPU, RAM und Laufwerke ansehen.
- Unter **Script execution** ein kurzes Bash-Skript ausführen, z. B. `uptime`.
- In der **Script library** ein Skript anlegen und als Cron-**Schedule** auf einen Tag planen.

## Wichtig: Backup

Sichere das Verzeichnis `ng_data/`. Es enthält den **CA-Schlüssel für die Agent-Zertifikate** (`agent-ca/ca-key.pem`). Geht er verloren, müssen alle Agents neu installiert werden. Die Datenbank liegt in `pg_data/`.

## Update

```bash
cd /opt/nodeguard
docker compose pull && docker compose up -d
```

## Häufige Probleme

| Symptom | Lösung |
| --- | --- |
| `DB_PASSWORD in der .env setzen` | `DB_PASSWORD` in der `.env` eintragen. |
| `curl: (22) … 404` beim Installieren | Noch kein Agent-Paket heruntergeladen → Schritt 3. |
| Agent registriert, aber nicht sichtbar | Noch nicht freigegeben → **Approval queue**. |
| Keine Versionen in **Download Files** | Docker-Host erreicht den CDN nicht: `curl https://cdn-uem.backend.pcsmw.de/versions` |
| Agent bleibt offline | Server-URL vom Zielrechner aus nicht erreichbar (Firewall, Port, `localhost`). |
| Passwort vergessen | `docker compose exec nodeguard reset-password`, danach Ersteinrichtung erneut. |
| Probleme mit Single Sign-on | Siehe [Fehlerbehebung OIDC](oidc.md#5-fehlerbehebung). |

## Feedback

Etwas klemmt oder fehlt? Über den **Bug-Report-Button** in der Web-App melden oder hier ein [Issue](https://github.com/mamphis/NodeGuard-Docs/issues) eröffnen. Bug-Reports werden nur gesendet, wenn du sie aktiv abschickst.
