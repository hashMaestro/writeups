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
1. VM updaten und neuen Ordner erstellen
   ```bash
   sudo apt update && sudo apt upgrade -y
   mkdir crapi
   cd crapi

2. docker-compose.yml aus dem offiziellen Repository herunterladen:  
   ```bash
   curl -o docker-compose.yml https://raw.githubusercontent.com/OWASP/crAPI/main/deploy/docker/docker-compose.yml
3. Erforderliche Docker images pullen:
   ```bash
   docker-compose pull
4. App mit docker-compose starten / stoppen:
   ```bash
   docker-compose -f docker-compose.yml --compatibility up -d
   ```
   
   ```bash
   docker-compose -f docker-compose.yml stop
5. Die App ist jetzt auf dem localhost unter Port **8888** (crAPI) und Port **8025** (MailHog) erreichbar und erfolgreich installiert.




### Proxy Setup 

Bevor ich die App zum ersten Mal benutze richte ich einen Proxy ein, um alle HTTP Requests zwischen dem Client (Browser) und dem Webserver (crAPI) für die spätere Analyse aufzuzeichnen.

1. mitmproxy / mitmweb installieren:
```bash
sudo apt install mitmproxy -y
```
2. Proxy starten (Hört auf localhost:8080, Web-UI auf localhost:8081):
```bash
mitmweb --listen-port 8080
```
Damit schneidet

3. Firefox so konfigurieren, dass der Traffic über den Proxy läuft:
   
   <img width="758" height="447" alt="image" src="https://github.com/user-attachments/assets/5bd6b62d-7a98-4963-9183-de4ea86c18bb" />


4. mitmproxy Root-CA auf http://mitm.it runterladen und in Firefox importieren um HTTPS-Traffic zu entschlüsseln:
   
   <img width="1848" height="1003" alt="image" src="https://github.com/user-attachments/assets/10537bb4-2520-4767-8262-7c288f48145e" />
   <img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/383d7875-1391-4ad1-b727-30069fc98c45" />

5. Dass alles richtig eingerichtet ist, sehen wir an den aufgezeichneten Requests in dem Web-UI des Proxys:

   <img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/56e6603f-ae29-4b9d-beaf-86d7de5c9926" />

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


