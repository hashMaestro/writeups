## Broken Access Control - BOLA 

### BOLA
BOLA (Broken Object Level Authorization) ist eine kritische API-Schwachstelle, die es Angreifern ermöglicht, fremde Objekte zu lesen oder zu verändern, indem sie lediglich Objekt-IDs in der URL oder im Request manipulieren. Ursache ist meist eine fehlende oder fehlerhafte serverseitige Prüfung, ob der anfragende Benutzer tatsächlich berechtigt ist, auf das angeforderte Objekt zuzugreifen.

So kann zum Beispiel User 1 sowohl auf das Objekt */users/dashboard/1* als auch */users/dashboard/2* zugreifen, wozu er allerdings nicht berechtigt ist.

1. Zweiten User *userB* erstellen per Request oder im Browser und einloggen für Bearer-Token:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/13064987-efc0-4a1d-9c4f-069fe68ae254" />

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/df6e9f1b-78c2-40c5-9c94-7b021fabb3c8" />

Anschließend in Postman jeweils ein Environment für *userA* und *userB* anlegen, um vor Requests schnell zwischen Usern wechseln zu können ohne den Body der Request verändern zu müssen.

<img width="958" height="675" alt="image" src="https://github.com/user-attachments/assets/9690c1c5-5259-4b07-89a9-3c6ac4b2143d" />




Für die BOLA-Tests wurden alle API-Endpunkte betrachtet, die Objekt-IDs im Pfad verwenden, also

**GET /identity/api/v2/user/videos/{id}** <br>
**PUT /identity/api/v2/user/videos/{id}** <br>

**GET /workshop/api/shop/orders/{id}?order_id=**<br>
**POST /workshop/api/shop/orders/{id}?order_id=** <br>

Der Endpoint **GET /community/api/v2/community/posts/{postId}** erlaubt den Aufruf beliebiger Community-Posts anhand ihrer Post-ID. Da die Plattform Community-Posts grundsätzlich öffentlich zugänglich macht und keine Rollen oder Zugriffsbeschränkungen (Wie public/private/friends) vorsieht, stellt dieser Zugriff in dem Fall keine BOLA-Schwachstelle dar.

Wären in der Anwendung jedoch Zugriffsstufen wie **privat**, **freigegeben** oder **nur für Freunde sichtbar** implementiert, müsste geprüft werden, ob der Server bei einem Aufruf mit fremden postId-Werten eine Autorisierungsprüfung durchführt. Nur dann wäre dieser Endpoint sicherheitsrelevant im Kontext von BOLA.

Um die Route */identity/api/v2/user/videos/{id}* auf BOLA zu testen, lade ich zunächst als **User A** über den vorgesehenen Upload-Endpoint ein neues Video hoch und erhalte dabei eine eindeutige *videoId*. Anschließend wechsle ich in Postman in das **User B**-Environment und sende einen GET-Request an dieselbe Route, jedoch mit der *videoId* von **User A**. Ich analysiere die Serverantwort, um festzustellen, ob **User B** unberechtigt auf die fremde Ressource zugreifen kann.“

