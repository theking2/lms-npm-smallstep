# lms-npm-smallstep

Installationsanleitung für drei Docker-Dienste auf einem Container-Host (Synology, QNAP oder Raspberry Pi):

| Dienst | Image | Zweck | Port |
|--------|-------|-------|------|
| `step-ca` | `smallstep/step-ca` | interne Zertifizierungsstelle (CA) mit ACME | 9000 |
| `nginx-proxy` (NPM) | `jc21/nginx-proxy-manager` | Reverse Proxy mit Web-Oberfläche | 9080, 9443, 81 |
| `lms` | `lmscommunity/lyrionmusicserver` | Lyrion Music Server | 9002 (Web), 3483, 9090 |

Die Konfiguration stammt aus den Setups in [srv-nas-pi](https://github.com/theking2/srv-nas-pi) (LMS, NPM) und [srv-monitor](https://github.com/theking2/srv-monitor) (step-ca) und ist auf einen gemeinsamen Host zusammengeführt.

## Wichtige Abweichungen von den Quell-Setups

- **Port 9000:** In den Quell-Setups lief LMS auf Host A mit Port 9000 und step-ca auf Host B mit Port 9000. Auf einem gemeinsamen Host kollidieren beide, weil LMS im Host-Netzwerk läuft. Hier bekommt LMS `9002`.
- **Ports 80/443:** NAS-Systeme belegen diese Ports selbst. NPM lauscht deshalb auf `9080`/`9443`; am Router wird `80 -> 9080` und `443 -> 9443` weitergeleitet.
- **Docker-Netzwerk `proxy`:** Wie im Quell-Setup ein externes Netzwerk, das NPM mit den Backends verbindet.
- **Pfade und Besitzer:** Die Quell-Setups nutzen OpenMediaVault-Pfade (`/pool0/...`). Hier sind alle Pfade in `.env` einstellbar.

## Voraussetzungen

- Docker Engine mit Compose-Plugin (`docker compose version`)
  - Synology: Container Manager (DSM 7.2+), SSH aktiviert
  - QNAP: Container Station, SSH aktiviert
  - Raspberry Pi: Raspberry Pi OS 64 Bit, Docker CE (`curl -fsSL https://get.docker.com | sh`), Benutzer in der Gruppe `docker`
- Feste IP-Adresse für den Host
- Ein DNS-Name für die CA (zum Beispiel `ca.example.lan`), der im lokalen DNS auf den Host zeigt
- Medienordner mit `music/` und `playlists/` für LMS

## Verzeichnisse je Plattform

| | Synology | QNAP | Raspberry Pi |
|---|----------|------|--------------|
| Datenbasis | `/volume1/docker` | `/share/Container` | `/srv/docker` |
| Medien | `/volume1/media` | `/share/Multimedia` | `/srv/media` |
| Benutzer-IDs prüfen | `id <benutzer>` | `id <benutzer>` | `id` |

Ordner anlegen (Beispiel Raspberry Pi, Pfade anpassen):

```sh
sudo mkdir -p /srv/docker/{step-ca,npm,lms}
sudo chown -R 1000:1000 /srv/docker/lms /srv/docker/npm
sudo chown -R 1000:1000 /srv/docker/step-ca   # step-ca läuft im Container als UID 1000
```

## 1. Repository holen und Netzwerk anlegen

```sh
git clone https://github.com/theking2/lms-npm-smallstep.git
cd lms-npm-smallstep
docker network create proxy
```

Jeder Dienst hat einen eigenen Ordner mit `compose.yaml` und `.env.example`. In jedem Ordner:

```sh
cp .env.example .env
nano .env        # Pfade, Ports, Namen anpassen
```

Die Reihenfolge ist step-ca, NPM, LMS. Sie lässt sich auch ändern, die Dienste hängen beim Start nicht voneinander ab.

## 2. step-ca

`step-ca/.env` anpassen:

- `CA_NAME`: Name der CA (Aussteller der Zertifikate)
- `CA_DNS_NAMES`: alle Namen und IPs, unter denen Clients die CA ansprechen, zum Beispiel `ca.example.lan,localhost,192.168.1.20`. Ein Name, der hier fehlt, führt später zu TLS-Fehlern.
- `STEP_DATA`: Datenordner, enthält Schlüssel und Datenbank der CA

Start:

```sh
cd step-ca
docker compose up -d
docker compose logs -f step-ca
```

Beim ersten Start initialisiert der Container die CA und gibt im Log Fingerprint und Zugangsdaten aus. Danach:

```sh
# Passwort der CA und des Admin-Provisioners
cat ${STEP_DATA}/secrets/password

# Fingerprint des Root-Zertifikats
docker exec step-ca step certificate fingerprint /home/step/certs/root_ca.crt

# Root-Zertifikat zum Verteilen an Clients
cp ${STEP_DATA}/certs/root_ca.crt ./root_ca.crt
```

Das Passwort wird zum Ausstellen und Verwalten benötigt. Es gehört in einen Passwortmanager, `secrets/` und `root_ca_key`/`intermediate_ca_key` gehören ins Backup und nicht in ein Repository.

Test:

```sh
curl --cacert ./root_ca.crt https://ca.example.lan:9000/health
# {"status":"ok"}
```

### Clients einrichten

Auf einem Rechner mit der [`step` CLI](https://smallstep.com/docs/step-cli/installation/):

```sh
step ca bootstrap --ca-url https://ca.example.lan:9000 --fingerprint <FINGERPRINT>
step certificate install $(step path)/certs/root_ca.crt   # Root in den Systemspeicher
```

Browser und Geräte, die den Systemspeicher nicht nutzen (Firefox, Android, iOS), importieren `root_ca.crt` manuell.

### ACME

Durch `DOCKER_STEPCA_INIT_ACME=true` existiert ein ACME-Provisioner `acme`. Die Directory-URL lautet:

```
https://ca.example.lan:9000/acme/acme/directory
```

ACME-Clients (certbot, acme.sh, Caddy, Proxmox) verwenden diese URL. Bei certbot muss die CA vertraut werden:

```sh
sudo REQUESTS_CA_BUNDLE=$(step path)/certs/root_ca.crt \
  certbot certonly --standalone -d dienst.example.lan \
  --server https://ca.example.lan:9000/acme/acme/directory
```

Die Challenge muss vom Host der CA aus erreichbar sein (`http-01` auf Port 80 des Clients).

### Zertifikatslaufzeit

step-ca stellt standardmäßig Zertifikate mit 24 Stunden Laufzeit aus. Für Zertifikate, die von Hand in NPM hochgeladen werden, ist das zu kurz. Laufzeit des Provisioners verlängern:

```sh
docker exec -it step-ca step ca provisioner update admin \
  --x509-default-dur=720h --x509-max-dur=2160h
docker restart step-ca
```

Flags vor dem Einsatz mit `step ca provisioner update --help` abgleichen, sie hängen von der step-ca-Version ab. Für ACME-Clients mit automatischer Erneuerung reicht die Standardlaufzeit.

## 3. Nginx Proxy Manager

`npm/.env` anpassen (`NPM_DATA`, Ports, `PUID`/`PGID`). Start:

```sh
cd ../npm
docker compose up -d
docker compose logs -f nginx-proxy
```

Web-Oberfläche: `http://<host-ip>:81`. Bei einer neuen Installation führt NPM durch die Anlage des Administrators. Ältere Versionen starten mit `admin@example.com` / `changeme`, dieses Passwort sofort ändern.

Daten liegen in `${NPM_DATA}/data` (Konfiguration, Datenbank) und `${NPM_DATA}/letsencrypt`.

### Proxy Host für LMS anlegen

LMS läuft im Host-Netzwerk und ist deshalb nicht über den Containernamen erreichbar, sondern über die Host-IP:

| Feld | Wert |
|------|------|
| Domain Names | `lms.example.lan` |
| Scheme | `http` |
| Forward Hostname / IP | `<host-ip>` |
| Forward Port | `9002` |
| Websockets Support | an |

Der DNS-Name `lms.example.lan` muss auf den Host zeigen. Der Zugriff über NPM erfolgt dann auf Port `9443` (oder `443`, wenn der Router weiterleitet beziehungsweise NPM auf 443 gebunden ist).

### Zertifikat der step-ca in NPM verwenden

NPM bietet nach meinem Kenntnisstand in der Oberfläche keine Option für eine eigene ACME-Directory-URL. Der Weg ist ein Custom Certificate:

1. Zertifikat auf einem Rechner mit `step` ausstellen:

   ```sh
   step ca certificate lms.example.lan lms.crt lms.key
   ```

   Den Provisioner `admin` wählen und das CA-Passwort eingeben.

2. In NPM: *SSL Certificates* → *Add SSL Certificate* → *Custom*. Name vergeben, `lms.key` als Key, `lms.crt` als Certificate hochladen. Falls NPM eine Intermediate-Datei verlangt, `intermediate_ca.crt` aus `${STEP_DATA}/certs/` verwenden.
3. Im Proxy Host unter *SSL* das Zertifikat auswählen, *Force SSL* aktivieren.
4. Vor Ablauf erneuern (`step ca renew` oder Schritt 1 wiederholen) und in NPM neu hochladen. Ein Ablaufdatum im Kalender oder ein Monitoring-Check sind erforderlich, NPM erneuert Custom Certificates nicht selbst.

Für Dienste, die ACME selbst sprechen, ist der ACME-Weg aus Abschnitt 2 wartungsärmer.

## 4. Lyrion Music Server

`lms/.env` anpassen:

- `LMS_CONFIG`: Konfigurationsordner (Bibliothek, Plugins, Einstellungen)
- `MEDIA`: Ordner mit `music/` (nur lesend eingebunden) und `playlists/`
- `LMS_HTTP_PORT`: Standard `9002`, siehe Port-Konflikt oben

Start:

```sh
cd ../lms
docker compose up -d
docker compose logs -f lms
```

Web-Oberfläche: `http://<host-ip>:9002`. Beim ersten Aufruf den Einrichtungsassistenten durchlaufen und den Musikordner `/music` sowie den Playlist-Ordner `/playlist` angeben. Das Scannen einer großen Bibliothek dauert; auf einem Raspberry Pi ist das spürbar.

Firewall des Hosts muss UDP/TCP `3483`, TCP `9090` und den Web-Port für das LAN erlauben, sonst finden Player den Server nicht.

## Betrieb

Logs:

```sh
docker logs -f step-ca
docker logs -f nginx-proxy
docker logs -f lms
```

Update (je Ordner):

```sh
docker compose pull
docker compose up -d
```

Vor Updates von step-ca die Datenordner sichern. NPM und LMS migrieren ihre Datenbanken beim Update, ein Backup von `NPM_DATA` und `LMS_CONFIG` vorher ist ratsam.

Backup (Dienste kurz anhalten oder Snapshot des Dateisystems nutzen):

| Dienst | Sichern |
|--------|---------|
| step-ca | gesamter `STEP_DATA`-Ordner (Schlüssel, Passwort, Datenbank) |
| NPM | gesamter `NPM_DATA`-Ordner |
| LMS | `LMS_CONFIG` (Medien liegen separat) |

## Fehlersuche

| Symptom | Ursache und Abhilfe |
|---------|---------------------|
| `step-ca` startet nicht, `permission denied` auf `/home/step` | Ordner gehört nicht UID 1000: `sudo chown -R 1000:1000 ${STEP_DATA}` |
| `x509: certificate is valid for ..., not ...` | Name fehlt in `CA_DNS_NAMES`. Eintrag ergänzen; bei bereits initialisierter CA `DNS_NAMES` in `${STEP_DATA}/config/ca.json` anpassen |
| `step-ca` nach Neustart des Hosts nicht erreichbar | Restart-Policy prüfen: `docker inspect step-ca --format '{{.HostConfig.RestartPolicy.Name}}'` muss `unless-stopped` sein |
| Port 9000 belegt | `ss -tlnp \| grep 9000`; `CA_PORT` oder `LMS_HTTP_PORT` ändern |
| NPM startet nicht, Port 80/443 belegt | Standard ist 9080/9443; Werte in `npm/.env` prüfen |
| NPM erreicht Backend nicht | LMS läuft im Host-Netzwerk: Host-IP statt Containername verwenden. Andere Backends müssen im Netzwerk `proxy` hängen |
| LMS-Player finden den Server nicht | Host-Firewall und VLAN-/Broadcast-Grenzen prüfen (UDP 3483) |

## Lizenz

MIT, siehe [LICENSE](LICENSE).
