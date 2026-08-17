# Installation auf der Synology

Anleitung für den Weg über den **Container Manager** – ohne Git-Klon und ohne SSH. Das Image wird fertig von Docker Hub geladen; dieses Repository liefert nur die Konfiguration drumherum.

Wer per SSH arbeitet, findet den kürzeren Weg unter „Build & Start" in der [README](README.md).

**Voraussetzungen:** DSM 7.2+ mit Container Manager, ein Admin-Konto, eine Domain in Cloudflare. Am Router ist nichts freizugeben, in der DSM-Firewall nichts zu ändern – der Tunnel baut ausschließlich ausgehend auf.

## 1 · Tunnel in Cloudflare anlegen

1. Auf `one.dash.cloudflare.com` anmelden, Account mit der Domain wählen.
2. **Networks → Tunnels → Create a tunnel**.
3. Connector-Typ **Cloudflared** → **Next**.
4. Namen vergeben (z. B. `synology`) → **Save tunnel**. Der Name erscheint nur im Dashboard.
5. Cloudflare zeigt einen Installationsbefehl an:

   ```
   cloudflared tunnel --no-autoupdate run --token eyJhIjoiN2Y4...
   ```

   Benötigt wird **nur der Token ab `eyJ`**, nicht der ganze Befehl.

> **Achtung:** Der Token ist mehrere hundert Zeichen lang und bricht in der Anzeige um. Unvollständig kopiert startet der Container normal, der Tunnel bleibt aber auf *Inactive* – der häufigste Fehler überhaupt. Die Seite offen lassen, Schritt 4 führt zurück.

## 2 · Projekt im Container Manager anlegen

1. Falls nötig: **Paket-Zentrum** → „Container Manager" installieren. Dabei entsteht der freigegebene Ordner `docker` unter `/volume1/docker`.
2. Container Manager → **Projekt** → **Erstellen**.
3. Projektname: `intnet-cloudflare-tunnel` – nur Kleinbuchstaben, Ziffern und Bindestriche, da Compose daraus interne Bezeichner ableitet.
4. Pfad: **Durchsuchen** → im Ordner `docker` einen Unterordner gleichen Namens anlegen und auswählen.
5. Quelle: **„Docker Compose erstellen"** wählen, nicht „Datei hochladen". Die Bezeichnung variiert zwischen DSM-Versionen leicht; gemeint ist die Option, YAML direkt einzutippen.
6. Den Inhalt der [`docker-compose.yml`](docker-compose.yml) dieses Repositories einfügen und die Zeile

   ```yaml
   TUNNEL_TOKEN: ${CF_TUNNEL_TOKEN:?set CF_TUNNEL_TOKEN in .env}
   ```

   durch den Token aus Schritt 1 ersetzen:

   ```yaml
   TUNNEL_TOKEN: "eyJhIjoi..."
   ```

7. Die Seite zu den Web-Portal-Einstellungen betrifft die Web Station und bleibt leer. Mit **Weiter** bis **Fertig**.

Container Manager lädt nun das Image (rund 28 MB) und startet den Container.

> Dass der Token auf diesem Weg direkt in der Compose-Datei steht statt in einer `.env`, ist unbedenklich: Der Ordner ist kein Git-Klon, es gibt nichts, wohin er versehentlich gelangen könnte. Auf der NAS liegt er in beiden Varianten im Klartext.

## 3 · Prüfen, ob der Tunnel steht

„Container läuft" und „Tunnel verbunden" sind zwei verschiedene Zustände.

Container Manager → **Projekt** → `intnet-cloudflare-tunnel` → **Protokoll**. Erwartet werden **vier** Verbindungen, verteilt auf zwei Rechenzentren:

```
INF Registered tunnel connection connIndex=0 ...
INF Registered tunnel connection connIndex=1 ...
INF Registered tunnel connection connIndex=2 ...
INF Registered tunnel connection connIndex=3 ...
```

Gegenprobe im Dashboard: Zero Trust → **Networks → Tunnels** → Status **HEALTHY**.

Zwei Meldungen sind erwartbar und kein Fehler:

- eine Warnung, dass die GID nicht in `ping_group_range` liegt. Sie betrifft ausschließlich ICMP-Proxying (Ping durch den Tunnel) und entsteht, weil das Image bewusst als UID 65532 läuft. HTTP-Ingress ist nicht betroffen.
- Hinweise auf fehlende Ingress-Regeln, solange noch kein Public Hostname existiert.

