# crAPI Writeup
### Kurzfassung (TL;DR)
- Traffic-Erfassung mit mitmproxy/mitmweb
- Vollständige API-Reconnaissance und automatische Erzeugung einer OpenAPI-Spezifikation
- Erstellung einer Postman-Collection und Environment als technische Basis für Tests
- API-Testing und Ausnutzung der OWASP Top10 Schwachstellen

### Kontext / Zielsetzung
crAPI ist eine absichtlich verwundbare Web- und API-Anwendung, die typische OWASP API Top 10 Schwachstellen demonstriert.
Sie dient als legale, lokal selbst gehostete Trainingsplattform für Web-Pentesting und API-Security.

#### Ziel dieses Writeups:
- meine Herangehensweise beim Testen dokumentieren
- die wichtigsten technischen Schritte festhalten
- die Analyse- und Denkprozesse sichtbar machen
- relevante Schwachstellen identifizieren und begründen
