# STRATO HiDrive in Proxmox VE einbinden

**Step-by-Step Anleitung via SSHFS/SFTP**

---

| | |
|---|---|
| **Zielumgebung** | Proxmox VE 8.x (Debian Bookworm) |
| **Protokoll** | SSHFS/SFTP (verschlüsselt) |
| **Zweck** | Offsite-Backup von VMs (VZDump) |

---

## Voraussetzungen

> ⚠️ **WICHTIG: Protokollpaket prüfen!**
>
> SFTP/SSH ist **NICHT** in jedem HiDrive-Tarif enthalten.
> Im Business Essential und Standard sind nur WebDAV und SMB inklusive.
> Erst ab dem **Advanced/Premium-Tarif** sind SFTP, SCP, rsync und Git dabei.
>
> **Prüfen:** HiDrive-Login → Einstellungen → Kontenverwaltung → Konteneinstellungen → „Admin- und Protokollrechte"
>
> **Falls nicht verfügbar:** Protokollpaket für ca. 5,50 €/Mon. hinzubuchen ODER Tarif upgraden.
>
> **Ohne SFTP-Zugang funktioniert diese Anleitung nicht!**

**Was du brauchst:**

- Proxmox VE mit Root-Zugang (SSH oder Webinterface → Shell)
- Strato HiDrive Account mit aktiviertem SFTP/SSH-Protokoll
- Deine HiDrive-Zugangsdaten (Benutzername + Passwort)
- Funktionierende Internetverbindung auf dem Proxmox-Host

> 💡 **IPv6 deaktiviert?**
>
> Falls IPv6 in deinem Netz intern deaktiviert ist, stellen wir sicher, dass SSHFS nur IPv4 nutzt (`AddressFamily inet` in der SSH-Config). So vermeidest du Timeouts durch fehlgeschlagene IPv6-Verbindungsversuche.

---

## Platzhalter in dieser Anleitung

| Platzhalter | Ersetzen durch | Beispiel |
|---|---|---|
| `<HIDRIVE-USER>` | Dein HiDrive-Benutzername (**Kleinbuchstaben!**) | `meier123` |
| `<HIDRIVE-ORDNER>` | Dein Backup-Ordnername im HiDrive | `proxmox-backup` |
| `<PROXMOX-IP>` | IP-Adresse deines Proxmox-Hosts | `192.168.1.100` |

---

## Durchführung

---

### Schritt 0: HiDrive-Konto vorbereiten (im Browser)

Bevor du am Proxmox arbeitest, richte dein HiDrive-Konto richtig ein:

**A) Protokolle prüfen:**

1. Melde dich an unter **https://my.hidrive.com**
2. Klicke links unten auf **„Einstellungen"** (orange hervorgehoben)
3. Gehe zu **Kontenverwaltung → Konteneinstellungen**
4. Prüfe unter **„Admin- und Protokollrechte"** ob **SFTP** und **rsync über SSH** aktiviert sind

**B) Backup-Ordner anlegen:**

