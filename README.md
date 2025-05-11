# Family3 - HackMyVM (Medium)

![Family3.png](Family3.png)

## Übersicht

*   **VM:** Family3
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Family3)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 7. November 2022
*   **Original-Writeup:** https://alientec1908.github.io/Family3_HackMyVM_Medium/
*   **Autor:** Ben C.

## Kurzbeschreibung

Das Ziel dieser Challenge war die Kompromittierung der virtuellen Maschine "Family3". Der Weg begann mit der Identifizierung eines CUPS-Dienstes, über den der Benutzername `mum` aufgedeckt wurde. Ein Brute-Force-Angriff auf SSH mit diesem Benutzernamen war erfolgreich (Passwort: `lovely`) und gewährte initialen Zugriff. Die weitere Eskalation umfasste mehrere Schritte:
1.  **mum -> dad:** Ausnutzung eines unsicheren Cronjobs (`/home/dad/project`), der als `dad` lief und ausführbare Dateien aus einem Verzeichnis (`~/survey`) ausführte, das durch `mum` über einen als `dad` laufenden Webserver (Port 8000) mit einer Reverse-Shell-Payload bestückt werden konnte.
2.  **dad -> baby:** Eine `sudo`-Regel erlaubte `dad`, `julia` als `baby` auszuführen, was zur Erlangung einer Shell als `baby` genutzt wurde.
3.  **baby -> root:** `baby` hatte eine `sudo`-Regel, die das Ausführen eines benutzerdefinierten Skripts (`/home/baby/chocapic`) als `root` erlaubte. Das Skript hatte eine Schwachstelle, die durch eine speziell präparierte Eingabe (`[ : ] ; bash`) die Ausführung beliebiger Befehle und somit eine Root-Shell ermöglichte.
Die Root-Flag war mit OpenSSL verschlüsselt; das Passwort wurde auf einer separaten, ungemounteten Partition gefunden.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `hydra`
*   `ssh`
*   `ls`, `cat`, `cd`, `echo`
*   `sudo`
*   `ss`, `netstat`
*   `find`
*   `env`, `printenv`
*   `who`
*   `strings`
*   `nc (netcat)`
*   `msfconsole` (für shell_to_meterpreter, local_exploit_suggester - obwohl nicht erfolgreich)
*   `python`
*   `wget`, `curl`
*   `stty`
*   `julia`
*   `cp`
*   `nano`
*   `grep`
*   `file`
*   `lsblk`, `mount`
*   `openssl`
*   Standard Linux-Befehle

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Family3" gliederte sich in folgende Phasen:

1.  **Reconnaissance:**
    *   IP-Findung mittels `arp-scan` (192.168.2.114).
    *   Portscan mit `nmap` identifizierte Port 22 (SSH OpenSSH 8.4p1) und Port 631 (CUPS 2.3).
    *   Untersuchung des CUPS-Webinterfaces deckte den Benutzernamen `mum` über eine Drucker-Jobliste auf.

2.  **Initial Access (SSH als `mum`):**
    *   `hydra` wurde verwendet, um einen Brute-Force-Angriff auf den SSH-Dienst für den Benutzer `mum` mit der `rockyou.txt`-Liste durchzuführen. Das Passwort `lovely` wurde gefunden.
    *   Erfolgreicher SSH-Login als `mum`.

3.  **Privilege Escalation (von `mum` zu `dad`):**
    *   Enumeration als `mum` zeigte einen Dienst auf `localhost:8000`, der als UID `1000` (`dad`) lief.
    *   Die Datei `/home/dad/project` (ein Bash-Skript, wahrscheinlich via Cronjob als `dad` ausgeführt) wurde analysiert. Das Skript verschob ausführbare Dateien von `dad` aus `/home/dad` nach `~/survey` (Home-Verzeichnis des Ausführenden, also `/home/dad/survey`) und führte diese dort aus.
    *   Eine Reverse-Shell-Payload (`rev`) wurde erstellt, auf das Zielsystem nach `/tmp` heruntergeladen und dann (implizit erfolgreich, trotz `curl`-Fehlermeldung) über den Webserver auf Port 8000 (als `dad` laufend) in `/home/dad` platziert.
    *   Der Cronjob führte die Payload aus, was zu einer Reverse Shell als `dad` führte. (Der Bericht zeigte zwischendurch einen fehlgeschlagenen Versuch, einen lokalen Exploit über Metasploit auf `mum`s Shell auszuführen).

