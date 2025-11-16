## Broken Access Control - BOLA 

### BOLA
BOLA (Broken Object Level Authorization) ist eine kritische API-Schwachstelle, die es Angreifern ermöglicht, fremde Objekte zu lesen oder zu verändern, indem sie lediglich Objekt-IDs in der URL oder im Request manipulieren. Ursache ist meist eine fehlende oder fehlerhafte serverseitige Prüfung, ob der anfragende Benutzer tatsächlich berechtigt ist, auf das angeforderte Objekt zuzugreifen.

So kann zum Beispiel User 1 sowohl auf das Objekt */users/dashboard/1* als auch */users/dashboard/2* zugreifen, wozu er allerdings nicht berechtigt ist.

1. Zweiten User *userB* erstellen per Request oder im Browser und einloggen für Bearer-Token:
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/13064987-efc0-4a1d-9c4f-069fe68ae254" />

<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/df6e9f1b-78c2-40c5-9c94-7b021fabb3c8" />

Anschließend in Postman jeweils ein Environment für *userA* und *userB* anlegen, um vor Requests schnell zwischen Usern wechseln zu können ohne den Body der Request verändern zu müssen.

<img width="958" height="675" alt="image" src="https://github.com/user-attachments/assets/9690c1c5-5259-4b07-89a9-3c6ac4b2143d" />




Für die BOLA-Tests wurden alle API-Endpunkte betrachtet, die Objekt-IDs im Pfad verwenden, also:

#### Videos
*GET /identity/api/v2/user/videos/{id}* <br>
*PUT /identity/api/v2/user/videos/{id}* <br>
*POST /identity/api/v2/user/videos/{id}* <br>
*DELETE /identity/api/v2/user/videos/{video_id}* <br>

#### Orders
*GET /workshop/api/shop/orders/{id}*<br>
*POST /workshop/api/shop/orders/{id}* <br>

#### Vehicles
*GET /identity/api/v2/vehicle/{vehicleId}/location*

#### Admin
*DELETE /identity/api/v2/admin/videos/{video_id}*

Der Endpoint *GET /community/api/v2/community/posts/{postId}* erlaubt den Aufruf beliebiger Community-Posts anhand ihrer Post-ID. Da die Plattform Community-Posts grundsätzlich öffentlich zugänglich macht und keine Rollen oder Zugriffsbeschränkungen (Wie public/private/friends) vorsieht, stellt dieser Zugriff in dem Fall keine BOLA-Schwachstelle dar.

Wären in der Anwendung jedoch Zugriffsstufen wie **privat**, **freigegeben** oder **nur für Freunde sichtbar** implementiert, müsste geprüft werden, ob der Server bei einem Aufruf mit fremden postId-Werten eine Autorisierungsprüfung durchführt. Nur dann wäre dieser Endpoint sicherheitsrelevant im Kontext von BOLA.

Um die Route **/identity/api/v2/user/videos/{id}** auf BOLA zu testen, lade ich zunächst als **User A** über den vorgesehenen Upload-Endpoint ein neues Video hoch und erhalte dabei eine eindeutige *videoId*. Anschließend wechsle ich in Postman in das **User B**-Environment und sende einen GET-Request an dieselbe Route, jedoch mit der *videoId* von **User A**. Ich analysiere die Serverantwort, um festzustellen, ob **User B** unberechtigt auf die fremde Ressource zugreifen kann.

1. Nach dem Hochladen im UI gibt die API als Response die Video-ID zurück:
   <img width="878" height="205" alt="image" src="https://github.com/user-attachments/assets/150ba517-4a65-40ec-a707-b13e410268be" />

2. Diese ID als ID im GET Endpunkt als userB verwenden:
   <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/bafe907e-57c6-4137-8832-3df52e943baa" />

   
Wie man sieht, gibt der Server einen Code 200 und das gesamte Video in base64 Kodierung zurück. <br>
Der Base64-kodierte Payload lässt sich ohne zusätzliche Informationen zu einer funktionsfähigen MP4-Datei dekodieren:

