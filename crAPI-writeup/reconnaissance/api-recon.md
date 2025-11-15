## Reconnaissance

### Ziel:
Eine vollständige Übersicht sämtlicher API-Endpunkte zu erhalten, inklusive:
- HTTP-Methoden
- Parameter
- Request-/Response-Strukturen
- Authentifizierungsmechanismen
- interne Objekt-IDs
- Fehlerverhalten
- Rollen-/Zugriffstrennung
- Diese API-Recon bildet die Grundlage für spätere Tests auf Broken Access Control, Injection, Rate Limit Issues, usw. 

### Vorgehen
Funktionale Erkundung (Black-Box) <br>
Die Anwendung wird vollständig durchgeklickt und dabei folgende Aspekte erfasst:
- Registrierung, Login, Session-Management
- Upload- und Post-Funktionen
- Profile, Einstellungen, Community-Elemente
- sämtliche fehleranfälligen Input-Felder (Mail, Datei-Uploads etc.)
- Parallel zeichnet mitmproxy alle Requests/Responses auf, inkl. Headern, Query-Parametern und Bodies.

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
