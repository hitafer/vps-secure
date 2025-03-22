# Linux VPS absichern in 3 einfachen Schritten


Legen wir los!

- - -

## SSH Grundkonfiguration

### Generieren von SSH Keys

Es gibt hosting Anbieter bei denen man SSH-Keys direkt bei der Einrichtung eines Servers generieren kann oder gar seine eigenen Keys hochladen kann. Wir machen das ganze aber nun manuell.

Dafür generieren wir die Codes auf unserem lokalen Rechner wie folgt:

```bash
┌─[user@hostmaschine]─[~]
└──╼$ ssh-keygen -t rsa -b 4096 -f vps-ssh

Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in vps-ssh
Your public key has been saved in vps-ssh.pub
The key fingerprint is:
SHA256:zXyVAWK00000000000000000000VS4a/f0000+ag user@hostmaschine
The key's randomart image is:
...SNIP...
```

Durch den oben stehenden Befehl, haben wir ein Schlüsselpaar generiert. Dabei ist `vps-ssh` der private Key, dieser sollte mit niemanden geteilt werden. Der zweite Key ist der Public key, welchen wir später auf unserem Server hinterlegen. Dieser heißt `vps-ssh.pub`.

```bash
┌─[user@hostmaschine]─[~]
└──╼$ ls -l vps*

-rw------- 1 user user 3434 Apr 18 18:39 vps-ssh
-rw-r--r-- 1 user user  741 Apr 18 18:39 vps-ssh.pub
```

- - -

### SSH als root

Als nächstes melden wir uns mit dem `root` User auf dem Server an, um einen eigenen User für weitere Zugriffe einzurichten.

```bash
┌─[user@hostmaschine]─[~]
└──╼$ ssh root@<vps-ip-address>

root@<vps-ip-address>'s password:

[root@VPS ~]#
```

Nun erstellen wir einen neuen User, da wir unsere Dienste und co. nicht mit `root` ausführen wollen und hinterlegen unseren Publickey aus dem generierten Schlüsselpaar.

```
[root@VPS ~]# adduser user
[root@VPS ~]# usermod -aG sudo user
[root@VPS ~]# su - user

[user@VPS ~]$
```

```
[user@VPS ~]$ mkdir ~/.ssh
[user@VPS ~]$ echo '<vps-ssh.pub>' > ~/.ssh/authorized_keys
[user@VPS ~]$ chmod 600 ~/.ssh/authorized_keys
```

Nachdem wir den public key zu `authorized_keys` hinzugefügt haben, können wir den `private key` für den login nutzen.

```bash
┌─[user@hostmaschine]─[~]
└──╼$ ssh user@<vps-ip-address> -i vps-ssh

[user@VPS ~]$
```

- - -

### OpenSSH config anpassen

Wir passen ebenfalls einige Grundeinstellungen in der OpenSSH Konfigurationsdatei an, auch hier machen wir aber vorher wieder ein Backup der Datei.

```bash
[user@VPS ~]**$** sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
[user@VPS ~]**$** sudo nano /etc/ssh/sshd_config
```

In dem File setzen wir jetzt folgende Parameter bzw. fügen sie ggf. auch neu hinzu.

| Settings | Description |
| --- | --- |
| LogLevel VERBOSE | Gibt die Ausführlichkeitsstufe an, die bei der Protokollierung von Meldungen des SSH-Daemons verwendet wird. |
| PermitRootLogin no | Gibt an, ob sich root über SSH anmelden kann. |
| MaxAuthTries 3 | Gibt die maximale Anzahl der zulässigen Authentifizierungsversuche pro Verbindung an. |
| MaxSessions 5 | Gibt die maximale Anzahl offener Shell-, Anmelde- oder Subsystem-Sitzungen (z. B. SFTP) an, die pro Netzwerkverbindung zulässig sind. |
| HostbasedAuthentication no | Gibt an, ob die rhosts- oder /etc/hosts.equiv-Authentifizierung zusammen mit einer erfolgreichen Client-Host-Authentifizierung mit öffentlichem Schlüssel zulässig ist (hostbasierte Authentifizierung). |
| PasswordAuthentication no | Gibt an, ob eine Passwortauthentifizierung zulässig ist. |
| PermitEmptyPasswords no | Wenn die Kennwortauthentifizierung erlaubt ist, gibt sie an, ob der Server die Anmeldung bei Konten mit leeren Kennwortzeichenfolgen erlaubt. |
| ChallengeResponseAuthentication yes | Gibt an, ob die Challenge-Response-Authentifizierung zulässig ist. |
| KbdInteractiveAuthentication yes | Legt fest, ob die tastaturinteraktive Authentifizierung erlaubt ist. |
| UsePAM yes | Gibt an, ob PAM-Module für die Authentifizierung verwendet werden sollen. |
| X11Forwarding no | Gibt an, ob die X11-Weiterleitung erlaubt ist. |
| ClientAliveInterval 600 | Legt ein Timeout-Intervall in Sekunden fest, nach dem der SSH-Daemon, wenn keine Daten vom Client empfangen wurden, eine Nachricht über den verschlüsselten Kanal sendet, um eine Antwort vom Client anzufordern. |
| ClientAliveCountMax 0 | Legt die Anzahl der Client-Alive-Nachrichten fest, die gesendet werden dürfen, ohne dass der SSH-Daemon eine Nachricht vom Client zurückerhält. |
| AllowUsers | Wenn angegeben ist die Anmeldung nur für Benutzernamen zulässig, die angegeben wurden. |
| Protocol 2 | Gibt die Verwendung des neueren und sichereren Protokolls an. |
| AuthenticationMethods publickey,keyboard-interactive | Gibt die Authentifizierungsmethoden an, die erfolgreich abgeschlossen werden müssen, damit ein Benutzer Zugang erhält. |