1. Gehe zurück zur Hauptansicht (klicke oben links auf „HiDrive" oder das Logo)
2. Du siehst links den Bereich **„Dateien"** – klicke darauf
3. Wechsle in den Ordner **„Privat"** (das ist dein persönlicher Bereich, Pfad: `/users/<HIDRIVE-USER>`)
4. Klicke oben auf **„Neuer Ordner"**
5. Erstelle einen Ordner mit dem gewünschten Namen (z. B. `proxmox-backup`)

> 💡 **Tipp:** Der vollständige Pfad zu deinem neuen Ordner ist dann:
> `/users/<HIDRIVE-USER>/<HIDRIVE-ORDNER>`
>
> Diesen Pfad brauchst du später in der SSHFS-Konfiguration. Der Ordner unter **„Privat"** ist nur für dein Konto sichtbar – nicht für andere HiDrive-Benutzer.

> ⚠️ **Benutzername immer komplett in Kleinbuchstaben verwenden!** Strato verlangt das bei allen Verbindungsarten.

---

### Schritt 1: Repositories prüfen und SSHFS installieren

Verbinde dich per SSH mit deinem Proxmox-Host (als root) oder nutze im Webinterface **Node → Shell**.

> ⚠️ **Auf Proxmox `apt-get` statt `apt` verwenden!**
>
> `apt update` oder `apt install` kann auf Proxmox zu Fehlern führen. Verwende immer die vollständigen Befehle `apt-get update` und `apt-get install`.

> ⚠️ **Proxmox Repository-Problem:**
>
> Nach einer frischen Proxmox-Installation ist standardmäßig nur das **Enterprise-Repository** aktiv.
> Ohne gültige Subscription schlägt `apt-get update` mit `401 Unauthorized` fehl!
>
> **Prüfe zuerst** ob deine Repos korrekt eingerichtet sind:

```bash
cat /etc/apt/sources.list.d/pve-enterprise.list
```

Falls dort eine **nicht auskommentierte** Zeile mit `enterprise.proxmox.com` steht und du **keine Subscription** hast, musst du zuerst umstellen:

**Option A – Über das Webinterface (empfohlen):**

1. Gehe zu **Dein Node → Updates → Repositories**
2. Markiere das **Enterprise-Repository** und klicke **Disable**
3. Klicke auf **Add** → wähle **No-Subscription** → **Add**

**Option B – Per Kommandozeile:**

```bash
# Enterprise-Repo deaktivieren (auskommentieren)
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list

# No-Subscription-Repo hinzufügen (falls noch nicht vorhanden)
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
```

> 💡 **Hinweis:** Falls du Proxmox 9 (Trixie) nutzt, ersetze `bookworm` durch `trixie`.
> Prüfe deine Version mit: `pveversion`

**Jetzt kannst du aktualisieren und SSHFS installieren:**

```bash
apt-get update
apt-get install -y sshfs
```

Prüfe, ob FUSE verfügbar ist:

```bash
ls -la /dev/fuse
```

Wenn die Ausgabe `/dev/fuse` zeigt (z. B. `crw-rw-rw- 1 root root ...`), ist FUSE aktiv.
Falls `/dev/fuse` nicht existiert:

```bash
modprobe fuse
```

> 💡 **Hinweis:** Bei der Installation von `sshfs` wird `fuse` durch `fuse3` ersetzt. Das ist normal – `pve-cluster` und `ceph-fuse` melden kurz eine Dependency-Warnung, funktionieren aber weiterhin.
> `lsmod | grep fuse` kann dabei leer bleiben – das ist kein Fehler. Entscheidend ist, dass `/dev/fuse` existiert.

---

### Schritt 2: Mountpoint erstellen

```bash
mkdir -p /mnt/hidrive
```

---

### Schritt 3: SSH-Schlüsselpaar erzeugen (ohne Passphrase)

Für den automatischen Mount nach einem Reboot brauchen wir einen SSH-Key ohne Passphrase. Wir erstellen einen dedizierten Key nur für HiDrive:

```bash
ssh-keygen -t ed25519 -f /root/.ssh/id_hidrive -N "" -C "proxmox-hidrive"
```

| Parameter | Bedeutung |
|---|---|
| `-t ed25519` | Moderner, sicherer Schlüsseltyp (von Strato unterstützt) |
| `-f /root/.ssh/id_hidrive` | Eigene Datei, damit der Standard-Key nicht überschrieben wird |
| `-N ""` | Leere Passphrase → automatischer Login möglich |
| `-C "proxmox-hidrive"` | Kommentar zur Identifikation |

Zeige den öffentlichen Schlüssel an – du brauchst ihn gleich:

```bash
cat /root/.ssh/id_hidrive.pub
```

**Kopiere die gesamte Ausgabe** (eine Zeile, beginnt mit `ssh-ed25519`).

> 💡 **Key-Datei mit WinSCP herunterladen:**
>
> Die Datei `id_hidrive.pub` liegt im versteckten Ordner `/root/.ssh/`. Um sie in WinSCP zu sehen:
>
> 1. Verbinde dich per WinSCP mit deinem Proxmox-Host
> 2. Versteckte Dateien einblenden: **Optionen → Einstellungen → Listenfenster → „Versteckte Dateien anzeigen (Strg+Alt+H)"** anhaken → OK
> 3. Navigiere zu `/root/.ssh/`
> 4. Ziehe die Datei `id_hidrive.pub` per **Drag & Drop** auf deinen Desktop
>
> Diese Datei brauchst du im nächsten Schritt zum Hochladen bei Strato.

---

### Schritt 4: SSH Public Key bei Strato HiDrive hinterlegen

1. Logge dich ein unter **https://my.hidrive.com**
2. Klicke links unten auf **„Einstellungen"** (orange hervorgehoben)
3. Scrolle auf der Einstellungsseite nach unten bis zum Abschnitt **„OpenSSH"**
4. Klicke auf **„SSH Schlüssel hinterlegen"**
5. Erstelle auf deinem PC eine Textdatei mit dem kopierten Key-Inhalt aus Schritt 3 (Dateiname z. B. `id_hidrive.pub`)
6. Wähle die Datei aus und speichere

> ⚠️ **Wichtig:**
> Der Upload erwartet eine Datei, du kannst den Key nicht einfach einfügen. Speichere ihn zuerst lokal als `.pub`-Datei ab.
> Strato unterstützt neben RSA auch ed25519, ecdsa und weitere Typen.

---

### Schritt 5: SSH-Config auf dem Proxmox erstellen

Damit SSHFS den richtigen Key verwendet und IPv4 erzwingt, legen wir eine SSH-Config an:

```bash
nano /root/.ssh/config
```

Füge folgenden Inhalt ein (**ersetze `<HIDRIVE-USER>` durch deinen echten HiDrive-Benutzernamen in Kleinbuchstaben!**):

```
Host hidrive
    HostName sftp.hidrive.strato.com
    User <HIDRIVE-USER>
    Port 22
    IdentityFile /root/.ssh/id_hidrive
    AddressFamily inet
    StrictHostKeyChecking accept-new
    ServerAliveInterval 30
    ServerAliveCountMax 3
```

> ⚠️ **Deinen Benutzernamen findest du so:**
>
> ```bash
> sftp hidrive
> ls users/
> exit
> ```
> Der angezeigte Ordnername unter `users/` ist dein Benutzername.
> Du kannst ihn auch im HiDrive-Webinterface sehen: **Einstellungen** → oben steht dein Kontoname.

> 🔧 **Erklärung der Optionen:**
>
> | Option | Bedeutung |
> |---|---|
> | `AddressFamily inet` | Erzwingt IPv4 (wichtig wenn IPv6 intern deaktiviert ist!) |
> | `StrictHostKeyChecking accept-new` | Akzeptiert den Server-Key beim ersten Verbinden automatisch |
> | `ServerAliveInterval 30` | Sendet alle 30 Sek. ein Keep-Alive, damit die Verbindung nicht abbricht |
> | `ServerAliveCountMax 3` | Nach 3 fehlgeschlagenen Keep-Alives wird die Verbindung getrennt |

Setze die richtigen Berechtigungen:

```bash
chmod 600 /root/.ssh/config
chmod 600 /root/.ssh/id_hidrive
```

---

### Schritt 6: Erste manuelle Verbindung testen

Teste zuerst die SSH-Verbindung direkt (nicht SSHFS), um den Host-Key zu akzeptieren:

```bash
sftp hidrive
```

Bei Erfolg siehst du den `sftp>` Prompt. Tippe `ls` um deine Dateien zu sehen, dann `exit` zum Beenden.

> ❌ **Falls Fehler auftreten:**
>
> | Fehler | Lösung |
> |---|---|
> | **Permission denied** | SSH-Key wurde nicht korrekt bei Strato hinterlegt. Prüfe Schritt 3 und 4. |
> | **Connection timed out** | SFTP-Protokoll ist nicht aktiviert. Prüfe dein HiDrive-Paket. |
> | **Network unreachable** | Internetverbindung prüfen. Test: `ping -4 sftp.hidrive.strato.com` |
> | **Host key verification failed** | Alten Eintrag löschen: `ssh-keygen -R sftp.hidrive.strato.com` |

---

### Schritt 7: SSHFS manuell mounten und testen

Jetzt testen wir den eigentlichen SSHFS-Mount:

```bash
sshfs hidrive:/users/<HIDRIVE-USER>/<HIDRIVE-ORDNER> /mnt/hidrive \
  -o allow_other,reconnect,ServerAliveInterval=30,ServerAliveCountMax=3
```

Prüfe ob der Mount funktioniert:

```bash
df -h /mnt/hidrive
ls -la /mnt/hidrive
```

Du solltest die Speichergröße deines HiDrive-Pakets sehen.

Teste Schreibzugriff:

```bash
touch /mnt/hidrive/testdatei.txt
ls -la /mnt/hidrive/testdatei.txt
rm /mnt/hidrive/testdatei.txt
```

> ✅ **Erfolgskontrolle:**
> Wenn `df` die richtige Größe zeigt und du eine Datei anlegen und löschen konntest, funktioniert alles!

Unmounte testweise wieder und gehe dann zum nächsten Schritt:

```bash
umount /mnt/hidrive
```

---

### Schritt 8: FUSE für allow_other konfigurieren

Damit auch Proxmox-Dienste (die nicht als root laufen) auf den Mount zugreifen können:

```bash
nano /etc/fuse.conf
```

Entferne das `#` vor der Zeile `user_allow_other` (entkommentieren). Die Zeile muss so aussehen:

```
user_allow_other
```

Alternativ per Einzeiler:

```bash
sed -i 's/#user_allow_other/user_allow_other/' /etc/fuse.conf
```

---

### Schritt 9: Systemd-Service erstellen (statt fstab)

> 💡 **Warum Systemd statt fstab?**
>
> SSHFS in der fstab ist ein bekanntes Problem – der Mount wird oft versucht, bevor das Netzwerk bereit ist.
> Mit einem Systemd-Service können wir exakt steuern, dass der Mount erst nach dem Netzwerk-Start erfolgt.

> ⚠️ **NICHT `systemd-networkd-wait-online.service` aktivieren!**
>
> Proxmox nutzt `networking.service` (ifupdown), **nicht** systemd-networkd.
> Das Aktivieren von `systemd-networkd-wait-online.service` führt dazu, dass **Proxmox beim Booten minutenlang hängt oder gar nicht mehr hochfährt!**

**Service erstellen:**

```bash
nano /etc/systemd/system/mnt-hidrive-mount.service
```

Inhalt (alles einfügen, **Platzhalter ersetzen!**):

```ini
[Unit]
Description=Mount Strato HiDrive via SSHFS
After=networking.service
Wants=networking.service

[Service]
Type=oneshot
ExecStartPre=/bin/sleep 15
ExecStart=/usr/bin/sshfs hidrive:/users/<HIDRIVE-USER>/<HIDRIVE-ORDNER> /mnt/hidrive -o allow_other,reconnect,ServerAliveInterval=30,ServerAliveCountMax=3,_netdev
ExecStop=/bin/umount /mnt/hidrive
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

> 🔧 **Erklärung:**
>
> | Option | Bedeutung |
> |---|---|
> | `After=networking.service` | Wartet bis Proxmox das Netzwerk konfiguriert hat |
> | `ExecStartPre=/bin/sleep 15` | Zusätzliche 15 Sek. Wartezeit damit die Internetverbindung steht |
> | `RemainAfterExit=yes` | Service bleibt als "active" markiert obwohl sshfs im Hintergrund läuft |
> | `ExecStop` | Sauberes Unmounten beim Herunterfahren |

**Aktivieren und starten:**

```bash
systemctl daemon-reload
systemctl enable mnt-hidrive-mount.service
systemctl start mnt-hidrive-mount.service
```

Prüfe den Status:

```bash
systemctl status mnt-hidrive-mount.service
df -h /mnt/hidrive
```

Die Ausgabe sollte `active (exited)` und `status=0/SUCCESS` zeigen, sowie das HiDrive mit der korrekten Größe.

---

### Schritt 10: Als Proxmox Storage hinzufügen

Jetzt binden wir den Mount als Storage in Proxmox ein:

1. Öffne das Proxmox-Webinterface: **https://\<PROXMOX-IP\>:8006**
2. Gehe zu **Datacenter → Storage → Add → Directory**
3. Fülle die Felder aus:

| Feld | Wert |
|---|---|
| **ID** | `hidrive` |
| **Directory** | `/mnt/hidrive` |
| **Content** | `Backup` (aus dem Dropdown wählen) |
| **Enable** | ✅ Haken setzen |
| **Shared** | ❌ Haken NICHT setzen (nur 1 Node) |
| **Max Backups** | Nach Wunsch (z. B. 3–5) |

4. Klicke auf **Add**

> 📋 **Hinweis zu `is_mountpoint`:**
>
> Optional kannst du in `/etc/pve/storage.cfg` die Option `is_mountpoint 1` hinzufügen.
> Damit erkennt Proxmox, dass der Storage extern gemountet wird und zeigt ihn als „offline" an, wenn der Mount fehlt – statt einen Fehler zu werfen.

So sieht der Eintrag in `/etc/pve/storage.cfg` aus (wird automatisch erstellt, hier nur zur Kontrolle):

```
dir: hidrive
    path /mnt/hidrive
    content backup
    maxfiles 5
    is_mountpoint 1
```

Falls du `is_mountpoint 1` ergänzen möchtest:

```bash
nano /etc/pve/storage.cfg
```

---

### Schritt 11: Backup-Job einrichten

Richte jetzt einen Backup-Job ein, der deine VMs auf das HiDrive sichert:

1. Gehe zu **Datacenter → Backup → Add**
2. Wähle als **Storage:** `hidrive`
3. Wähle den **Zeitplan** (z. B. täglich um 03:00 Uhr nachts)
4. Wähle die **VMs/CTs** die gesichert werden sollen
5. Setze **Mode:** `Snapshot` (empfohlen – kein Stopp der VM nötig)
6. Setze **Compression:** `ZSTD` (schnell und gute Kompression)
7. Setze **Retention:** Keep Last = 3–5 (je nach verfügbarem Platz)

> ⚠️ **Bedenke: Upload-Geschwindigkeit!**
>
> HiDrive ist über das Internet angebunden. Das Backup dauert je nach Upload-Geschwindigkeit deutlich länger als auf lokalen Speicher!
> Plane den Backup-Job so, dass er in ruhigen Zeiten läuft (z. B. nachts).
>
> **Tipp:** Teste erst mit einer kleinen VM, bevor du alle VMs einplanst. So siehst du, wie lange es dauert.

---

### Schritt 12: Reboot-Test

**Dieser Schritt ist essenziell!** Starte den Proxmox-Host neu und prüfe, ob alles automatisch funktioniert:

```bash
reboot
```

Nach dem Neustart prüfen (ca. **30 Sekunden warten** wegen dem eingebauten sleep 15):

```bash
# Systemd-Status prüfen
systemctl status mnt-hidrive-mount.service

# Mount-Status prüfen
df -h /mnt/hidrive
mount | grep hidrive
```

Prüfe auch im Proxmox-Webinterface unter **Datacenter → Storage** ob **hidrive** als „active" angezeigt wird.

> ⚠️ **Proxmox zeigt nach Reboot kurzzeitig falsche Größe!**
>
> Das ist normal – nach einem Reboot dauert es ca. 15–30 Sekunden bis der SSHFS-Mount steht.
> Bis dahin zeigt Proxmox die Größe der darunterliegenden lokalen Partition an.
> Sobald der Mount aktiv ist, zeigt ein **F5** im Webinterface die korrekte Größe.

---

## Fehlerbehebung

| Problem | Lösung |
|---|---|
| **Mount hängt / Timeout** | Prüfe IPv6: `ssh -4 -v hidrive` – sollte durch die Config schon auf IPv4 forciert sein. Prüfe Firewall/NAT. |
| **Permission denied bei SFTP** | SSH-Key nochmal prüfen. Ggf. Key neu erstellen und hochladen. Benutzernamen in Kleinbuchstaben! |
| **Permission denied beim Mount** | Falscher Benutzername oder Ordner im sshfs-Befehl. Benutzername muss **Kleinbuchstaben** sein! |
| **Storage zeigt 0 bytes** | Mount existiert, aber Verbindung unterbrochen. `systemctl restart mnt-hidrive-mount.service` |
| **Transport endpoint not connected** | Verbindung abgebrochen. `umount -l /mnt/hidrive && systemctl restart mnt-hidrive-mount.service` |
| **Backup bricht ab** | Upload zu langsam / Timeout. Prüfe Netzwerk. Teste: `dd if=/dev/zero bs=1M count=100 of=/mnt/hidrive/speedtest && rm /mnt/hidrive/speedtest` |
| **Nach Reboot kein Mount** | `systemctl status mnt-hidrive-mount.service` prüfen. Falls `failed`: Netzwerk war noch nicht bereit – den `sleep`-Wert in der Service-Datei erhöhen (z. B. von 15 auf 30). |
| **Proxmox Storage offline** | `is_mountpoint 1` in `storage.cfg` setzen. Dann zeigt Proxmox den Status korrekt an. |
| **Proxmox hängt beim Booten** | Falls `systemd-networkd-wait-online.service` aktiviert wurde: **NICHT verwenden auf Proxmox!** Deaktivieren: `systemctl disable systemd-networkd-wait-online.service` – Proxmox nutzt `networking.service`, nicht systemd-networkd. |

**Logs prüfen:**

```bash
journalctl -u mnt-hidrive-mount.service -n 50 --no-pager
```

---

## Zusammenfassung aller Befehle

Hier die komplette Befehlsfolge zum schnellen Nachschlagen (**Platzhalter anpassen!**):

```bash
# 1. Repos prüfen (ohne Subscription!)
# Enterprise-Repo deaktivieren falls nötig:
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
# No-Subscription-Repo hinzufügen (bookworm für PVE 8, trixie für PVE 9):
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list

# 2. Pakete installieren
apt-get update && apt-get install -y sshfs

# 3. Mountpoint erstellen
mkdir -p /mnt/hidrive

# 4. SSH-Key erstellen
ssh-keygen -t ed25519 -f /root/.ssh/id_hidrive -N "" -C "proxmox-hidrive"
cat /root/.ssh/id_hidrive.pub   # → bei Strato HiDrive hinterlegen!

# 5. SSH-Config anlegen
cat << 'EOF' > /root/.ssh/config
Host hidrive
    HostName sftp.hidrive.strato.com
    User <HIDRIVE-USER>
    Port 22
    IdentityFile /root/.ssh/id_hidrive
    AddressFamily inet
    StrictHostKeyChecking accept-new
    ServerAliveInterval 30
    ServerAliveCountMax 3
EOF
chmod 600 /root/.ssh/config
chmod 600 /root/.ssh/id_hidrive

# 6. Verbindung testen
sftp hidrive

# 7. FUSE konfigurieren
sed -i 's/#user_allow_other/user_allow_other/' /etc/fuse.conf

# 8. Systemd-Service erstellen
cat << 'EOF' > /etc/systemd/system/mnt-hidrive-mount.service
[Unit]
Description=Mount Strato HiDrive via SSHFS
After=networking.service
Wants=networking.service
[Service]
Type=oneshot
ExecStartPre=/bin/sleep 15
ExecStart=/usr/bin/sshfs hidrive:/users/<HIDRIVE-USER>/<HIDRIVE-ORDNER> /mnt/hidrive -o allow_other,reconnect,ServerAliveInterval=30,ServerAliveCountMax=3,_netdev
ExecStop=/bin/umount /mnt/hidrive
RemainAfterExit=yes
[Install]
WantedBy=multi-user.target
EOF

# 9. Aktivieren und testen
systemctl daemon-reload
systemctl enable mnt-hidrive-mount.service
systemctl start mnt-hidrive-mount.service
df -h /mnt/hidrive

# 10. Reboot-Test!
reboot
```

---

## Wartung

Dieses Setup läuft **dauerhaft ohne Wartung**:

- Der SSH-Key hat **kein Ablaufdatum**
- Die Systemd-Unit überlebt Updates und Reboots
- Der Proxmox Storage-Eintrag bleibt bestehen

**Was das Setup kaputt machen könnte:**

- **Proxmox Major-Upgrade** (z. B. PVE 8 → 9) – danach ggf. `apt-get install -y sshfs` erneut ausführen
- **HiDrive-Vertrag ausgelaufen / gekündigt**
- **Strato ändert den SFTP-Servernamen** (sehr unwahrscheinlich)

**Empfehlung:** Ab und zu im Proxmox unter **Datacenter → Backup** prüfen, ob die Backups erfolgreich durchlaufen.

---

## Lizenz

MIT License – frei nutzbar und anpassbar.
