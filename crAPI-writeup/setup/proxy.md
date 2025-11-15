## Proxy Setup 

Zur vollständigen Analyse des API-Traffics wird ein Proxy zwischengeschaltet.

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

   
7. Ruft man *http://crapi.local* auf den entsprechenden Ports **8888** (crAPI) und **8025** (MailHog) auf sieht man die aufgezeichneten Requests im Proxy Web-UI:
   <img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/232a9897-f17a-43b1-9edf-1d438db41782" />
