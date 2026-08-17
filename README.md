# CLOUDFLARE-TUNNEL-V1

DESOLIC-IT PROJECT (shared infrastructure)

Geteilter Cloudflare Tunnel – als eigenständiger Docker-Container auf der Synology, der als einziger Ingress für beliebig viele Anwendungen dient. Andere Container werden im Cloudflare Zero Trust Dashboard über zusätzliche Public Hostnames auf `http://<container-name>:<port>` freigegeben und hängen sich am geteilten Netzwerk `cloudflare-tunnel` ein.

Für weitere Informationen das Dokument "DESOLIC – LEITFADEN CLOUDFLARE TUNNEL" im INTNET aufrufen.

Schrittweise Installation über den Synology Container Manager – ohne Git und ohne SSH – in [INSTALL.md](INSTALL.md).

## Konfiguration

Sämtliche Einstellungen erfolgen über Umgebungsvariablen (GUI → Container → Umgebung) bzw. über die `.env`-Datei des `docker compose`-Projekts. Es ist kein Terminal/interaktive Eingabe nötig. Eine vollständige Liste steht in `.env.example`.

Das eigentliche Routing (welche Subdomain zu welchem App-Container geht) wird nicht hier konfiguriert, sondern im Cloudflare Zero Trust Dashboard → Networks → Tunnels → `<Tunnel>` → Public Hostname → Add. Als URL im Dashboard `http://<container-name>:<port>` eintragen; App-Container müssen dazu am Netzwerk `cloudflare-tunnel` hängen und einen stabilen `container_name` haben.

## App anschließen

In der App-`docker-compose.yml`:

```yaml
services:
  myapp:
    # ...
    container_name: myapp          # stabiler DNS-Name für den Tunnel
    networks:
      - cloudflare-tunnel

networks:
  cloudflare-tunnel:
    external: true
    name: cloudflare-tunnel
```

Die App bekommt dadurch keinen veröffentlichten Host-Port und hat keinen direkten Internetzugang – nur eingehend über den geteilten Tunnel erreichbar.

## Sicherheitsmerkmale

Läuft unprivilegiert (`no-new-privileges:true`, `cap_drop: ALL`) mit Ressourcen-Limits (`mem_limit`, `pids_limit`, `cpu_shares`); das cloudflared-Image ist distroless und startet standardmäßig als UID `65532` (nonroot).

Ein hartes CPU-Limit (`cpus`) ist bewusst nicht gesetzt: Es bildet auf eine CFS-Quota ab, die Synology-Kernel in der Regel nicht unterstützen – der Docker-Daemon lehnt den Container dann mit `nanoCPUs cannot be set …` ab. `cpu_shares` gewichtet stattdessen die CPU-Zeit bei Konkurrenz und kommt ohne Quota-Unterstützung aus.

Zwei Docker-Netze: `cloudflare-tunnel` ist `internal: true` – App-Container darauf haben keinen Internetzugang; nur cloudflared selbst hängt zusätzlich auf einem privaten `egress`-Bridge, um Cloudflare zu erreichen.

Nur ausgehender Tunnel – keine Portfreigaben auf dem Router, kein Host-Port veröffentlicht, kein Konflikt mit DSM auf 443/5001.

Versionsgepinntes Image (`cloudflare/cloudflared:2026.7.3`); Token nur in der `.env`, nicht im Git. Der eingebaute Auto-Updater ist per `--no-autoupdate` deaktiviert, damit der Pin verbindlich bleibt – Updates erfolgen durch Anheben des Tags und Neuerstellen des Containers.

Das Container-Log ist auf 3 × 10 MB begrenzt (`json-file`), damit es auf dem Synology-Volume nicht unbegrenzt wächst.

## Build & Start

```
cp .env.example .env
# CF_TUNNEL_TOKEN eintragen (aus Cloudflare Zero Trust → Tunnel → Install and run a connector → Docker)
docker compose up -d
```

Beim ersten Start legt Compose das Netzwerk `cloudflare-tunnel` an; alle App-Projekte, die es als `external: true` referenzieren, können danach hochgefahren werden.
