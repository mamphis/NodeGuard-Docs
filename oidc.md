# Anmeldung per OIDC (Single Sign-on)

NodeGuard kann sich an jeden OpenID-Connect-Provider anbinden, z. B. **Authentik**, **Keycloak**, **Authelia**, **Zitadel**, **Pocket ID** oder **Kanidm**. Auf der Login-Seite erscheint dann zusätzlich der Button **Sign in with …**. Die Anmeldung mit lokalem Passwort bleibt daneben bestehen.

NodeGuard nutzt den Authorization Code Flow mit PKCE. Die Provider-Daten (Endpunkte, Schlüssel) werden automatisch über `<issuer>/.well-known/openid-configuration` geladen.

## 1. Client beim Identity Provider anlegen

Lege beim Provider einen neuen OIDC-Client (Application) an:

| Einstellung | Wert |
| --- | --- |
| Client-Typ | *Confidential* (mit Client Secret) – oder *Public*, dann ohne Secret, nur PKCE |
| Grant Type | Authorization Code |
| Redirect URI | `http(s)://<server-adresse>/api/auth/oidc/callback` |
| Scopes | `openid profile email`, für Gruppen zusätzlich `groups` (je nach Provider) |

Die genaue Redirect URI zeigt NodeGuard in der Ersteinrichtung und unter **Settings → Sign-in** an (mit Kopier-Button). Sie muss exakt übereinstimmen – inklusive `http`/`https` und Port.

Hinweise zu einzelnen Providern:

- **Authentik:** Provider-Typ *OAuth2/OpenID Provider*. Issuer ist `https://<authentik>/application/o/<slug>/`. Gruppen stehen mit dem Scope `profile` im Claim `groups`.
- **Keycloak:** Client mit *Client authentication* an. Issuer ist `https://<keycloak>/realms/<realm>`. Für Gruppen einen *Group Membership*-Mapper mit Claim-Name `groups` anlegen oder als Groups claim `realm_access.roles` (Realm-Rollen) eintragen.
- **Authelia:** Client unter `identity_providers.oidc.clients` mit `redirect_uris` wie oben. Issuer ist die Authelia-Adresse. Für Gruppen den Scope `groups` ergänzen.

## 2. In NodeGuard einrichten

### Bei der Ersteinrichtung

1. Öffne die Web-App. Die **Ersteinrichtung** erscheint automatisch.
2. Wechsle auf den Reiter **Single sign-on**.
3. Trage **Issuer URL**, **Client ID** und ggf. **Client secret** ein. Unter *Allowed groups* kannst du festlegen, welche Gruppen sich anmelden dürfen.
4. Klicke **Save and sign in at the provider** und melde dich beim Provider an.

Das Konto, mit dem du dich anmeldest, wird zum NodeGuard-Administrator. Es hat kein lokales Passwort. NodeGuard prüft die Provider-Adresse beim Speichern; ist sie nicht erreichbar, erscheint die Fehlermeldung direkt im Formular.

Du kannst auch wie bisher ein lokales Konto anlegen und OIDC später einrichten.

### Später in den Einstellungen

Unter **Settings → Sign-in** lassen sich alle Werte ändern und OIDC ein- und ausschalten. Dort verknüpfst du auch dein bestehendes Konto (siehe unten).

### Per Umgebungsvariablen

Alternativ wird OIDC komplett über Umgebungsvariablen konfiguriert. Sobald `OIDC_ISSUER` gesetzt ist, gelten nur noch diese Werte; die Formulare in der Web-App zeigen sie dann schreibgeschützt an.

```yaml
  nodeguard:
    environment:
      OIDC_ISSUER: "https://auth.example.com/application/o/nodeguard/"
      OIDC_CLIENT_ID: "nodeguard"
      OIDC_CLIENT_SECRET: "…"
      OIDC_DISPLAY_NAME: "Authentik"
      OIDC_ALLOWED_GROUPS: "nodeguard-admins"
```

| Variable | Standard | Bedeutung |
| --- | --- | --- |
| `OIDC_ISSUER` | – | Issuer-URL des Providers. Aktiviert die Konfiguration per Umgebung. |
| `OIDC_CLIENT_ID` | – | Client ID. |
| `OIDC_CLIENT_SECRET` | leer | Client Secret. Leer = Public Client (nur PKCE). |
| `OIDC_ENABLED` | `true` | `false` blendet die OIDC-Anmeldung aus, ohne die Werte zu entfernen. |
| `OIDC_DISPLAY_NAME` | `Single sign-on` | Beschriftung des Buttons: „Sign in with …“. |
| `OIDC_SCOPES` | `openid profile email` | Angeforderte Scopes, muss `openid` enthalten. |
| `OIDC_USERNAME_CLAIM` | `preferred_username` | Claim für den Benutzernamen neuer Konten. |
| `OIDC_GROUPS_CLAIM` | `groups` | Claim mit den Gruppen, auch verschachtelt wie `realm_access.roles`. |
| `OIDC_ALLOWED_GROUPS` | leer | Kommagetrennte Gruppen. Leer = jedes Konto des Providers darf sich anmelden. |
| `OIDC_AUTO_CREATE_USERS` | `false` | Unbekannte Identitäten erhalten beim ersten Login ein eigenes Konto. |
| `OIDC_LINK_BY_USERNAME` | `false` | Unbekannte Identitäten werden mit dem gleichnamigen lokalen Konto verknüpft. |
| `OIDC_REDIRECT_URI` | automatisch | Nur nötig, wenn NodeGuard hinter einem Reverse Proxy seine öffentliche Adresse nicht erkennt. |

