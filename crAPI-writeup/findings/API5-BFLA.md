# BFLA 
Broken Function Level Authorization (BFLA) ist eine Autorisierungs-Schwachstelle, bei der ein Benutzer Funktionen (Endpunkte/Actions) ausführen kann, für die seine Rolle nicht berechtigt ist. Es geht dabei um den Funktions- bzw. Feature-Level, nicht primär um einzelne Datenobjekte.

## TL;DR
- Admin-Funktionen wie DELETE /identity/api/v2/admin/videos/{id} sind ohne Rollenprüfung für jeden Benutzer aufrufbar.
- Ein normaler User kann damit beliebige Videos löschen, obwohl dies eine privilegierte Admin-Funktion ist.
- Durch einfache Enumeration der Video-IDs wäre ein vollständiges Löschen aller Videos der Plattform möglich.
- Ursache: fehlende serverseitige Funktionsautorisation (BFLA) und keine Trennung zwischen normalen und privilegierten Endpunkten.

### BFLA
In crAPI sind für diese Schwachstelle alle Endpunkte interessant, die eine privilegierte Action (z.B. Admin Funktionalitäten) ermöglichen wie der Endpunkt *DELETE /identity/api/v2/admin/videos/{id}*

Wenn ein User den Endpunkt aufruft, wird fälschlicherweise nicht die Rolle des User überprüft, wodurch ein beliebiger User jedes beliebige Video auf der Plattform löschen kann:
<img width="1915" height="1077" alt="image" src="https://github.com/user-attachments/assets/e5bbaf06-df8c-41e9-8279-8a594d388915" />

Dieses Verhalten stellt ein erhebliches Sicherheitsrisiko dar:
Da die Video-IDs auf der Plattform ohne weitere Schutzmechanismen enumerierbar sind, könnte ein Angreifer eine automatisierte Anfragefolge erstellen, um sämtliche Videos der Plattform vollständig zu löschen. Ein solcher Angriff wäre trivial durchführbar und hätte unmittelbare Auswirkungen auf Verfügbarkeit, Datenintegrität sowie das Vertrauen der Benutzer.


## Fix:
- Serverseitige Rollen- und Berechtigungsprüfung
- Entfernen oder Härten interner Admin-Routen (ggfs. VPN/IP-Whitelisting)
- JWT Claims korrekt validieren
  
