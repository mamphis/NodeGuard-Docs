# NodeGuard
A Lightweight Selfhosted UEM for local networks

Verwalte die Server in deinem Heimnetz zentral über eine Web-App: Status und Hardware im Blick, Skripte ausführen und planen, Logs und Dateien einsehen, Agents aktualisieren.

## Einblicke

**Agents im Überblick:** Status, Zertifikat, Betriebssystem, Tags und Agent-Version aller Server auf einen Blick.

![Liste der verbundenen Agents](images/agents.png)

**Verfügbarkeit & Berichte:** Uptime aus den Heartbeats und Erfolgsquote der ausgeführten Aktionen.

![Verfügbarkeit und Aktionsauswertung](images/availability.png)

**Zeitpläne:** Skripte per Cron automatisch auf Servern ausführen, ausgewählt über Tags oder einzeln.

![Zeitpläne für Skripte](images/schedules.png)

## Typische Anwendungsfälle

Die Beispiele sind Bash-Skripte für Linux-Agents (`x86_64`) mit `apt`. Windows-Agents werden noch nicht unterstützt.

### Updates planen

Updates laufen bei NodeGuard als Skripte, die du per Zeitplan ausführst:

1. Unter **Automation → Script library** ein Skript anlegen, z. B.:

   ```bash
   export DEBIAN_FRONTEND=noninteractive
   apt-get update && apt-get -y upgrade
   ```

2. Unter **Automation → Schedules** mit **New schedule** einen Cron-Zeitplan anlegen, z. B. `0 3 * * 0` (sonntags um 03:00 UTC). Als Ziel wählst du einen Tag (z. B. `Homelab`) oder einzelne Agents.
3. Das Ergebnis pro Server steht unter **Automation → Script runs**.

> Zeitpläne führen nur Skripte aus. Updates des NodeGuard-Agents selbst stößt du manuell an, und eine automatische Erkennung fehlender Sicherheitsupdates gibt es noch nicht.

### Software installieren

Eine eigene Software-Verwaltung gibt es noch nicht. Installiert wird mit dem Paketmanager in einem Skript, z. B.:

```bash
DEBIAN_FRONTEND=noninteractive apt-get install -y htop
```

So führst du das Skript aus:

- **Einmalig:** Skript in der Script library anlegen und einem Agent oder Tag zuweisen. Die Ausführungsübersicht zeigt das Ergebnis aller Agents auf einer Seite.
- **Regelmäßig:** Skript als Zeitplan anlegen, z. B. um Pakete auf neuen Servern nachzuinstallieren.
- **Zum Testen:** Im Agent-Detail unter **Script execution** auf einem einzelnen Agent ausführen.

Tipp: Mit `dpkg -s htop >/dev/null 2>&1 || apt-get install -y htop` installiert das Skript nur, was noch fehlt, und lässt sich gefahrlos wiederholen.

### Software deinstallieren

Auch das Entfernen läuft über ein Skript. Ausführung und Kontrolle sind wie bei der Installation:

```bash
DEBIAN_FRONTEND=noninteractive apt-get remove -y htop
```

`apt-get purge` entfernt zusätzlich die Konfigurationsdateien. Ein Software-Inventar mit Deinstallations-Aktion in der Oberfläche ist noch in Planung.

## Loslegen

- **[Quick Start](quick-start.md)** – Server per Docker Compose starten und den ersten Agent einbinden
- **[Anmeldung per OIDC](oidc.md)** – Single Sign-on mit Authentik, Keycloak, Authelia & Co. einrichten
- [`docker/docker-compose.yml`](docker/docker-compose.yml) und [`docker/.env.example`](docker/.env.example) – Beispiel-Setup