## 3. Zuordnung zu NodeGuard-Konten

Eine Identität beim Provider wird über das Paar *Issuer + Subject (`sub`)* eindeutig einem NodeGuard-Konto zugeordnet. Der Benutzername oder die E-Mail-Adresse allein reicht für eine Anmeldung nicht.

So kommt die Zuordnung zustande:

1. **Ersteinrichtung per OIDC** – das erste Konto wird aus der Identität angelegt.
2. **Verknüpfen in den Einstellungen** – angemeldet mit deinem lokalen Konto unter **Settings → Sign-in → Link identity** klicken und beim Provider anmelden. Danach funktioniert beides: Passwort und Single Sign-on. Verknüpfungen lassen sich dort auch wieder entfernen.
3. **Automatisch anlegen** (*Create accounts automatically*) – jede erlaubte Identität bekommt beim ersten Login ein eigenes Konto. Schränke den Zugang dann unbedingt mit *Allowed groups* ein.
4. **Nach Benutzername verknüpfen** (*Link by username*) – eine unbekannte Identität wird mit dem lokalen Konto gleichen Namens verknüpft. Nur einschalten, wenn Benutzer ihren Benutzernamen beim Provider nicht selbst wählen können.

Ohne passende Zuordnung lehnt NodeGuard die Anmeldung mit einem Hinweis ab. Alle NodeGuard-Konten haben dieselben (vollen) Rechte.

*Allowed groups* gilt für jede OIDC-Anmeldung, auch für bereits verknüpfte Konten. Wer aus der Gruppe entfernt wird, kann sich per OIDC nicht mehr anmelden.

## 4. Sitzung und Abmeldung

Nach der Anmeldung gilt dieselbe Sitzung wie beim Passwort-Login (verschlüsseltes Cookie, 7 Tage). **Sign out** beendet nur die NodeGuard-Sitzung, nicht die Sitzung beim Provider. Alle Anmeldungen, Verknüpfungen und Änderungen der Einstellungen stehen im **Audit log**.

## 5. Fehlerbehebung

| Meldung / Symptom | Ursache und Lösung |
| --- | --- |
| `Could not load the provider configuration from …` beim Speichern | Issuer-URL falsch oder vom NodeGuard-Container aus nicht erreichbar. Prüfen mit `docker compose exec nodeguard wget -qO- <issuer>/.well-known/openid-configuration`. |
| Provider meldet `invalid redirect_uri` | Die Redirect URI beim Provider stimmt nicht exakt mit der in NodeGuard angezeigten überein. Hinter einem Reverse Proxy `X-Forwarded-Proto`/`X-Forwarded-Host` setzen oder die Redirect URI explizit eintragen. |
| `No NodeGuard account is linked to this identity.` | Identität ist keinem Konto zugeordnet. Mit Passwort anmelden und unter **Settings → Sign-in** verknüpfen, oder *Create accounts automatically* einschalten. |
| `Your account is not in a group that may use NodeGuard.` | Gruppen fehlen im Token. Scope `groups` ergänzen bzw. beim Provider einen Gruppen-Mapper einrichten und den Groups claim prüfen. |
| `The sign-in request expired or was started in another browser.` | Zwischen Start und Rückkehr lagen mehr als 10 Minuten, oder Cookies wurden blockiert. Erneut versuchen; bei reinem HTTP ggf. `SESSION_COOKIE_SECURE=false` setzen. |
| `Single sign-on failed.` | Details stehen im Container-Log (`docker compose logs nodeguard`, Einträge mit `[OIDC]`), z. B. falsches Client Secret. |
| Provider mit eigenem/selbstsigniertem Zertifikat | Die CA dem Container bekannt machen: Zertifikat einbinden und `NODE_EXTRA_CA_CERTS=/pfad/zur/ca.pem` setzen. Provider per reinem HTTP werden ebenfalls unterstützt. |
| Provider ausgefallen, Anmeldung nicht möglich | Konten mit Passwort können sich weiter lokal anmelden. Notfalls `docker compose exec nodeguard reset-password` und Ersteinrichtung neu durchlaufen; die OIDC-Einstellungen bleiben erhalten. |
