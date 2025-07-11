# UnbakedPie - HackMyVM - Hard

**Schwierigkeitsgrad:** Hard 🔴

---

## ℹ️ Maschineninformationen

*   **Plattform:** HackMyVM
*   **VM Link:** [https://hackmyvm.eu/machines/machine.php?vm=UnbakedPie](https://hackmyvm.eu/machines/machine.php?vm=UnbakedPie)
*   **Autor:** DarkSpirit

![UnbakedPie Machine Icon](UnbakedPie.png)

---

## 🏁 Übersicht

Dieser Bericht dokumentiert den Penetrationstest der virtuellen Maschine "UnbakedPie" von HackMyVM. Das Ziel war die Erlangung von Systemzugriff und die Ausweitung der Berechtigungen bis auf Root-Ebene auf dem Host-System. Die Maschine wies eine kritische unsichere Python Pickle Deserialisierungs-Schwachstelle in einem Web-Cookie auf Port 5003 auf, die zum initialen Root-Zugriff in einem Docker-Container führte. Die weitere Privilegien-Eskalation umfasste einen Ausbruch aus dem Container, die horizontale Eskalation von Benutzer `ramsey` zu Benutzer `oliver` durch die Ausnutzung einer fehlerhaften Sudo-Regel in Verbindung mit einem modifizierbaren Python-Skript, und schließlich die Root-Eskalation auf dem Host durch das Hijacking eines Modulimports (`PYTHONPATH`) in einem weiteren Sudo-fähigen Python-Skript.

---

## 📖 Zusammenfassung des Walkthroughs

Der Pentest gliederte sich in folgende Hauptphasen:

### 🔎 Reconnaissance

*   Identifizierung der Ziel-IP (192.168.2.48) mittels `arp-scan`.
*   Hinzufügen des Hostnamens `unbakedpie.hmv` zur lokalen `/etc/hosts`.
*   Umfassender Portscan (`nmap`) identifizierte Port 5003 (HTTP - WSGIServer/Python 3.8.6) und Port 22 (SSH - OpenSSH) als offen.

### 🌐 Web Enumeration

*   Scan mit `nikto` meldete potenzielle Shellshock-Schwachstellen auf CGI-Pfaden und fehlendes `httponly` bei CSRF-Cookies.
*   Verzeichnis-Brute-Force mit `feroxbuster` fand `/accounts/signup`, `/accounts/login`, `/share`.
*   Analyse der Homepage zeigte Benutzernamen: `ramsey`, `wan`, `oliver`.
*   Aufruf einer Fehlerseite enthüllte `DEBUG = True` in den Django-Einstellungen.
*   Test des Shellshock-Potenzials auf `/cgi_wrapper` schlug fehl, lieferte aber detaillierte Django-Debug-Informationen.
*   Analyse des POST-Requests an `/search` identifizierte das `search_cookie`.
*   Dekodierung des `search_cookie`-Werts (`gASV...`) bestätigte eine BASE64-kodierte Python Pickle-Struktur.
*   Disassemblierung des Pickle-Bytecodes zeigte, dass der Suchterm serialisiert wurde, was auf eine unsichere Deserialisierung hindeutet.

### 💻 Initialer Zugriff

*   Erstellung eines Python-Skripts zur Generierung eines bösartigen Pickle-Payloads mit einem Reverse Shell Bash-Befehl (`import os; os.system(...)` via `__reduce__`).
*   Vorbereitung eines Netcat-Listeners auf dem Angreifer-System (Port 4444).
*   Senden eines präparierten HTTP POST-Requests an `/search`, wobei der `search_cookie`-Wert durch den bösartigen Pickle-Payload ersetzt wurde.
*   Erfolgreiche Etablierung einer Reverse Shell auf dem Zielsystem mit Root-Berechtigungen.

### 🐳 Container Breakout & Interne Enumeration

*   Identifizierung der Shell als Root innerhalb eines Docker-Containers (`root@...:/#`, `/proc/1/cgroup` enthielt `/docker/...`).
*   Überprüfung der Capabilities zeigte Standard-Container-Berechtigungen ohne offensichtlichen direkten Ausbruchsweg.
*   Interner Netzwerkscan (`ip a`, `nc -znv 172.17.0.1`) identifizierte offene Ports 22 (SSH) und 5003 (HTTP) auf der Docker-Host-IP (172.17.0.1).
*   Exfiltration der Django-Datenbank (`db.sqlite3`) aus dem Container auf das Angreifer-System mittels Netcat.
*   Analyse der `db.sqlite3` mittels `sqlite3` enthüllte Benutzerkonten (`aniqfakhrul`, `testing`, `ramsey`, `oliver`, `wan`) und ihre gehashten Passwörter (pbkdf2_sha256). Cracking-Versuche liefen an.

### 🔑 Horizontale Eskalation (ramsey)

*   Unter Annahme des erfolgreichen Knackens des Passworts für den Benutzer `ramsey` aus der Datenbank, erfolgreiche SSH-Anmeldung als `ramsey` auf dem Host-System (Port 2222).
*   Enumeration im Home-Verzeichnis von `ramsey` fand `user.txt`, `payload.png` und das Python-Skript `vuln.py`.
*   Prüfung der Sudo-Berechtigungen für `ramsey` zeigte die Möglichkeit, `/usr/bin/python /home/ramsey/vuln.py` als Benutzer `oliver` auszuführen (`(oliver) /usr/bin/python /home/ramsey/vuln.py`).
*   Analyse des Quellcodes von `vuln.py` zeigte die Verwendung von `pytesseract` und einer verwundbaren `eval()`-Funktion mit Eingabe aus `payload.png`.
*   Versuche, Code über Bild-Metadaten oder sichtbaren Text in `payload.png` für die `eval()`-Schwachstelle einzubetten, schlugen fehl.
*   **Strategiewechsel:** Modifizierung des `vuln.py`-Skripts (da `ramsey` Schreibrechte hat) mit Code zur Erlangung einer Shell (`import pty;pty.spawn("/bin/bash")`).
*   Ausführung des modifizierten `vuln.py` Skripts mit Sudo als `oliver` (`sudo -u oliver /home/ramsey/vuln.py`).
*   Erfolgreiche horizontale Privilegien-Eskalation zu Benutzer `oliver`.

### 👑 Root Eskalation (oliver)

*   Prüfung der Sudo-Berechtigungen für `oliver` enthüllte eine kritische Regel: `(root) SETENV: NOPASSWD: /usr/bin/python /opt/dockerScript.py`.
*   Analyse des Skripts `/opt/dockerScript.py` zeigte einen Import des `docker`-Moduls.
*   Ausnutzung der Sudo-Regel mit `SETENV` und der `PYTHONPATH`-Umgebungsvariable zur Manipulation des `docker`-Modulimports.
*   Erstellung einer bösartigen `docker.py`-Datei (`import pty;pty.spawn("/bin/bash")`) in einem beschreibbaren Verzeichnis (`/tmp`).
*   Ausführung des anfälligen Skripts: `sudo PYTHONPATH=/tmp python /opt/dockerScript.py`.
*   Erfolgreiche Erlangung einer Root-Shell auf dem Host-System.

### 🚩 Flags

*   **User Flag:** `Unb4ked_W00tw00t`
*   **Root Flag:** `Unb4ked_GOtcha!`

---

## 🧠 Wichtige Erkenntnisse

*   **Unsichere Deserialisierung:** Die Verwendung von Python Pickle mit nicht vertrauenswürdigen Daten (z.B. aus Cookies) führt direkt zu Remote Code Execution. Sichere Serialisierungsformate müssen verwendet werden.
*   **Debug-Modus in Produktion:** Das Aktivieren von `DEBUG = True` in Django-Produktionsumgebungen leakt kritische Informationen über die Anwendung und das System.
*   **Schwache Sudo-Konfigurationen:** Regeln, die die Ausführung von modifizierbaren Skripten oder Interpretern (insbesondere mit `NOPASSWD` und `SETENV`) erlauben, sind ein direkter Weg zur Privilegien-Eskalation. Sudo-Regeln müssen sorgfältig auditiert werden.
*   **Container Breakout:** Auch Root-Zugriff in einem Container bedeutet nicht vollständige Kompromittierung, aber interne Netzwerke und falsch konfigurierte Mounts/Services können Ausbruchswege zum Host bieten.
*   **Datenbank-Sicherheit:** Lokale Datenbankdateien sollten nicht exfiltriert werden können. Passwörter müssen sicher gehasht und gesalzen gespeichert werden, und kritische Konten sollten 2FA nutzen.
*   **Code Audits:** Regelmäßige Überprüfung des Anwendungscodes ist notwendig, um gefährliche Funktionen wie `eval()` oder unsichere Deserialisierungspraktiken zu identifizieren.
*   **Umgebungsvariablen-Hijacking:** Die Manipulation von Umgebungsvariablen wie `PYTHONPATH` kann zur Code-Ausführung in privilegierterem Kontext missbraucht werden.

---

## 📄 Vollständiger Bericht

Eine detaillierte Schritt-für-Schritt-Anleitung, inklusive Befehlsausgaben, Analyse, Bewertung und Empfehlungen für jeden Schritt, finden Sie im vollständigen HTML-Bericht, erstellt von Ben Chehade:

[**➡️ Vollständigen Pentest-Bericht hier ansehen**](https://alientec1908.github.io/UnbakedPie_HackMyVM_Hard/)

---

*Berichtsdatum: 16. Juni 2025*
*Pentest durchgeführt von Ben Chehade*
