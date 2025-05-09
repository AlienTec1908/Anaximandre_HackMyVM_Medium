# Anaximandre - HackMyVM Writeup

![Anaximandre VM Icon](Anaximandre.png)

Dieses Repository enthält das Writeup für die HackMyVM-Maschine "Anaximandre" (Schwierigkeitsgrad: Medium), erstellt von DarkSpirit. Das Ziel war es, initialen Zugriff auf die virtuelle Maschine zu erlangen und die Berechtigungen bis zum Root-Benutzer zu eskalieren.

## VM-Informationen

*   **VM Name:** Anaximandre
*   **Plattform:** HackMyVM
*   **Autor der VM:** DarkSpirit
*   **Schwierigkeitsgrad:** Medium
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=Anaximandre](https://hackmyvm.eu/machines/machine.php?vm=Anaximandre)

## Writeup-Informationen

*   **Autor des Writeups:** Ben C.
*   **Datum des Berichts:** 11. April 2023
*   **Link zum Original-Writeup (GitHub Pages):** [https://alientec1908.github.io/Anaximandre_HackMyVM_Medium/](https://alientec1908.github.io/Anaximandre_HackMyVM_Medium/)

## Kurzübersicht des Angriffspfads

Der Angriff auf die Maschine Anaximandre umfasste mehrere Schritte, beginnend mit der Netzwerkaufklärung, über die Ausnutzung von Web-Schwachstellen und Fehlkonfigurationen bis hin zur vollständigen Kompromittierung des Systems.

1.  **Reconnaissance:** Identifizierung der Ziel-IP (`192.168.2.114`) mittels `arp-scan`. Ein `nmap`-Scan offenbarte offene Ports: SSH (22), HTTP (80) mit Apache und WordPress 6.1.1, sowie Rsync (873).
2.  **Web Enumeration (WordPress):**
    *   `nikto` und `gobuster` bestätigten die WordPress-Installation und fanden interessante Pfade sowie ein offenes Upload-Verzeichnis (`/wp-content/uploads/`).
    *   `wpscan` identifizierte die WordPress-Version 6.1.1 als veraltet und verwundbar (CVE-2022-3590 - Unauthenticated Blind SSRF). Zwei Benutzer (`admin`, `webmaster`) wurden enumeriert.
    *   Ein Passwort-Brute-Force-Angriff mit `wpscan` auf den Benutzer `webmaster` war erfolgreich und ergab das Passwort `mickey`.
    *   Eine im System gefundene "Admin-Notiz" enthielt den String `Yn89m1RFBJ`, der sich als Schlüssel herausstellte.
3.  **Rsync Enumeration:**
    *   Anonymer Zugriff auf den Rsync-Share `share_rsync` war möglich.
    *   Verschlüsselte Logdateien (`.cpt`-Dateien) wurden heruntergeladen.
    *   Die Datei `access.log.cpt` wurde mit `ccrypt` und dem zuvor gefundenen Schlüssel `Yn89m1RFBJ` erfolgreich entschlüsselt.
4.  **LFI / RCE Exploitation:**
    *   Analyse der entschlüsselten `access.log` offenbarte eine Local File Inclusion (LFI) Schwachstelle im Skript `/exemplos/codemirror.php` (Parameter `pagina`).
    *   Die LFI wurde genutzt, um `/etc/passwd` auszulesen und den Benutzer `chaz` zu entdecken.
    *   Die LFI-Schwachstelle wurde mittels des `data://` Wrappers und Base64-kodiertem PHP-Code zu Remote Code Execution (RCE) als Benutzer `www-data` eskaliert.
5.  **Initial Access:**
    *   Ein Base64-kodierter PHP-Reverse-Shell-Payload wurde über die LFI/RCE-Schwachstelle ausgeführt.
    *   Eine interaktive Shell als `www-data` wurde über einen `netcat`-Listener auf dem Angreifer-System etabliert und stabilisiert.
6.  **Privilege Escalation:**
    *   Die Datei `/etc/rsyncd.auth` enthielt die Klartext-Zugangsdaten `chaz:alanamorrechazado`.
    *   Mit `su chaz` und dem gefundenen Passwort wurde zum Benutzer `chaz` gewechselt.
    *   `sudo -l` für `chaz` zeigte die Regel `(ALL : ALL) NOPASSWD: /usr/bin/cat /home/chaz/*`.
    *   Die `user.txt` Flag wurde mit `cat /home/chaz/user.txt` gelesen.
    *   Durch Ausnutzung der unsicheren `sudo`-Regel mit Path Traversal (`sudo -u root /usr/bin/cat /home/chaz/../../root/root.txt`) wurde die `root.txt` Flag gelesen.
    *   Ebenso wurde der private SSH-Schlüssel des Root-Benutzers (`/root/.ssh/id_rsa`) mittels der `sudo cat`-Regel ausgelesen.
    *   Ein SSH-Login als `root` mit dem erbeuteten Schlüssel ermöglichte den vollständigen Systemzugriff.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `nikto`
*   `gobuster`
*   `curl`
*   `wpscan`
*   `rsync`
*   `ccrypt`
*   `cat`
*   `grep`
*   `echo`
*   `base64`
*   `nc` (netcat)
*   `python3`
*   `stty`
*   `sudo`
*   `id`
*   `uname`
*   `su`
*   `ls`
*   `vi`
*   `chmod`
*   `ssh`

## Identifizierte Schwachstellen (Zusammenfassung)

*   Veraltete WordPress-Version (6.1.1) mit bekannter SSRF-Schwachstelle (CVE-2022-3590).
*   Schwache WordPress-Passwörter (`webmaster:mickey`).
*   Offenes Verzeichnislisting für das WordPress Upload-Verzeichnis.
*   Anonym zugänglicher Rsync-Dienst mit sensiblen, wenn auch verschlüsselten, Daten.
*   Speicherung eines Entschlüsselungspassworts/Schlüssels in einer für Angreifer zugänglichen Notiz/Datei.
*   Local File Inclusion (LFI) Schwachstelle in einem PHP-Skript (`/exemplos/codemirror.php`).
*   Möglichkeit zur Remote Code Execution (RCE) durch die LFI via `data://` Wrapper.
*   Speicherung von Rsync-Benutzer-Credentials im Klartext in `/etc/rsyncd.auth` (lesbar für `www-data`).
*   Unsichere `sudo`-Konfiguration, die `cat` mit Wildcard und Path Traversal für den Benutzer `chaz` erlaubte (`NOPASSWD`).

## Flags

*   **User Flag (`/home/chaz/user.txt`):** `d151c8ace0dbdd0ef23a3e3200f696f1`
*   **Root Flag (`/root/root.txt`):** `a3cbb8984cf5f19086595c6a2f569786`

---
*Bericht von Ben C. - Cyber Security Reports*