4.  **Privilege Escalation (von `dad` zu `baby`):**
    *   `sudo -l` für `dad` zeigte: `(baby) NOPASSWD: /usr/bin/julia`.
    *   Mit `sudo -u baby /usr/bin/julia` wurde die Julia REPL gestartet. Innerhalb von Julia wurde mit `run(``bash``)` eine Shell als `baby` erlangt. Die User-Flag wurde gelesen.

5.  **Privilege Escalation (von `baby` zu `root`):**
    *   `sudo -l` für `baby` zeigte: `(root) NOPASSWD: /home/baby/chocapic`.
    *   Das Skript `/home/baby/chocapic` wurde analysiert. Es enthielt Filter für die Eingabe, die jedoch umgangen werden konnten. Nach den Filtern wurde die gesamte Eingabe mit `bash -c` ausgeführt.
    *   Durch die Eingabe `[ : ] ; bash` bei der Ausführung von `sudo /home/baby/chocapic` wurde eine Root-Shell erlangt.
    *   Die Root-Flag (`/root/root.txt`) war mit OpenSSL verschlüsselt (`file root.txt`).
    *   `lsblk` zeigte eine ungemountete Partition (`sda3`). Diese wurde gemountet und enthielt eine Datei `password` mit dem Entschlüsselungspasswort.
    *   Mit `openssl enc -aes128 -pbkdf2 -d -in root.txt -out file.txt` und dem gefundenen Passwort wurde die Root-Flag entschlüsselt.

## Wichtige Schwachstellen und Konzepte

*   **Informationspreisgabe über CUPS:** Der Benutzername `mum` wurde durch die Jobliste des CUPS-Druckdienstes aufgedeckt.
*   **Schwache SSH-Passwörter:** Der initiale Zugriff erfolgte durch Brute-Forcing eines schwachen SSH-Passworts.
*   **Unsicherer Cronjob:** Ein Cronjob führte ein Skript (`/home/dad/project`) aus, das Dateien aus dem Home-Verzeichnis von `dad` in ein Unterverzeichnis verschob und dort ausführte. Da ein Webserver als `dad` lief und Uploads nach `/home/dad` (implizit) ermöglichte, konnte dies zur Codeausführung als `dad` missbraucht werden.
*   **Unsichere `sudo`-Konfigurationen:**
    *   `dad` durfte `julia` als `baby` ausführen, was zu einer Shell als `baby` missbraucht wurde.
    *   `baby` durfte ein benutzerdefiniertes Skript (`chocapic`) als `root` ausführen. Dieses Skript hatte eine Command-Injection-Schwachstelle.
*   **Command Injection in benutzerdefiniertem Sudo-Skript:** Das `chocapic`-Skript verwendete `bash -c` mit Benutzereingaben nach unzureichender Filterung, was die Ausführung beliebiger Befehle ermöglichte.
*   **Verschlüsselte Flag mit Passwort auf separater Partition:** Die Root-Flag war verschlüsselt, das Passwort befand sich jedoch unverschlüsselt auf einer leicht zugänglichen, ungemounteten Partition.

## Flags

*   **User Flag (`/home/baby/user.txt`):** `4fd7881d10d000c69c6f3bca0c8f7ad5`
*   **Root Flag (entschlüsselt aus `/root/root.txt`):** `8d8ff4976efccbfc8ff7d7554b9239e5`

## Tags

`HackMyVM`, `Family3`, `Medium`, `CUPS`, `Hydra`, `SSHBruteForce`, `CronjobExploitation`, `SudoJulia`, `SudoCustomScript`, `CommandInjection`, `OpenSSLEncryption`, `PasswordInPartition`, `Linux`, `Web`, `Privilege Escalation`
