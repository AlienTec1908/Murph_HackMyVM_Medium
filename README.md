# Murph - HackMyVM - Medium

**Schwierigkeitsgrad:** Medium 🟡

---

## ℹ️ Maschineninformationen

*   **Plattform:** HackMyVM
*   **VM Link:** [https://hackmyvm.eu/machines/machine.php?vm=Murph](https://hackmyvm.eu/machines/machine.php?vm=Murph)
*   **Autor:** DarkSpirit
[**➡️ Vollständigen Pentest-Bericht hier ansehen**](https://alientec1908.github.io/Murph_HackMyVM_Medium/)
![Murph Machine Icon](Murph.png)

---

## 🏁 Übersicht

Dieser Bericht dokumentiert den Penetrationstest der virtuellen Maschine "Murph" von HackMyVM. Das Ziel war die Erlangung von Systemzugriff und die Ausweitung der Berechtigungen bis auf Root-Ebene. Die Maschine wies mehrere verkettete Schwachstellen auf, darunter eine unsichere Datei-Upload-Funktion, die Remote Code Execution (RCE) als Benutzer `www-data` ermöglichte. Die Privilegien-Eskalationskette führte über ein ausnutzbares SUID-Binary, das zu einer Shell als Benutzer `jen` eskalierte, gefolgt von einer Sudo-Fehlkonfiguration für `groff`, die Zugriff auf den Benutzer `pat` ermöglichte, und schließlich einer weiteren kritischen Sudo-Fehlkonfiguration für das `login`-Binary, die die direkte Erlangung von Root-Rechten ermöglichte.

---

## 📖 Zusammenfassung des Walkthroughs

Der Pentest gliederte sich in folgende Hauptphasen:

### 🔎 Reconnaissance

*   Identifizierung der Ziel-IP (192.168.2.50) im lokalen Netzwerk mittels `arp-scan`.
*   Hinzufügen des Hostnamens `murph.hmv` zur lokalen `/etc/hosts`.
*   Umfassender Portscan (`nmap`), der Port 22 (SSH - OpenSSH 8.4p1) und Port 80 (HTTP - nginx 1.18.0) als offen identifizierte.

### 🌐 Web Enumeration

*   Scan des Nginx-Webservers auf Port 80 mit `nikto`, der fehlende Sicherheits-Header und einen Hinweis auf `/wp-config.php#` (möglicherweise False Positive) identifizierte.
*   Verzeichnis-Brute-Force mit `gobuster` identifizierte `/index.html` und das Verzeichnis `/uploads` (Status 301).
*   Analyse von `/index.html` zeigte ein Formular zur Datei-Upload-Funktion (`saveit.php`) mit Parametern `filename` und `content` (GET-Methode) und einem Kommentar, der auf eine 'php' zu 'wtf' Ersetzung hindeutet.
*   Das Verzeichnis `/uploads` wurde als nicht direkt listbar (`403 Forbidden`) verifiziert, aber die Möglichkeit zum Hochladen von Dateien über `saveit.php` wurde durch einen Test-Upload (`shell.sh`) bestätigt.
*   Auslesen des Quellcodes von `saveit.php` über die `www-data` Shell bestätigte die Case-Sensitive `str_replace("php","wtf",...)` Logik und den Upload-Pfad `/var/www/html/uploads/`.

### 💻 Initialer Zugriff

*   Ausnutzung der unsicheren Datei-Upload-Funktion und Umgehung des Case-Sensitive 'php' Filters durch Hochladen einer PHP-Webshell (`<?PhP system($_GET['cmd']); ?>`) mit der Dateierweiterung `.phtml` über `saveit.php`.
*   Erfolgreiche Ausführung von Code als Benutzer `www-data` durch Aufruf der hochgeladenen Datei (`/uploads/neu.phtml`) mit dem `cmd` Parameter (`curl http://murph.hmv/uploads/neu.phtml?cmd=id`).
*   Initiierung einer stabilen Reverse Shell zum Angreifer-System mittels Netcat über die kompromittierte Webshell.
*   Erfolgreiche Erlangung einer interaktiven Shell als Benutzer `www-data`.

### 📈 Privilege Escalation

*   Von der `www-data` Shell: System-Enumeration. Prüfung der `sudo` Berechtigungen (`sudo -l`) zeigte keine NOPASSWD-Regeln für `www-data`. Enumeration von Benutzer-Home-Verzeichnissen (`/home/jen/`, `/home/pat/`). Die Datei `user.txt` wurde in `/home/jen/` gefunden, war aber für `www-data` nicht lesbar.
*   Identifizierung eines SUID/SGID-Binaries `/opt/murph` owned by `root:www-data` mittels `find / -perm -4000`.
*   String-Analyse von `/opt/murph` zeigte Hinweise auf Signalbehandlung ('Waiting SIGUSR1....') und Systemaufrufe (`setuid`, `setgid`, `system`, `/bin/bash`).
*   Ausführung von `/opt/murph` im Hintergrund (`/opt/murph &`) als `www-data`.
*   Senden des Signals `SIGUSR1` an den laufenden `/opt/murph` Prozess (`kill -SIGUSR1 <PID>`) von einer anderen Shell.
*   Erfolgreiche Privilegien-Eskalation auf Benutzer `jen` in der ursprünglichen Shell.
*   Von der `jen` Shell: Prüfung der `sudo` Berechtigungen (`sudo -l`). Gefunden: `(pat) NOPASSWD: /usr/bin/groff`.
*   Analyse des `groff`-Binaries.
*   Ausnutzung der Sudo-Berechtigung für `groff`: Erstellung einer roff-Datei (`/tmp/exploit.roff`) mit dem Makro `.sy /bin/bash` und Ausführung mittels `sudo -u pat /usr/bin/groff -U /tmp/exploit.roff` (unter Verwendung der Option `-U` zur Deaktivierung des 'sicheren Modus').
*   Erfolgreiche Privilegien-Eskalation auf Benutzer `pat`. Die erhaltene Shell war zunächst instabil.
*   Initiierung einer stabilen Reverse Shell zum Angreifer-System mittels Bash-Code von der instabilen `pat` Shell.
*   Erfolgreiche Erlangung einer stabilen Shell als Benutzer `pat`.
*   Von der `pat` Shell: Prüfung der `sudo` Berechtigungen (`sudo -l`). Gefunden: `(root) NOPASSWD: /usr/bin/login`.
*   Ausnutzung der kritischen Sudo-Berechtigung für `login`: Direkte Anmeldung als Root mittels `sudo /usr/bin/login -f root`.
*   Erfolgreiche Erlangung einer interaktiven Root-Shell.

### 🚩 Flags

*   **User Flag:** Gefunden in `/home/jen/user.txt` (lesbar nach PE auf `jen` oder Root)
    ` HMVkwXxGPGcaoPNKkH `
*   **Root Flag:** Gefunden in `/root/root.txt` (lesbar nach PE auf Root)
    ` HMVRzKuifxeTnsjOpt `

---

## 🧠 Wichtige Erkenntnisse

*   **Unsichere Datei-Uploads:** Datei-Upload-Funktionen sind kritische Angriffsflächen, insbesondere wenn sie nicht strikt validiert werden. Einfache Filter (wie Case-Sensitive `str_replace`) sind leicht zu umgehen. Upload-Verzeichnisse sollten nicht zur Code-Ausführung konfiguriert sein.
*   **SUID/SGID-Binaries:** SUID-Binaries sind ein direkter Weg zur Privilegien-Eskalation, wenn sie unsicher sind (z.B. anfällig für Signal-Handling-Exploits) oder unnötige Berechtigungen haben. Auditen und minimieren Sie SUID/SGID-Programme.
*   **Sudo Fehlkonfigurationen:** Fehlerhafte `sudo` Regeln, insbesondere solche mit `NOPASSWD`, ermöglichen die Ausführung von Befehlen mit erhöhten Rechten und sind eine Hauptursache für PE. Seien Sie extrem vorsichtig bei der Erlaubnis, Binaries wie `login`, `su`, oder auch weniger offensichtliche Tools wie `groff` mit `NOPASSWD` auszuführen.
*   **Verkettung von Schwachstellen:** Oft erfordert die Erlangung von Root das Ausnutzen einer Kette von Schwachstellen auf verschiedenen Berechtigungsebenen (www-data -> jen -> pat -> root). Die Enumeration auf jeder Ebene ist entscheidend.
*   **Schwachstellen in Systemprogrammen:** Standard-Systemprogramme (wie `groff`, `login`) können in bestimmten Konfigurationen oder Versionen Schwachstellen aufweisen, die in Kombination mit Sudo-Berechtigungen ausnutzbar sind.

---

## 📄 Vollständiger Bericht

Eine detaillierte Schritt-für-Schritt-Anleitung, inklusive Befehlsausgaben, Analyse, Bewertung und Empfehlungen für jeden Schritt, finden Sie im vollständigen HTML-Bericht:



---

*Berichtsdatum: 16. Juni 2025*
*Pentest durchgeführt von DarkSpirit*
