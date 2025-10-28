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

1. mitmproxy / mitmdump installieren:
```bash
sudo apt install mitmproxy -y
```
2. Proxy starten (hört auf localhost:8080, Web-UI auf Port 8081):
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

   Dadurch wird die Adresse lokal aufgelöst, aber nicht als „localhost“ erkannt, sodass Firefox den Proxy nutzt und der Traffic von mitmproxy aufgezeichnet werden kann.
   Nach dem schreiben mit Strg + O speichern ung mit Strg + X verlassen.

   
7. Ruft man *http://crapi.local* auf den entsprechenden Ports **8888** (crAPI) und **8025** (MailHog) auf sieht man die aufgezeichneten Requests im Proxy Web-UI:
   <img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/232a9897-f17a-43b1-9edf-1d438db41782" />


---

## Reconnaissance

**Ziel:**  
Ich möchte die gesamte Angriffsfläche der Anwendung erfassen. Dazu zeichne ich alle API-Endpoints, Funktionen und zugehörigen Requests auf um eine vollständige API-Bibliothek der Anwendung in Postman zu erstellen.

**Vorgehen:**  
1. Die Anwendung regulär im Browser bedienen und konsequent **alle** verfügbaren Funktionen durchspielen - sowohl mit regulären Eingaben als auch mit falschen/unerwarteten inputs.
2. Sämtliche ausgehenden HTTP(S)-Requests im Proxy-Interface mitschneiden (Headers, Query-Parameter, Request-Body, Response).  
3. Besonderes Augenmerk auf kritische Funktionen wie Authentifizierungs- und Session-Mechanismen, Datei-Uploads, Formular-Inputs und Fehlermeldungen legen.  

**Anschließend:**
1. Flows aus dem Web-UI runterladen (File -> Save)

2. Die Flows für Postman in OpenAPI-Spezifikation konvertieren (-i = input; -o = output; -p = API-Prefix; -f = filetype):
   ```bash
   pipx install mitmproxy2swagger
   mitmproxy2swagger -i flows -o spec.yml -p http://crapi.local -f flow
   ```
   Output ist eine Standardbeschreibung für APIs nach OpenAPI-Standard:
   ```bash
   nano spec.yml
   ```
   <img width="1116" height="996" alt="image" src="https://github.com/user-attachments/assets/79ce9f2d-cb6d-4d44-a78e-9ef119f68b5d" />

4. Anschließend die url definieren und *ignore::*'s der API Endpunkte entfernen und statische Ressourcenpfade trennen:
   <img width="1115" height="996" alt="image" src="https://github.com/user-attachments/assets/5606fc2a-8bab-417a-b22f-78499e8e1501" />
Nach dem aufräumen erneut den mitmproxy2swagger Befehl ausführen und das *--examples*-flag setzen.

5. Jetzt kann die spec.yml in Postman mit import -> file als vorsortiere Collection mit Beispiel-Calls importiert werden:
   <img width="1852" height="1046" alt="image" src="https://github.com/user-attachments/assets/ef05622c-83f1-4aa3-9141-ef92394474ef" />

### Postman Setup
Um sich in jeder Postman-Request zu authentifizieren, muss über den Login-Endpoint ein Bearer-Token für den aktuellen Benutzer angefordert werden, bevor authentifizierte Requests gemacht werden können:

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/98cb0fc9-e66d-45c6-8f17-80f32de5bb3d" />

Dieser Token kann wird im Collection-Tab unter Auth als Bearer Token eingefügt. Dadurch wird dieser Bearer-Token in nachfolgenden Requests als Authorization-Header mitgeschickt um Requests ihren Usern zuzuordnen:

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/5fc2aa95-a447-48a6-8b14-0bd7cbc3a19c" />

## Broken Access Control - BOLA & BFLA

### BOLA
BOLA (Broken Object Level Authorization) ist eine kritische API-Schwachstelle, die es Angreifern ermöglicht, fremde Objekte zu lesen oder zu verändern, indem sie lediglich Objekt-IDs in der URL oder im Request manipulieren. Ursache ist meist eine fehlende oder fehlerhafte serverseitige Prüfung, ob der anfragende Benutzer tatsächlich berechtigt ist, auf das angeforderte Objekt zuzugreifen.

So kann zum Beispiel User 1 sowohl auf das Objekt */users/dashboard/1* als auch */users/dashboard/2* zugreifen, wozu er allerdings nicht berechtigt ist.

1. Zweiten User *userB* erstellen per Request oder im Browser und einloggen für Bearer-Token:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/13064987-efc0-4a1d-9c4f-069fe68ae254" />

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/df6e9f1b-78c2-40c5-9c94-7b021fabb3c8" />

Anschließend in Postman jeweils ein Environment für *userA* und *userB* anlegen, um vor Requests schnell zwischen Usern wechseln zu können ohne den Body der Request verändern zu müssen.

BILD VON ENVIRONMENT!!!!



Für die BOLA-Tests wurden alle API-Endpunkte betrachtet, die Objekt-IDs im Pfad verwenden, also

**GET /identity/api/v2/user/videos/{id}** <br>
**PUT /identity/api/v2/user/videos/{id}** <br>
**GET /workshop/api/shop/orders/{id}?order_id=**<br>
**POST /workshop/api/shop/orders/{id}?order_id=** <br>

Der Endpoint **GET /community/api/v2/community/posts/{postId}** erlaubt den Aufruf beliebiger Community-Posts anhand ihrer Post-ID. Da die Plattform Community-Posts grundsätzlich öffentlich zugänglich macht und keine Rollen oder Zugriffsbeschränkungen (Wie public/private/friends) vorsieht, stellt dieser Zugriff in dem Fall keine BOLA-Schwachstelle dar.

Wären in der Anwendung jedoch Zugriffsstufen wie privat, freigegeben oder nur für Freunde sichtbar implementiert, müsste geprüft werden, ob der Server bei einem Aufruf mit fremden postId-Werten eine Autorisierungsprüfung durchführt. Nur dann wäre dieser Endpoint sicherheitsrelevant im Kontext von BOLA.






### BFLA