### Updaten des Systems

Bevor wir nun mit dem absichern fortfahren, bringen wir das System ersteinmal auf den neusten Stand.

```bash
[user@VPS ~]**$** sudo apt update && sudo apt upgrade
```

- - -

## Fail2Ban

Als nächstes installieren und konfigurieren wir Fail2Ban.

```bash
[user@VPS ~]**$** sudo apt install fail2ban
```

Nach dem installieren, sichern wir erst einmal die default config Datei, falls wir irgend etwas falsch machen sollten, können wir jederzeit auf die Datei zurückgreifen.

```bash
[user@VPS ~]**$** sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.conf.bak
[user@VPS ~]**$** sudo nano /etc/fail2ban/jail.conf
```

In der Datei finden wir die auskommentierte Zeile `# [sshd]`. Darunter setzen wir dann folgende Einträge.

```bash
# [sshd]
enabled = true
bantime = 4w
maxretry = 3
```

Damit aktivieren wir das monitoring für den SSH-Server, setzen die Banzeit auf 4 Wochen und erlauben maximal 3 Versuche.

- - -

## 2 Faktor Authentifizierung

2 Faktor wird mit dem Google Authenticator ermöglicht. Dafür brauchen wir das [Google Authenticator PAM Module](https://github.com/google/google-authenticator-libpam)

```bash
[user@VPS ~]$ sudo apt install libpam-google-authenticator
[user@VPS ~]$ google-authenticator

Do you want authentication tokens to be time-based (y/n) y

Warning: pasting the following URL into your browser exposes the OTP secret to Google:
  https://www.google.com/chart?chs=200x200&chld=M|0&cht=qr&chl=otpauth://totp/user@hostmaschine%3Fsecret%...SNIP...%26issuer%3Dhostmaschine

   [ ---- QR Code ---- ]

Your new secret key is: ***************
Enter code from app (-1 to skip):
```

Der QR Code muss mit der 2 Faktor App gescannt werden. anschließend erhalten wir noch Backupcodes, welche wir aufbewahren sollten.

```
Enter code from app (-1 to skip): <Google-Auth Code>

Code confirmed
Your emergency scratch codes are:
  21323478
  43822347
  60232018
  73234726
  45456791

Do you want me to update your "/home/user/.google_authenticator" file? (y/n) y

Do you want to disallow multiple uses of the same authentication
token? This restricts you to one login about every 30s, but it increases
your chances to notice or even prevent man-in-the-middle attacks (y/n) y

By default, a new token is generated every 30 seconds by the mobile app.
In order to compensate for possible time-skew between the client and the server,
we allow an extra token before and after the current time. This allows for a
time skew of up to 30 seconds between authentication server and client. If you
experience problems with poor time synchronization, you can increase the window
from its default size of 3 permitted codes (one previous code, the current
code, the next code) to 17 permitted codes (the 8 previous codes, the current
code, and the 8 next codes). This will permit for a time skew of up to 4 minutes
between client and server.
Do you want to do so? (y/n) n

If the computer that you are logging into isn't hardened against brute-force
login attempts, you can enable rate-limiting for the authentication module.
By default, this limits attackers to no more than 3 login attempts every 30s.
Do you want to enable rate-limiting? (y/n) y
```

als nächstes müssen wir das PAM Modul für den SSH-Deamon anpassen um 2FA nutzen zu können. Dafür machen wir als erstes wieder ein Backup der Datei und öffnen die Datei dann.

```bash
[user@VPS ~]$ sudo cp /etc/pam.d/sshd /etc/pam.d/sshd.bak
[user@VPS ~]$ sudo nano /etc/pam.d/sshd
```

Wir setzen ein `#` vor die Zeile `@include common-auth` um sie auszukommentieren. Außerdem fügen wir 2 weitere Zeilen am ende hinzu.

```
#@include common-auth

...

auth required pam_google_authenticator.so
auth required pam_permit.so
```

### SSH Dienst neu starten

Als letztes muss der SSH-Dienst neu gestartet werden zum übernehmen der Konfiguration

```bash
[user@VPS ~]**$** sudo service ssh restart
```

- - -

Fertig! ab sofort kann sich auf dem Server nur noch mit dem angegebene User angemeldet werden. Außerdem wird der private-SSH-Key benötigt und der Code aus er 2 Faktor App.

