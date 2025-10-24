# crAPI Writeup

## Kurzfassung (TL;DR)
1. Reconnaissance mit **Burp Proxy** und **mitmweb**
2. Analyse mit
3. Exploitation mit
4. 

---

## Kontext / Ziel
crAPI ist eine absichtlich vulnerable WebAPP zum demonstrieren der 10 häufigsten API Security Schwachstellen und dient als legale, self hosted Lernplattform für Pentester und Security Enthusiasten.

In diesem Writeup zeige ich meine Herangehensweise und Gedanken während des Pentests und was ich dabei gelernt habe.

---

## Setup

### crAPI Setup
crAPI wird auf einer Ubuntu-VM ausgeführt und mit Docker Compose bereitgestellt:
1. Docker installieren wie in den Docs beschrieben: https://docs.docker.com/engine/install/ubuntu
2. crAPI installieren wie in Docs den beschrieben:  https://owasp.org/crAPI/docs/setup.html
5. Die App ist jetzt auf dem localhost unter Port **8888** (crAPI) und Port **8025** (MailHog) erreichbar und erfolgreich installiert: 
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/f65f4e29-c7ad-4421-8ad7-1e2f9d9d99e5" />
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/7b960514-9da6-4427-958b-9e02ba6968a7" />

### Proxy Setup 

Bevor ich die App zum ersten Mal benutze richte ich einen Proxy ein, um alle Requests zwischen dem Client (Browser) und dem Webserver (crAPI) für die spätere Analyse aufzuzeichnen.

1. mitmproxy / mitmweb installieren:
```bash
sudo apt install mitmproxy -y
```
2. Proxy starten (Hört auf localhost:8080, Web-UI auf localhost:8081):
```bash
mitmweb 
```

3. Firefox so konfigurieren, dass der Traffic über den Proxy läuft:
   
   <img width="758" height="447" alt="image" src="https://github.com/user-attachments/assets/5bd6b62d-7a98-4963-9183-de4ea86c18bb" />


4. mitmproxy Root-CA auf http://mitm.it runterladen und in Firefox importieren um HTTPS-Traffic zu entschlüsseln:
   
   <img width="1848" height="1003" alt="image" src="https://github.com/user-attachments/assets/10537bb4-2520-4767-8262-7c288f48145e" />
   <img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/383d7875-1391-4ad1-b727-30069fc98c45" />
   
5. Da Firefox bei Anfragen zum localhost/127.0.0.1 standardmäßig den Proxy bypasst muss in /etc/hosts ein Eintrag für crapi erstellt werden:
   ```bash
   sudo nano /etc/hosts
   ```
   <img width="814" height="576" alt="image" src="https://github.com/user-attachments/assets/ad01f36a-e14c-48ee-9dfd-9123ecb39692" />

   Nach dem schreiben mit Strg + O speichern ung mit Strg + X verlassen.

   
7. Ruft man *http://crapi.local* auf den entsprechenden Ports **8888** (crAPI) und **8025** (MailHog) auf sieht man die aufgezeichneten Requests im Proxy Web-UI:
   <img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/232a9897-f17a-43b1-9edf-1d438db41782" />


---

## Reconnaissance

**Ziel:**  
Ich möchte die gesamte Angriffsfläche der Anwendung erfassen. Dazu zeichne ich alle API-Endpoints, Funktionen und zugehörigen Requests vollständig auf.

**Vorgehen:**  
1. Die Anwendung regulär im Browser bedienen und konsequent **alle** verfügbaren Funktionen durchspielen - sowohl mit regulären Eingaben als auch mit falschen/unerwarteten inputs.
2. Sämtliche ausgehenden HTTP(S)-Requests im Proxy-Interface mitschneiden (Headers, Query-Parameter, Request-Body, Response).  
3. Besonderes Augenmerk auf kritische Funktionen wie Authentifizierungs- und Session-Mechanismen, Datei-Uploads, Formular-Inputs und Fehlermeldungen legen.  

**Anschließend:**
1. Flows in .har konvertieren:
   <Bild> von Options -> Export

   ```bash
   mitmweb -r flows.har
   ```
   
2. In Postman importieren und nach "Thema" (Auth, Admin, Users, Upload) sortieren:
   <Bild>

Jetzt kann ich anfangen die OWASP Top 10 durchzugehen und die entsprechenden Requests auf Schwachstellen untersuchen oder direkt interessante Requests untersuchen.


