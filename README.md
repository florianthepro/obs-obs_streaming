# obs--obs_streaming

# OBS → OBS über Internet (SRT, verschlüsselt)

Lokaler Win11-PC sendet Bild+Ton an die entfernte Win11-Streaming-Maschine.
Transport: SRT über UDP, AES-256, ~400 ms Latenz.

## 0. Platzhalter festlegen

Einmal ausfüllen, überall gleich verwenden.

```text
<PASS>        = Passphrase, 10-79 Zeichen, auf beiden Seiten identisch
<REMOTE_IP>   = öffentliche IP der Streaming-Maschine
<LOCAL_IP>    = öffentliche IP des lokalen PCs (nur für die Firewall-Regel)
<PORT>        = 9000
```

Passphrase erzeugen (lokal, PowerShell):

```powershell
[Convert]::ToBase64String((1..24 | % {Get-Random -Max 256}))
```

---

# TEIL A — Remote (Empfänger). Zuerst einrichten.

## A1. Firewall öffnen, auf Quell-IP begrenzt

PowerShell **als Administrator** auf der Streaming-Maschine:

```powershell
New-NetFirewallRule -DisplayName "SRT-In" -Direction Inbound -Protocol UDP -LocalPort 9000 -RemoteAddress <LOCAL_IP> -Action Allow
```

*Warum:* ohne `-RemoteAddress` ist der Port für das gesamte Internet offen.

Prüfen:

```powershell
Get-NetFirewallRule -DisplayName "SRT-In" | Get-NetFirewallAddressFilter
```

## A2. Router/Cloud-Firewall

Falls die Maschine hinter NAT steht: UDP `9000` an ihre interne IP weiterleiten.
Bei VPS zusätzlich in der Cloud-Firewall (Security Group) UDP 9000 freigeben.

## A3. Medienquelle anlegen

OBS → **Quellen** → **+** → **Medienquelle** → Name `SRT-In` → OK.

## A4. Medienquelle konfigurieren

Einstellungen im Dialog:

| Feld | Wert |
|---|---|
| Lokale Datei | **Haken entfernen** |
| Eingabe | siehe Block unten |
| Eingabeformat | `mpegts` |
| Netzwerkpuffer | Minimum (1 MB) |
| Wiedergabe neu starten, wenn Quelle aktiv wird | **aus** |
| Datei schließen, wenn inaktiv | **aus** |
| Bei Wiedergabeende Quelle abschalten | **aus** |

Eingabe-URL (Passphrase einsetzen):

```text
srt://0.0.0.0:9000?mode=listener&latency=400000&passphrase=<PASS>&pbkeylen=32
```

*Erklärung:* `0.0.0.0` = auf allen Interfaces lauschen. `latency` in Mikrosekunden. `pbkeylen=32` = AES-256.

## A5. Quelle scharf lassen

Rechtsklick auf `SRT-In` → **„Deaktivieren, wenn nicht sichtbar"** darf **nicht** gesetzt sein.
Schwarzes Bild bis der Sender verbindet ist korrekt.

---

# TEIL B — Lokal (Sender)

## B1. Upload messen

Vor allem anderen: realen Upload messen (z. B. speedtest). Bitrate = **70 %** davon.

## B2. Encoder

OBS → **Einstellungen → Ausgabe** → Ausgabemodus **Erweitert** → Tab **Streaming**:

| Feld | Wert |
|---|---|
| Encoder | NVENC H.264 (bzw. QSV / x264) |
| Ratensteuerung | **CBR** |
| Bitrate | 70 % des Uploads |
| Keyframe-Intervall | **1 s** |
| Preset | P4 / Quality |
| Profil | high |
| B-Frames | 2 |

*Warum CBR:* SRT-Puffer und Netzpfad werden auf eine konstante Rate ausgelegt; VBR-Spitzen erzeugen Paketverlust.

## B3. Audio

**Einstellungen → Ausgabe → Tab Audio**: Spur 1, AAC, **160 kbit/s**.
**Einstellungen → Audio**: Abtastrate **48 kHz**, Kanäle Stereo.