## 4 · Public Hostname anlegen

Das Routing wird nicht in der Compose-Datei konfiguriert, sondern ausschließlich im Dashboard.

Zero Trust → **Networks → Tunnels** → Tunnel → **Public Hostname** → **Add a public hostname**:

| Feld | Wert | Anmerkung |
|---|---|---|
| Subdomain | `app` | frei wählbar |
| Domain | `deine-domain.de` | aus der Liste |
| Path | leer | nur für Unterpfade |
| Service Type | `HTTP` | nicht HTTPS – die Strecke ist containerintern |
| URL | `myapp:8080` | `container_name` der App, keine IP |

> Bei **URL** gehört der `container_name` der Ziel-App hinein – **nicht** `localhost` und **nicht** die NAS-IP. Die Auflösung übernimmt Dockers internes DNS im gemeinsamen Netz; mit `localhost` würde cloudflared sich selbst adressieren.

Den DNS-Eintrag legt Cloudflare automatisch an.

## 5 · App anschließen

Das Tunnel-Projekt muss laufen – es erzeugt das Netz `cloudflare-tunnel`. Zur Einbindung siehe „App anschließen" in der [README](README.md).

Hatte die App bisher einen `ports:`-Block, muss er entfernt werden. Ein veröffentlichter Host-Port macht sie weiterhin direkt über die NAS-IP erreichbar, an Cloudflare vorbei – der Sinn des Aufbaus ist, dass es genau einen Eingang gibt.

Für jede weitere App einen weiteren Public Hostname nach Schritt 4 anlegen; ein Tunnel bedient beliebig viele.

## Aktualisieren

Projekt → Compose-Datei bearbeiten → Tag in der `image:`-Zeile anheben → speichern → **Aktion → Neu erstellen**. Das neue Image wird dabei automatisch geladen.

Der Auto-Updater ist per `--no-autoupdate` deaktiviert, damit der Versions-Pin verbindlich bleibt. Versionswechsel passieren ausschließlich über den Tag.

## Fehlerbilder

| Symptom | Ursache und Behebung |
|---|---|
| `nanoCPUs cannot be set as your kernel does not support cpu cfs scheduler` | Ein `cpus:`-Limit in der Compose-Datei. Synology-Kernel sind meist ohne `CONFIG_CFS_BANDWIDTH` gebaut. Zeile entfernen bzw. durch `cpu_shares` ersetzen – die Fassung in diesem Repo tut das bereits. |
| Container läuft, Dashboard zeigt *Inactive* | Token unvollständig kopiert. Compose-Datei öffnen, Token vollständig neu einsetzen, Projekt neu erstellen. |
| **502 Bad Gateway** | App hängt nicht am Netz `cloudflare-tunnel`; `container_name` weicht von der URL im Dashboard ab; falscher Port; oder die App lauscht intern auf `127.0.0.1` statt `0.0.0.0`. |
| App startet nicht, Netz nicht gefunden | Das Tunnel-Projekt läuft nicht – es erzeugt `cloudflare-tunnel` und muss zuerst gestartet sein. |
| App erreicht kein Internet | Kein Fehler, sondern Absicht (`internal: true`). Anwendungen mit ausgehendem Bedarf – OAuth, SMTP, externe APIs – brauchen zusätzlich ein eigenes Egress-Netz. |
| Ressourcen-Limits wirkungslos | Älteres DSM mit dem alten Docker-Paket statt Container Manager: dort fehlt Compose v2, `mem_limit` und `pids_limit` werden stillschweigend ignoriert. |

## Warum „Projekt" und nicht „Container erstellen"

Der naheliegende Weg über **Registry → Image herunterladen → Container erstellen** führt zu einem laufenden, aber deutlich schwächeren Aufbau: Der Einzelcontainer-Assistent kann weder Capabilities entziehen (`cap_drop`) noch `no-new-privileges` setzen noch das Logging begrenzen, und der Netz-Split müsste von Hand nachgebaut werden. Genau dieser Split ist das Sicherheitsmerkmal des Setups.

Das Image ist in beiden Fällen dasselbe fertige von Cloudflare – der Unterschied liegt vollständig in der Konfiguration drumherum.
