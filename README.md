# SRT → MediaMTX → YouTube Relay

Dieses Setup ist aus einem praktischen Feldproblem entstanden:

- **Direkter SRT-Ingest zu YouTube** ist in diesem Setup nicht nutzbar
- **Direktes H.265 per RTMP** von den ZowieTek-Encodern wurde von YouTube nicht akzeptiert
- Die **kritische Strecke** ist der erste Uplink vom Veranstaltungsort in die Hetzner-Cloud
- Vor Ort ist diese Strecke oft **bandbreitenarm, latenzempfindlich und störanfällig**
  - WLAN
  - wenig Upload
  - wechselhafte Bedingungen
  - Outdoor-/Event-Betrieb

Deshalb wird die Übertragung in **zwei Abschnitte** aufgeteilt:

1. **Encoder → Hetzner Cloud**  
   per **SRT + H.265**, um Bandbreite zu sparen und die problematische erste Strecke robuster zu machen

2. **Cloud → YouTube**  
   per **MediaMTX + ffmpeg** als Relay

Die eigentliche Optimierung betrifft also vor allem den **ersten Hop** vom Event zur Cloud.  
Dort sind **SRT** und **H.265** entscheidend.

--- 
## ZowietekBox
Quelle, klassisches sauberes SRT als Caller:
```
srt://192.168.95.18:8890?streamid=publish:table6&pkt_size=1316
```
---

## Architektur

```text
ZowieTek Encoder vor Ort
   -> SRT Push mit H.265
   -> MediaMTX in der Hetzner Cloud
   -> runOnReady startet yt-push.sh
   -> ffmpeg liest lokalen RTSP-Stream aus MediaMTX
   -> ffmpeg pusht weiter zu YouTube
```

## Ziel des Setups

Dieses Relay-Setup dient dazu:

- die erste problematische Strecke mit SRT + H.265 effizient zu überbrücken
- mehrere Tische (table1 bis table6) parallel zu verarbeiten
- die YouTube-Streamkeys sauber getrennt zu verwalten
- keine Streamkeys direkt in mediamtx.yml zu speichern
- Änderungen an den YouTube-Keys zentral an einer Stelle vorzunehmen

## meine Verzeichnisstruktur

### MediaMTX Binary
```
/usr/local/bin/mediamtx
```
### MediaMTX Konfiguration
```
/usr/local/etc/mediamtx.yml
```
Hier ist der paths:-Block definiert, der die Pfade table1 bis table6 annimmt und das Weiterleitungs-Script startet.
### YouTube Push Script
```
/usr/local/bin/yt-push.sh
```
Dieses Script:

- liest den aktuellen MediaMTX-Pfad (MTX_PATH)
- sucht den passenden YouTube-Key
- startet ffmpeg
- leitet den Stream zu YouTube weiter

### Datei mit den YouTube-Keys
```
/usr/local/etc/youtube_keys.map
```
Das ist die wichtigste Datei, wenn ein YouTube-Streamkey geändert werden soll.
**Dort steht die Zuordnung im Format:**
```text
table1=YOUTUBE_KEY_1
table2=YOUTUBE_KEY_2
table3=YOUTUBE_KEY_3
table4=YOUTUBE_KEY_4
table5=YOUTUBE_KEY_5
table6=YOUTUBE_KEY_6
```
## MediaMTX Konfiguration
Beispiel für den relevanten paths:-Block in:
```
/usr/local/etc/mediamtx.yml
```

```yaml
paths:
  "~^table([1-6])$":
    source: publisher
    runOnReady: /usr/local/bin/yt-push.sh
    runOnReadyRestart: yes

  # example:
  # my_camera:
  #   source: rtsp://my_camera

  # Settings under path "all_others" are applied to all paths that
  # do not match another entry.
  all_others:
```

Damit werden die Pfade

* table1
* table2
bis
* table6
  
automatisch vom gleichen Hook verarbeitet.

## Script: `yt-push.sh`

Pfad: `/usr/local/bin/yt-push.sh`

### Aufgabe des Scripts:

1. passende Key-Datei laden
2. den aktuellen Pfad (MTX_PATH) auswerten
3. den zugehörigen YouTube-Key aus youtube_keys.map lesen
4. ffmpeg mit dem lokalen MediaMTX-RTSP-Stream starten
5. den Stream zu YouTube weiterleiten

Beispiel:
```bash
#!/bin/sh
set -eu

KEYFILE="/usr/local/etc/youtube_keys.map"

if [ ! -f "$KEYFILE" ]; then
  echo "Key file not found: $KEYFILE" >&2
  exit 1
fi

if [ -z "${MTX_PATH:-}" ]; then
  echo "MTX_PATH is not set" >&2
  exit 1
fi

KEY="$(
  awk -F= -v path="$MTX_PATH" '
    $1 == path {
      print substr($0, index($0, "=") + 1)
      found=1
      exit
    }
    END {
      if (!found) exit 1
    }
  ' "$KEYFILE"
)" || {
  echo "No YouTube key found for path: $MTX_PATH" >&2
  exit 1
}

exec ffmpeg \
  -loglevel warning \
  -rtsp_transport tcp \
  -i "rtsp://127.0.0.1:${RTSP_PORT}/${MTX_PATH}" \
  -c copy \
  -f flv \
  "rtmp://a.rtmp.youtube.com/live2/${KEY}"
```

### Rechte
Empfohlene Rechte:
```bash
sudo chown root:root /usr/local/bin/yt-push.sh
sudo chmod 700 /usr/local/bin/yt-push.sh

sudo chown root:root /usr/local/etc/youtube_keys.map
sudo chmod 600 /usr/local/etc/youtube_keys.map

sudo chown root:root /usr/local/etc/mediamtx.yml
sudo chmod 644 /usr/local/etc/mediamtx.yml
```
Da die Dienste auf dieser Maschine als root laufen, ist diese Absicherung passend. Besser ist natürlich ein spezieller Dienst-User.
automatisch vom gleichen Hook verarbeitet.

### Log
Logs live ansehen:
```
journalctl -u mediamtx -f
```
