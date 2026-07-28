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

OBS → **Einstellungen → Ausgabe** → Ausgabemodus **Erweitert** → Tab **Streaming**:

| Feld | Wert |
|---|---|
| Encoder | NVENC H.264 (bzw. QSV / x264) |
| Ratensteuerung | **CBR** |
| Bitrate | 70 % des Uploads(online speed test) |
| Keyframe-Intervall | **1 s** |
| Preset | P4 / Quality |
| Profil | high |
| B-Frames | 2 |

**Einstellungen → Ausgabe → Tab Audio**: Spur 1, AAC, **160 kbit/s**.
**Einstellungen → Audio**: Abtastrate **48 kHz**, Kanäle Stereo.

**Einstellungen → Stream**:

| Feld | Wert |
|---|---|
| Dienst | **Benutzerdefiniert...** |
| Server | `srt://<REMOTE_IP>:9000?mode=caller&latency=400000&passphrase=<PASS>&pbkeylen=32` |
| Streamschlüssel | **leer lassen** |

**„Stream starten"**

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
