# SOC-Hub — Guida all'utilizzo per il Team

> Documento operativo destinato agli analisti SOC. Copre accesso, gestione delle credenziali, architettura e workflow dei due Hub disponibili: **Intel-Hub** (Identity Intelligence) e **Phish-Hub** (Email & Phishing OSINT).

---

## Indice

1. [Introduzione](#1-introduzione)
2. [Accesso & Credenziali](#2-accesso--credenziali)
3. [Overview Architetturale](#3-overview-architetturale)
4. [Guida Operativa — Setup iniziale](#4-guida-operativa--setup-iniziale)
5. [Guida Operativa — Intel-Hub](#5-guida-operativa--intel-hub)
6. [Guida Operativa — Phish-Hub](#6-guida-operativa--phish-hub)
7. [Troubleshooting & FAQ](#7-troubleshooting--faq)
8. [Riferimenti rapidi](#8-riferimenti-rapidi)

---

## 1. Introduzione

**SOC-Hub** è una suite di strumenti OSINT browser-based pensata per supportare le attività investigative del SOC. Si compone di tre elementi:

| Componente | Tipo | Funzione |
|---|---|---|
| **SOC-Hub Suite (OSINT WebApp)** | Frontend statico (HTML/JS) | Portale + Intel-Hub + Phish-Hub |
| **SOC-Hub Proxy** | Micro-servizio Node.js | CORS proxy per le chiamate verso le API OSINT |
| **Threat-Hub** | *Coming soon* | Placeholder visibile nel portale |

Tutta la logica investigativa gira **lato client** (browser): nessun server applicativo, nessun database, nessuna telemetria centralizzata. Le credenziali ai servizi OSINT (IntelX, Hunter.io, VirusTotal, ecc.) sono memorizzate in `localStorage` **cifrate AES-GCM-256** con la Master Key dell'analista.

---

## 2. Accesso & Credenziali

### 2.1 Master Key (autenticazione alla suite)

L'accesso alla suite è protetto da una singola **Master Key** (passphrase a 32 caratteri base64). Lo schema di autenticazione è:

1. L'analista digita la Master Key nella login del portale.
2. Il browser ne calcola lo SHA-256 e lo confronta con un hash hardcoded nel codice della WebApp (`MASTER_KEY_HASH`).
3. In caso di match, viene aperta una sessione di **8 ore** legata alla scheda del browser (`sessionStorage`).
4. La chiusura della scheda invalida la sessione.

**La Master Key NON è mai memorizzata in chiaro né su disco né su server.** Viene utilizzata solo come passphrase per (a) confronto hash in fase di login e (b) derivazione della chiave AES per cifrare le API key di servizio in `localStorage`.

#### Come ottenere la Master Key

> ⚠️ **La Master Key non è contenuta nel repository, in `.env`, in CI o in alcun artefatto di build.**

- È custodita nel **vault condiviso del SOC** (es. Bitwarden / 1Password — chiedere al referente di team).
- Per nuovi onboarding: richiedere accesso al vault al **SOC Lead**, poi recuperare l'entry "SOC-Hub — Master Key".
- Per il reset / recovery contattare l'owner della suite all'email di recovery configurata in repo (vedi `index.html`, costante `RECOVERY_EMAIL`).

#### Buone pratiche
- **Non condividere** la Master Key via chat, email o ticket.
- Non scriverla in note locali, screenshot o documenti collaborativi.
- Cambio Master Key: ruotare la passphrase nel vault, generare nuovo hash con `echo -n "<nuova>" | sha256sum`, aggiornare `MASTER_KEY_HASH` nel codice e ridistribuire (fuori scope di questa guida).

### 2.2 API Key dei servizi OSINT

Ogni Hub richiede un set distinto di credenziali verso servizi terzi. Vanno inserite **una sola volta** nella tab **Configuration** del rispettivo Hub: vengono cifrate AES-GCM-256 con la Master Key e persistite in `localStorage`.

#### Intel-Hub
| Servizio | Dove ottenerla | Tier consigliato |
|---|---|---|
| **IntelX** | https://intelx.io → account → API Key | Free 50/giorno; paid se necessario |
| **RapidAPI (BreachDirectory)** | https://rapidapi.com → BreachDirectory API → Subscribe | Free 50/mese |
| **LeakIX** | https://leakix.net → account → API Key | Researcher tier (free) |
| **DeHashed** | https://dehashed.com → My Account → API Credentials (email + key) | Subscription a pagamento |
| **Anthropic Claude** *(opzionale)* | https://console.anthropic.com → API Keys | Pay-as-you-go |

#### Phish-Hub
| Servizio | Dove ottenerla | Tier consigliato |
|---|---|---|
| **Hunter.io** | https://hunter.io → API → Generate Key | Free 25/mese |
| **EmailRep** *(opzionale)* | https://emailrep.io | Anonimo OK (rate-limit per IP) |
| **URLScan.io** | https://urlscan.io → User → API Key | Free |
| **VirusTotal** | https://virustotal.com → User → API Key | Free 4 req/min |
| **Anthropic Claude** *(opzionale)* | https://console.anthropic.com → API Keys | Pay-as-you-go |

> Le chiavi dei servizi a pagamento condivisi tra il team (es. DeHashed, IntelX paid) sono nel medesimo vault della Master Key. **Recuperarle dal vault, non rigenerarle** salvo coordinamento col SOC Lead.

### 2.3 Bundle di configurazione (deploy unificato)

Per evitare che ogni analista incolli manualmente tutte le API key, l'owner può:

1. Configurare un'istanza completa.
2. Cliccare **Configuration → Generate bundle**.
3. Ottiene un blob JSON cifrato (con la Master Key) da incollare nel sorgente HTML pre-deploy.
4. Al login, gli analisti che digitano la Master Key vedono le API key già pre-popolate.

> 🔐 Il bundle è inutile senza la Master Key: la cifratura è AES-GCM-256 con derivazione PBKDF2.

### 2.4 CORS Proxy

Le API OSINT non espongono CORS verso il browser, quindi le chiamate passano per un **CORS proxy** dedicato. La URL del proxy va configurata nella tab **Configuration** di entrambi gli Hub (campo *CORS Proxy URL*).

Opzioni di deploy del proxy (codice già pronto nei due HTML degli Hub, sezione Configuration → Deploy):
- **Cloudflare Worker** (consigliato — free tier sufficiente, low-latency).
- **Render.com** Node.js service (alternativa).
- Locale per sviluppo: `node server.js` da `SOC-Hub-Proxy/` (porta 3000, env `PORT` per override).

I proxy pubblici (`corsproxy.io`, `api.allorigins.win`) sono accettati come **fallback degradato** ma stripping degli header di Authorization → da evitare in operations reali.

---

## 3. Overview Architetturale

### 3.1 Diagramma a blocchi

```
            ┌──────────────────────────────────────────────────────┐
            │                    Analista SOC                       │
            │                       (browser)                       │
            └──────────────────────────────┬───────────────────────┘
                                           │ HTTPS
                                           ▼
            ┌──────────────────────────────────────────────────────┐
            │           SOC-Hub Suite — Static Site                 │
            │           (Cloudflare Pages / hosting statico)        │
            │  ┌────────────┐ ┌──────────────┐ ┌────────────────┐  │
            │  │  Portale   │ │  Intel-Hub   │ │   Phish-Hub    │  │
            │  │ index.html │ │ identity-... │ │ phish-hub.html │  │
            │  └────────────┘ └──────────────┘ └────────────────┘  │
            │   • Auth Master Key (SHA-256, sessionStorage)         │
            │   • API key cifrate AES-GCM (localStorage)            │
            │   • i18n EN/IT, dark/light theme, sidebar SPA         │
            └──────────────────────────────┬───────────────────────┘
                                           │ fetch() HTTPS
                                           ▼
            ┌──────────────────────────────────────────────────────┐
            │           SOC-Hub Proxy (CORS bypass)                 │
            │  Cloudflare Worker  |  Render.com  |  Localhost:3000  │
            │  Express + node-fetch • timeout 25s • body 5MB        │
            └──────────────────────────────┬───────────────────────┘
                                           │
       ┌───────────────────────────────────┼────────────────────────────────────┐
       ▼                                   ▼                                    ▼
  Intelligence APIs                   Email/Phish APIs                       AI
  • IntelX                            • Hunter.io                          • Anthropic
  • HIBP Pwned Pwd                    • EmailRep                            Claude
  • BreachDirectory (RapidAPI)        • VirusTotal
  • LeakIX                            • URLScan.io
  • DeHashed                          • DoH (DNS audit SPF/DKIM/DMARC)
```

### 3.2 Componenti

#### SOC-Hub Suite — OSINT WebApp
- **Repository:** `SOC-Hub-Suite---OSINT-WebApp`
- **Stack:** HTML5, Vanilla JS (ES6+), CSS custom properties (no framework UI).
- **Librerie CDN:** `lucide` (icone), `marked` (markdown render), `DOMPurify` (sanitize), Web Crypto API (hash + AES).
- **File principali:**
  - `index.html` — portale di login + selezione Hub (~700 righe).
  - `identity-intelligence-hub.html` — Intel-Hub (~5100 righe).
  - `phish-hub.html` — Phish-Hub (~5200 righe).
- **CSP rigorosa** (meta tag in testa): whitelist dei soli domini API necessari.
- **Persistenza:**
  - `sessionStorage`: token di sessione, timestamp (8h).
  - `localStorage`: API key cifrate, lingua, history ricerche.

#### SOC-Hub Proxy
- **Repository:** `SOC-Hub-Proxy`
- **Stack:** Node.js + Express 4 + node-fetch 2.
- **Endpoint unico:** `GET|POST|PUT|… /?<URL_target_encoded>` → forward verso `<URL_target>` con header CORS permissivi.
- **Hardening minimo:** rimozione di `host`/`origin` in upstream, timeout 25s, body cap 5MB.
- **No state, no DB, no auth** — l'autenticazione resta all'API target tramite header originali del client.

### 3.3 Flusso di autenticazione

```
[Browser]                                [WebApp statica]
    │                                          │
    │ 1. GET /                                 │
    ├─────────────────────────────────────────▶│
    │                                          │
    │ 2. Login form                            │
    ◀─────────────────────────────────────────┤
    │                                          │
    │ 3. Master Key → SHA-256(client-side)     │
    │ 4. Confronto con MASTER_KEY_HASH         │
    │    (hardcoded in HTML)                   │
    │                                          │
    │  ✓ match → sessionStorage[SESSION] = 1   │
    │            sessionStorage[SESSION_TS]    │
    │                                          │
    │ 5. (se presente) decrypt bundle config   │
    │    AES-GCM con Master Key → API keys     │
    │                                          │
    │ 6. Redirect a Hub scelto (?returnTo=…)   │
    ◀─────────────────────────────────────────┤
```

### 3.4 Flusso dati di una ricerca (esempio Intel-Hub)

```
1. Analista compila keyword + tipo + servizi → "Run Search"
2. Per ciascun servizio attivo:
     a. Compone request (header con API key del servizio)
     b. fetch() → SOC-Hub Proxy → API OSINT
     c. Risposta normalizzata in oggetto "finding"
3. Aggregazione client-side dei finding (dedup per email/domain/IP/hash)
4. (Opzionale, se Anthropic configurato + AI Consent ON):
     a. Build prompt con findings aggregati
     b. fetch() → Proxy → api.anthropic.com
     c. Render Markdown del report Claude
5. Storage in-memory + localStorage (history)
```

### 3.5 Modello di sicurezza (sintesi)

- **Confidenzialità delle API key:** AES-GCM-256, key derivata via PBKDF2 dalla Master Key. Senza Master Key il blob è inutilizzabile.
- **Separation of duties:** ogni analista usa la propria istanza browser; nessuna telemetria centralizzata; cronologia ricerche solo locale.
- **Limiti noti:**
  - L'hash della Master Key è incluso nel bundle HTML servito (necessario per auth client-only) → **forza brute-force sull'hash**: la Master Key DEVE essere ad alta entropia (≥256 bit).
  - Nessuna revoca server-side: in caso di compromissione, ruotare Master Key + ridistribuire build + invalidare API key servizi.

---

## 4. Guida Operativa — Setup iniziale

### 4.1 Prerequisiti

- Browser moderno (Chrome/Edge/Firefox aggiornato, supporto Web Crypto API).
- Accesso al vault SOC (Master Key + API key servizi a pagamento).
- URL del CORS proxy SOC (chiedere al SOC Lead).

### 4.2 Primo accesso

1. Aprire l'URL della suite (deploy su Cloudflare Pages — chiedere al referente).
2. Inserire la Master Key.
3. Selezionare l'Hub desiderato dalla home.
4. Andare in **Configuration** (sidebar):
   - Inserire le API key dei servizi che si intende usare.
   - Inserire la **CORS Proxy URL**.
   - Salvare.
5. Verificare nella stessa tab che ogni servizio mostri lo stato ✅ *Configured*.

### 4.3 (Solo manutentore) Avvio del proxy locale

```bash
cd SOC-Hub-Proxy
npm install
npm start                # default porta 3000
PORT=5000 node server.js # porta custom
```

Health-check:
```bash
curl 'http://localhost:3000/?https://httpbin.org/status/200' -i
```

---

## 5. Guida Operativa — Intel-Hub

**Scopo:** indagini di **Identity Intelligence & Breach Detection** su email, nominativi, telefoni, domini, IP, URL, password.

### 5.1 Layout

Sidebar a sinistra: **Portal · Operations · Search · Results · System · Rate Limits · Configuration**.

### 5.2 Workflow tipico

1. Tab **Search**:
   - **Keyword** (es. `mario.rossi@example.com`).
   - **Type** (email, name, phone, domain, ip, url, password).
   - **Active Services** (checkbox: IntelX, BreachDirectory, LeakIX, DeHashed, HIBP).
   - **Run Search**.
2. Tab **Results**: badge per servizio + count + latency, drill-down findings.
3. **AI analysis** (se abilitata): report in Markdown generato da Claude — sintesi delle breach, indicatori correlati, raccomandazioni.
4. Tab **Rate Limits**: monitorare quota residua per servizio (daily/monthly/lifetime).
5. **History** in sidebar: ri-eseguire o consultare query passate.

### 5.3 Casi d'uso operativi

- **Compromise check** di un account utente segnalato (email → IntelX + BreachDirectory + DeHashed + HIBP).
- **OSINT su dominio** sospetto (domain → LeakIX, IntelX).
- **Verifica password** (password type → solo HIBP, k-anonymity, nessuna password lascia il browser intatta).
- **Pivot da numero telefonico** o nome.

### 5.4 Integrazione SOAR / playbook

Dalla Configuration è possibile recuperare la **SOAR URL**:

```
<APP_URL>?apikey=<MASTER_KEY>&q=<keyword>&type=<email|domain|...>
```

Da invocare da Cortex SOAR / SIEM per pre-popolare una ricerca. ⚠️ La Master Key in querystring non è raccomandata in canali non protetti — usare canali interni only.

---

## 6. Guida Operativa — Phish-Hub

**Scopo:** investigazione su **email, mittenti, domini e URL sospetti**, supporto a triage di phishing segnalati (es. da Defender for O365, user reports).

### 6.1 Layout

Layout speculare a Intel-Hub: sidebar con sezioni Portal, Operations, Search, Results, Rate Limits, Configuration.

### 6.2 Tipologie di query

| Query Type | Cosa fa | Servizi coinvolti |
|---|---|---|
| **Email validation** | Verifica esistenza/validità mailbox | Hunter.io |
| **Domain search** | Email + metadati di un dominio | Hunter.io |
| **Email reputation** | Score reputazionale, flag fraud/spam | EmailRep |
| **DNS audit** | Controllo SPF/DKIM/DMARC | DoH (Cloudflare/Google) |
| **URL detonation** | Analisi URL/dominio sospetto | VirusTotal, URLScan.io |

### 6.3 Workflow tipico (triage di phishing)

1. **Domain search** sul dominio mittente → reputation, MX, prima registrazione.
2. **DNS audit** → presenza/correttezza SPF, DKIM, DMARC (`p=`).
3. **Email reputation** dell'indirizzo specifico.
4. **URL detonation** sui link contenuti (VirusTotal verdict, URLScan screenshot).
5. (Opzionale) **AI synthesis** Claude: report che correla i 4 punti precedenti in formato comunicabile.
6. Risultati esportabili copiando dal risultato (Markdown).

### 6.4 Note operative

- EmailRep funziona anche **senza API key** (anonimo) ma con rate-limit per IP — utile in pinch, non per volumi.
- VirusTotal free è **4 req/min**: in caso di triage massivo schedulare le query.
- URLScan.io: distinguere scan **public** (visibili a tutti) vs **unlisted/private** in base alla sensibilità del target.

---

## 7. Troubleshooting & FAQ

| Sintomo | Causa probabile | Risoluzione |
|---|---|---|
| Login non accetta la Master Key | Refuso o copia con whitespace | Ricopiare dal vault senza spazi/newline |
| "CORS error" / fetch failed | Proxy non configurato o offline | Configuration → impostare CORS Proxy URL valido; verificare con `curl '<proxy>/?https://httpbin.org/status/200'` |
| Servizio mostra ⚠️ *Not configured* | API key mancante o rigettata | Re-inserire la chiave in Configuration; verificarla sul portale del servizio |
| Sessione scaduta dopo 8h | TTL `SESSION_TTL_MS` raggiunto | Re-login (le API key restano in localStorage) |
| Rate limit hit | Quota giornaliera/mensile esaurita | Tab **Rate Limits** per stato; attendere reset o passare a tier paid |
| AI analysis non parte | API key Anthropic mancante o consent OFF | Configuration → inserire chiave + togglare AI Consent |
| Risultati persi cambiando scheda | sessionStorage isolato per scheda | Lavorare in una sola scheda; export risultati prima di chiudere |
| Browser dice "blocked by CSP" | URL fuori whitelist CSP | Verificare che il dominio del proxy sia incluso nella `Content-Security-Policy` (`identity-intelligence-hub.html` / `phish-hub.html`, meta tag in testa) |

### FAQ

**Q: Le mie API key sono al sicuro se mi rubano il laptop?**
Sì, finché non rivelano la Master Key: il blob in `localStorage` è AES-GCM-256. Comunque, in caso di smarrimento avvisare il SOC Lead per ruotare la Master Key e revocare le API key di servizio.

**Q: Posso usare SOC-Hub da uno smartphone?**
Tecnicamente sì (è un sito statico), ma le UI sono pensate desktop. Non raccomandato.

**Q: Le ricerche fatte sono tracciate centralmente?**
No. La history è solo nel `localStorage` del browser dell'analista. I provider OSINT vedono però le query ricevute (privacy del target — usare con disciplina).

**Q: Come elimino traccia delle ricerche?**
DevTools → Application → Local Storage → cancellare entry `sochub-*`. Oppure logout + clear browser data.

**Q: Posso aggiungere un nuovo servizio OSINT?**
Sì, ma richiede modifica del codice degli HTML (handler fetch + parser + voce in Configuration + voce in CSP). Richiesta da girare al manutentore della suite.

---

## 8. Riferimenti rapidi

- **Recovery Master Key:** vedere costante `RECOVERY_EMAIL` in `index.html`.
- **Hash Master Key (SHA-256):** costante `MASTER_KEY_HASH` in `index.html`, `identity-intelligence-hub.html`, `phish-hub.html`.
- **TTL sessione:** 8 ore (`SESSION_TTL_MS`).
- **Repo:**
  - `SOC-Hub-Suite---OSINT-WebApp` — frontend.
  - `SOC-Hub-Proxy` — CORS proxy.
- **Deploy raccomandato:** Cloudflare Pages (frontend) + Cloudflare Worker (proxy).
- **Vault credenziali:** vault SOC condiviso (richiedere accesso al SOC Lead).

---

*Documento mantenuto dal team SOC. Per modifiche o dubbi, aprire issue sul repo `SOC-Hub-Suite---OSINT-WebApp`.*
