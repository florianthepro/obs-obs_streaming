<details>
<summary>Allgemein</summary>

```powershell
[Convert]::ToBase64String((1..24 | % {Get-Random -Max 256}))
```
save as`<PASS>`

</details>

<details>
<summary>Server</summary>

PowerShell **als Administrator**:

```powershell
New-NetFirewallRule -DisplayName "SRT-In" -Direction Inbound -Protocol UDP -LocalPort 9000 -RemoteAddress <ip-local> -Action Allow
```

Prüfen:

```powershell
Get-NetFirewallRule -DisplayName "SRT-In" | Get-NetFirewallAddressFilter
```

OBS → **Quellen** → **+** → **Medienquelle** → Name `SRT-In` → OK.

Einstellungen im Dialog:

| Feld | Wert |
|---|---|
| Lokale Datei | **Haken entfernen** |
| Eingabe | <`srt://0.0.0.0:9000?mode=listener&latency=400000&passphrase=<PASS>&pbkeylen=32`> |
| Eingabeformat | `mpegts` |
| Netzwerkpuffer | Minimum (1 MB) |
| Wiedergabe neu starten, wenn Quelle aktiv wird | **aus** |
| Datei schließen, wenn inaktiv | **aus** |
| Bei Wiedergabeende Quelle abschalten | **aus** |

</details>

<details>
<summary>Local</summary>

Upload messen (z. B. speedtest).

Bitrate = **70 %** davon.
---
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

</details>

<details>
<summary>Rollentausch</summary>

Wenn kein Port freigegeben werden kann
Remote hinter NAT, lokal erreichbar: Modi vertauschen.

Lokal (Stream-Server):
```text
srt://0.0.0.0:9000?mode=listener&latency=400000&passphrase=<PASS>&pbkeylen=32
```
Remote (Medienquelle):
```text
srt://<LOCAL_IP>:9000?mode=caller&latency=400000&passphrase=<PASS>&pbkeylen=32
```

</details>