## B4. Ziel eintragen

**Einstellungen → Stream**:

| Feld | Wert |
|---|---|
| Dienst | **Benutzerdefiniert...** |
| Server | siehe Block unten |
| Streamschlüssel | **leer lassen** |

```text
srt://<REMOTE_IP>:9000?mode=caller&latency=400000&passphrase=<PASS>&pbkeylen=32
```

## B5. Starten

**„Stream starten"**. Bild erscheint remote nach 1–2 s.

---

# TEIL C — Prüfen

## C1. Statistiken öffnen

Auf **beiden** Rechnern: **Ansicht → Docks → Statistiken**.

Sollwerte lokal:

```text
Verworfene Frames (Netzwerk)  = 0
Überlastete Enkodierung       = nein
Bitrate                       = stabil auf Sollwert
```

## C2. Logfile bei Problemen

**Hilfe → Logdateien → Aktuelles Log anzeigen**.
Falsche Passphrase erscheint nur hier als Connection-Reject, nicht im UI.

## C3. Fehlerbilder

| Symptom | Ursache | Maßnahme |
|---|---|---|
| Remote bleibt schwarz | Port/NAT/Firewall | A1–A2 prüfen, Teil E |
| Verbindung bricht sofort ab | Passphrase ungleich | beide URLs vergleichen |
| Ruckler, verworfene Frames | Bitrate zu hoch | Bitrate −20 % |
| Artefakte, kurze Aussetzer | Latenz zu klein | `latency` erhöhen, Teil D |
| Ton läuft vor/nach | Buffer-Versatz | D2 |

---

# TEIL D — Tuning

## D1. Latenz berechnen

RTT messen (lokal):

```powershell
ping <REMOTE_IP>
```

Regel: `latency [µs] ≥ 4 × RTT [ms] × 1000`, Minimum 120000.

```text
RTT  25 ms  ->  latency=120000   (Minimum greift)
RTT  60 ms  ->  latency=240000
RTT 100 ms  ->  latency=400000
```

Wert in **beiden** URLs setzen; SRT verwendet den höheren der beiden.

## D2. A/V-Sync

Remote: Rechtsklick Mixer → **Erweiterte Audioeigenschaften** → `SRT-In` → **Sync-Versatz** in ms anpassen.

---

# TEIL E — Wenn kein Port freigegeben werden kann

## E1. Rollentausch

Remote hinter NAT, lokal erreichbar: Modi vertauschen.

Lokal (Stream-Server):
```text
srt://0.0.0.0:9000?mode=listener&latency=400000&passphrase=<PASS>&pbkeylen=32
```
Remote (Medienquelle):
```text
srt://<LOCAL_IP>:9000?mode=caller&latency=400000&passphrase=<PASS>&pbkeylen=32
```

## E2. Beide hinter NAT

Auf beiden Seiten `mode=rendezvous`, gleicher Port, beide gleichzeitig starten.

## E3. Empfohlen: Tunnel statt offener Port

WireGuard oder Tailscale auf beiden Maschinen installieren, dann in den URLs die
Tunnel-IP statt der öffentlichen IP verwenden:

```text
srt://100.x.y.z:9000?mode=caller&latency=400000&passphrase=<PASS>&pbkeylen=32
```

Damit ist UDP 9000 nirgends öffentlich erreichbar; die Firewall-Regel aus A1 entfällt.
Passphrase trotzdem gesetzt lassen (Defense in Depth).

---

# Checkliste

```text
[ ] Passphrase erzeugt, identisch auf beiden Seiten
[ ] Firewall-Regel mit -RemoteAddress gesetzt (oder Tunnel aktiv)
[ ] Remote: Medienquelle, mode=listener, mpegts, Puffer minimal
[ ] Lokal:  Stream-Ziel, mode=caller, Streamschlüssel leer
[ ] Lokal:  CBR, Keyframe 1 s, Bitrate = 70 % Upload
[ ] latency auf beiden Seiten gleich und >= 4x RTT
[ ] Statistiken: verworfene Frames = 0
```
