# Broken Authentication
**API2:2023 – Broken Authentication** betrifft alle Fehler, bei denen sich ein Angreifer unberechtigt authentifizieren, fremde Accounts übernehmen 
oder Tokens missbrauchen kann. Dazu zählen fehlerhafte Login-Mechanismen, unsichere Passwort- oder Token-Verwaltung sowie unzureichende 
Schutzmaßnahmen gegen Brute-Force- oder Credential-Stuffing-Angriffe. Die Folge sind Kontoübernahmen, unautorisierte API-Aufrufe 
und ein umfassender Verlust der Zugriffskontrolle. 

## Auth in crAPI
crAPI verwendet als Authentifizierungsmethode einen JWT (Single Token) als Bearer Token in jeder Request um den Nutzer zu authentifizieren. 
Der Token wird bei jedem "frischen" Login generiert und dem User zugeteilt, der sich anmeldet. Mit [jwt.io](https://jwt.io) lassen sich JWTs dekodieren:
<img width="1840" height="992" alt="image" src="https://github.com/user-attachments/assets/36abaeb4-e95e-4e32-8e1a-f781178e9a55" />


Die JWTs sind mit RS256 signiert, enthalten die Rolle (z.B. user), die Login-Email sowie die Gültigkeitsdauer.

Was sofort auffällt ist die unüblich lange Gültigkeitsdauer von 7 Tagen! Die Gültigkeitsdauer ist in den JWTs im Unix-Zeitformat angegeben 
und lässt sich einfach zurückrechnen: 
- exp – iat = 1763979951 – 1763375151 = 604800 Sekunden = ~7 Tage

Dies stellt ein erhebliches Sicherheitsrisiko dar, da ein kompromittierter Token über die gesamte Laufzeit ungehinderten Zugriff ermöglicht. 
Wenn die API keine serverseitige Token-Invalidierung implementiert, kann ein Angreifer selbst nach einem Logout des Nutzers das Konto weiterhin 
vollständig missbrauchen.

## Token-Invalidierung
1. Um die API auf Token-Invalidierung zu prüfen logge ich mich über das UI ein und speichere den Token, den die API nach dem Einloggen zurückgibt:
<img width="920" height="772" alt="image" src="https://github.com/user-attachments/assets/a4faccb8-c202-42e3-9983-629c23868415" />
2. Anschließend ausloggen um die Token-Invalidierung auszulösen
3. Wenn die Token-Invalidierung korrekt implementiert ist, sollte die API Requests mit diesem Bearer Token nicht mehr akzeptieren und einen
   Code 401 Unauthorized zurückgeben:
   <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/481c21d7-8809-46fd-93ed-44bdde963b1c" />
Die API akzeptiert die Anfrage auch nach Abmelden des Users, was belegt, dass keine Token-Invalidierung implementiert wurde.

### Konsequenzen
Ein Angreifer, der den Token zuvor abgefangen hat (z. B. durch Logging, XSS, Proxy oder Sniffing), erhält mindestens bis zum 
Ablauf der Token-Lebenszeit (7 Tage!) vollständigen Zugriff auf das Konto des Opfers. Dabei gibt es keine Möglichkeit den Token zu widerrufen, 
selbst wenn der Vorfall bemerkt wurde.

### Fix
- Serverseitigen Token-Revocation-Mechanismus einführen (z.B. Blacklist, Session-Store, Session-Versioning)
- Token-Laufzeit auf 30-60 Minuten reduzieren
- Invalidation aller aktiven Tokens bei kritischen Aktionen wie Logout, Passwortänderung, MFA-Änderung etc.

## Token-Expiry-Handling
Um zu prüfen, ob der JWT überhaupt nach seiner Lebenszeit abläuft, wird ein gültiger Token so manipuliert, dass *exp*  

