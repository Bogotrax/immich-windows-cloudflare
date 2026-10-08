# Immich auf Windows (Docker Desktop) hinter einem Cloudflare Tunnel

Funktionierendes Setup mit fünf Containern: Immich-Server, Machine Learning, Postgres (VectorChord), Redis und cloudflared.
Läuft ohne Portfreigabe im Router, erreichbar über deine eigene Domain per Cloudflare Tunnel.

## Getestet mit (Stand 2026-10-08)

| Komponente | Version |
|---|---|
| Windows + Docker Desktop (WSL2-Backend) | Docker 29.8.2, Compose v5.5.1 |
| Immich Server | **v3.2.0** (gepinnt) |
| Postgres | `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0` |
| Machine Learning | `ghcr.io/immich-app/immich-machine-learning:v3.2.0` |
| Redis | `redis:7-alpine` |
| cloudflared | `latest` |

Das Repo-Paket (Compose mit `.env`) wurde am 2026-10-08 in einem frischen Ordner mit neuen Volumes durchgespielt:
Container `Up`, Server `healthy restarts=0`, Ping-Antwort `{"res":"pong"}`.
Der Machine-Learning-Container wurde danach im selben Stack ergänzt: Status `healthy`, und der Server meldet im Log
`Machine learning server became healthy`.
Nicht geprüft wurde, ob Smart Search und Gesichtserkennung mit echten Fotos funktionieren.

## Voraussetzungen

- Windows mit installiertem Docker Desktop (WSL2-Backend)
- Eine Domain, die bei Cloudflare liegt
- Einen Cloudflare-Tunnel mit Token (Anleitung unten)

## Installation

### 1. Dateien holen

```powershell
git clone <DEIN-REPO> C:\immich
cd C:\immich
```

Ohne Git: die drei Dateien (`docker-compose.yml`, `.env.example`, `README.md`) manuell nach `C:\immich` kopieren.
Dateien lieber mit Notepad anlegen als große Blöcke in PowerShell einzufügen. Das Einfügen mehrzeiliger Texte in PowerShell ging bei uns schief.

### 2. `.env` anlegen

```powershell
Copy-Item .env.example .env
notepad .env
```

`DB_PASSWORD` und `TUNNEL_TOKEN` eintragen.

### 3. Cloudflare Tunnel einrichten

Im Cloudflare Zero Trust Dashboard einen Tunnel erstellen (Typ cloudflared), den Token kopieren und in die `.env` eintragen.
Beim Tunnel einen Public Hostname anlegen:

- Hostname: `immich.deine-domain.de`
- Service: `HTTP` und `immich-server:2283`

Menünamen im Dashboard können sich ändern. Wichtig ist nur: Der Dienst muss `http://immich-server:2283` heißen, also der Container-Name, nicht `localhost`.

### 4. Starten

```powershell
docker compose up -d
```

Der erste Start dauert ein bis zwei Minuten, der Server wartet auf die Datenbank.

### 5. Prüfen

```powershell
docker compose ps
docker inspect immich-server --format "{{.State.Health.Status}} restarts={{.RestartCount}}"
curl.exe -s http://localhost:2283/api/server/ping
```

Erwartet: Alle fünf Container `Up`, Server `healthy restarts=0`, Antwort `{"res":"pong"}`.
Dann `https://immich.deine-domain.de` im Browser öffnen.

### 6. Sofort den Admin-Account anlegen

Solange kein Admin existiert, kann jeder, der die URL kennt, die Einrichtung durchführen.
Lege den Account direkt nach dem ersten Start an.

## Stolpersteine (alle bei uns aufgetreten)

**1. `release`-Tag zog eine kaputte Version (v3.3.0).**
Mit `immich-server:release` startete der Server in einer Endlosschleife mit
`Could not load the "sharp" module ... libjpeg.so.62: cannot open shared object file`.
Mit `v3.2.0` lief es. Die genaue Ursache in v3.3.0 haben wir nicht geklärt, einen passenden GitHub-Issue haben wir nicht gefunden.
Deshalb ist die Version gepinnt. Vor einem Update die Release-Notes lesen.

**2. `REDIS_HOSTNAME` fehlte.**
Laut Immich-Doku ist der Standard-Redis-Host `redis`, der Standard-DB-Host `database`.
Wenn deine Container anders heißen, musst du die Hostnamen setzen. Ohne das hing der Server beim Start
(Log: `missing 'error' handler on this Redis client`, danach `CONNECT_TIMEOUT immich-postgres:5432`).
Ob der Postgres-Timeout nur ein Folgefehler war, haben wir nicht eindeutig belegt. Nach dem Setzen von `REDIS_HOSTNAME` lief alles.

**3. pgvecto.rs wird von Immich v3 nicht mehr unterstützt.**
In den Release-Notes von v3.0.0-rc.0 steht "drop pgvecto.rs support". Das alte Image `tensorchord/pgvecto-rs` nicht mehr verwenden.
Offizielles Image laut Immich-Doku: `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`.
Dieses Image basiert auf Postgres 14. Ein Datenordner, der mit einer anderen Postgres-Version angelegt wurde (bei uns 16), gehört nicht in dieses Image.
Bei einer frischen Installation einfach `docker compose down -v` und neu starten (löscht alle Daten!).

**4. `DB_PASSWORD` wirkt nur beim ersten Start der Datenbank.**
Wer das Passwort später in der `.env` ändert, ändert es nicht in der bestehenden Datenbank.

**5. `docker compose down -v` löscht die Volumes.**
Das gilt auch für alle hochgeladenen Fotos. Nur auf einer leeren Installation verwenden.

**6. `cloudflared` hängt in einer Neustart-Schleife.**
Im Test stand `cloudflared` auf `Restarting (255)`, solange in der `.env` kein echter Token stand.
Die Immich-Container liefen trotzdem, nur der Zugriff über die Domain ging nicht.
Nach dem Eintragen des echten Tokens lief cloudflared (`Up`).
Bei Problemen: `docker logs cloudflared --tail 15` und die `.env` prüfen, ohne den Token zu posten.

**7. Ohne ML-Container steht im Server-Log `Machine learning server became unhealthy`.**
Der Server sucht den Dienst unter `http://immich-machine-learning:3003`. Der Compose-Dienst muss deshalb genau
`immich-machine-learning` heißen. Mit dem Dienst in dieser Compose-Datei wechselt die Meldung auf `became healthy`.
Prüfen: `docker logs immich-server --tail 200 | Select-String "Machine learning"`

## Updates

1. Release-Notes auf GitHub lesen (Breaking Changes!).
2. `IMMICH_VERSION` in der `.env` erhöhen.
3. `docker compose pull; docker compose up -d`

Laut Immich-Doku ist der Server nur mit der passenden Major-Version kompatibel, und Downgrades werden nicht unterstützt.
Vor jedem Update ein Backup machen (Immich-Doku: Administration, Backup and Restore).

## Offen / nicht getestet

- Smart Search und Gesichtserkennung wurden nicht mit echten Fotos getestet (nur: ML-Container ist `healthy` und vom Server erkannt).
- Backups: nicht eingerichtet.
- Handy-App: nicht getestet.

## Sicherheit

- `.env` nie committen (steht der Tunnel-Token drin). Empfohlen: `.gitignore` mit dem Eintrag `.env`.
- Wurde ein Token irgendwo gepostet oder geteilt, im Cloudflare-Dashboard einen neuen erzeugen.