```
 base64 -d input.txt > output.mp4
```

<img width="1471" height="828" alt="image" src="https://github.com/user-attachments/assets/933f75c6-cb73-4e65-8a65-3a1dde19663f" />
<br>

Mit der HTTP Methode DELETE wird Code 404 zurückgegeben, da dies eine Admin-Funktionalität ist. Wie man dies umgehen kann, wird später überprüft. <br>
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/003307ec-d262-4587-9744-87749fe1a39c" />


### Sicherheitsbewertung

Die API-Route:

GET | POST | PUT */identity/api/v2/user/videos/{id}*

weist eine Broken Object Level Authorization (BOLA)-Schwachstelle auf.
Die zentrale Ursache:
- Die API prüft nicht, ob der anfragende Benutzer Eigentümer des Videos ist.
- Die Ressource ist allein über eine vorhersehbare, nicht geschützte ID adressierbar.
- Jede gültige Video-ID führt zu einer erfolgreichen, unautorisierten Datenoffenlegung.

### Konsequenzen

Ein Angreifer kann:

- IDs iterieren oder gezielt aus Responses extrahieren,
- sämtliche Videos der Plattform herunterladen,
- auf fremde, möglicherweise sensible Daten zugreifen.
- Das stellt eine klare Verletzung der Zugriffskontrolle dar.

### Fix
Die Schwachstelle kann serverseitig verhindert werden, indem jede Videoressource beim Upload eindeutig einem Benutzer zugeordnet wird (z. B. über ein Feld owner_user_id). Bei jedem Zugriff auf eine konkrete Ressource – egal ob GET, PUT oder DELETE – muss der Server den Benutzer aus dem Bearer-Token bestimmen und prüfen, ob dieser Benutzer tatsächlich der Eigentümer des Videos ist oder über explizite Berechtigungen verfügt. Nur wenn die Zuordnung Video → Besitzer mit der User-ID aus dem Token übereinstimmt, darf der Zugriff erlaubt werden; andernfalls muss der Server mit 403 Forbidden antworten.

## Orders
Um die Route */workshop/api/shop/orders/{id}* auf BOLA zu testen gehe ich wie bei der Videos-Route vor, indem ich mit userA eine neue Order platziere und versuche, mit userB darauf zuzugreifen.

1. Order platzieren mit userA und Order-ID in der Response entnehmen:
   <img width="906" height="742" alt="image" src="https://github.com/user-attachments/assets/b9d1ba61-aecd-4131-b3f1-f88729f0a792" />

2. Mit userB über den GET-Endpoint die ID anfragen:
  <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/addf3486-7de9-45f6-a017-95cb61f2cc26" />

```json
{
    "order": {
        "id": 6,
        "user": {
            "email": "userA@mail.com",
            "number": "1111111111"
        },
        "product": {
            "id": 2,
            "name": "Wheel",
            "price": "10.00",
            "image_url": "images/wheel.svg"
        },
        "quantity": 1,
        "status": "delivered",
        "transaction_id": "84c7df5e-edae-40a0-8aa7-fb93e06149c0",
        "created_on": "2025-11-15T23:50:27.438267"
    },
    "payment": {
        "transaction_id": "84c7df5e-edae-40a0-8aa7-fb93e06149c0",
        "order_id": 6,
        "amount": 10,
        "paid_on": "2025-11-15T23:50:27.438267",
        "card_number": "XXXXXXXXXXXX9372",
        "card_owner_name": "userA",
        "card_type": "Visa",
        "card_expiry": "10/2027",
        "currency": "USD"
    }
}
```

Damit zeigt der Test eindeutig, dass auch diese Route nicht gegen BOLA geschützt ist. Zusätzlich werden hier besonders sensible Daten offengelegt, darunter E-Mail-Adresse, Telefonnummer, Timestamps und vollständige Zahlungsinformationen. Dadurch entsteht nicht nur ein klassischer Objektzugriffsschutzfehler, sondern auch eine schwerwiegende Verletzung von Datenschutz- und Sicherheitsanforderungen.




