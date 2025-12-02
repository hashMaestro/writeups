# crAPI Writeup

### Kontext / Zielsetzung
crAPI ist eine absichtlich verwundbare Web- und API-Anwendung, die typische OWASP API Top 10 Schwachstellen demonstriert.
Sie dient als legale, lokal selbst gehostete Trainingsplattform für Web-Pentesting und API-Security.

### TL;DR
- Traffic-Erfassung und Analyse mit mitmproxy/mitmweb 
- Vollständige API-Rekonstruktion und automatische Generierung einer OpenAPI-Spezifikation
- Aufbau einer Postman-Collection samt Environment als Testgrundlage
- Systematisches API-Testing und Proof-of-Concept-Exploitation ausgewählter Schwachstellen

### Ziel dieses Writeups:
- Dokumentation meiner praktischen Herangehensweise an das Testen einer API-basierten Anwendung
- Strukturierte Darstellung der wichtigsten technischen Schritte und Werkzeuge
- Transparente Darstellung des Analyse- und Entscheidungsprozesses
- Identifikation und begründete Bewertung relevanter Schwachstellen innerhalb der crAPI-Instanz

### Inhaltsverzeichnis
**1. Setup** <br>
- [Environment](environment.md)
- [Proxy](proxy.md)

**2. Reconnaissance**
- [API-Recon](api-recon.md)

**3. Findings**
- [Broken Object Level Authorization](API1-BOLA.md)
- [Broken Authentication](API2-Broken_Authentication.md)
- [Broken Object Property Level Authorization](API3-BOPLA.md)
- [Broken Function Level Authorization](API5-BFLA.md)
