# AiAccess — riconciliazione della ricostruzione col codice reale

Fonte: `p1/real/aiaccess-harness.html`
SHA-256 `ab1a289f69bdfb9b57d4f5b4408aaa3f45dd08ba937d22598138f29cf64a1163` · 17.589 byte

Il debito è chiuso: `p1/aiaccess.js` non è più una ricostruzione dal contratto,
è estratto dal codice ratificato. Sotto: dove la ricostruzione sbagliava, e i tre
difetti trovati nel codice reale eseguendolo in Chromium.

## 1. Dove la ricostruzione v1.0 divergeva

| Aspetto | Ricostruzione (sbagliata) | Codice reale |
|---|---|---|
| Gate GDPR | **assente** | `ConsentManager.hasConsent('ai')` prima di tutto → `AUTH_ERROR` |
| Cache | **assente** | lookup prima dei layer, hit con `provider: 'cache'` |
| Codici errore | `RATE_LIMIT` `AUTH` `UPSTREAM` `BAD_REQUEST` `NO_PROVIDER` | `RATE_LIMITED` `AUTH_ERROR` `NETWORK_ERROR` `TIMEOUT` `PROVIDER_ERROR` `NOT_CONFIGURED` |
| Cascata su errore non ritentabile | **si ferma** | **prosegue**: un 401 su un layer non dice nulla sugli altri |
| Busta | `{…, ms, attempts}` | `{…, model, groundingApplied, timestamp}` |
| `callQwenAI` | ritorna una **stringa** | ritorna un **oggetto** `{text}` \| `{error}` |
| Nessun provider | `NO_PROVIDER` | `NOT_CONFIGURED` + `needsSetup: true` |
| Priorità del parser | per apparizione | **per transitorietà**: 429 → timeout → network → auth |

Il caso `"auth timeout"` è documentato nel banco: vince `TIMEOUT` (ritentabile),
non `AUTH_ERROR`. La ricostruzione lo classificava all'opposto.

## 2. Tre difetti nel codice reale (riprodotti in Chromium)

### B1 — bloccante: il gate blocca *sempre*

```
hasConsent_diretto:    true      ← ConsentManager.hasConsent('ai')
window_ConsentManager: "undefined"
hasConsent_via_window: false     ← quello che ask() legge davvero
```

`ask()` verifica `window.ConsentManager?.hasConsent('ai')`, ma in uno `<script>`
classico un `const ConsentManager = {…}` **non** finisce su `window`. Con il
consenso attivo nell'interfaccia, ogni chiamata torna
`AUTH_ERROR — "Consenso AI non attivo"`. Nel banco tutta la cascata è morta:
nessun layer viene mai raggiunto.

Se lo stesso schema è nel file AiEvents reale, è una spiegazione meccanica del
fatto che la parte AI «funziona sempre e solo Gemini» o non funziona affatto:
non è la qualità del modello, è il gate che non vede il consenso.
**Da verificare sul sorgente 2.9.25 prima di trattarla come causa confermata.**

### B2 — la cache perde fonti e grounding

```
primo giro:   provider gemini · sources 3 · grounding true
secondo giro: provider cache  · sources 0 · grounding false
```

`_wrap(cached.text, 'cache', null, false, [])` scarta ciò che era stato salvato.
Una risposta con fonti torna senza fonti: con `fonti_obbligatorie` il re-hit
fallisce la verifica pur avendo dato la stessa risposta. Il fix «cache→sources»
di Merge A **non è presente in questo file**.

### B3 — la chiave di cache ignora il flag

```
ask('spiegami x')                    → gemini
ask('spiegami x', {flag:'aiNerd'})   → cache   (testo identico)
```

La chiave è il solo prompt: la richiesta AiNerd riceve la risposta normale.
Il flag è wired ma silenziosamente scavalcato dalla cache.

## 3. Cosa è stato corretto in `p1/aiaccess.js` v1.1

B1 il gate è **iniettato** (`deps.consent`), non cercato in una globale.
B2 la cache salva e restituisce `sources`, `groundingApplied`, `model`.
B3 la chiave include prompt + system + flag + modello.

Contratto pubblico invariato: stessi codici, stessa busta, stesso wrapper,
stessa cascata che prosegue sugli errori non ritentabili.

## 4. Proposta aperta (cambia il contratto — serve ratifica)

Consenso mancante e 401 del provider condividono `AUTH_ERROR` e sono
indistinguibili dal chiamante: la UI non può dire «attiva il consenso» invece di
«chiave non valida». Qui è stato aggiunto solo `error.needsConsent: true`
(additivo, non rompe nulla). Un codice dedicato `CONSENT_MISSING` sarebbe più
pulito ma è un cambio di contratto: non applicato.

## 5. Resta aperto

`aiaccess.js` è ora fedele al **banco di prova**, dove i provider sono simulati.
Le firme HTTP reali (endpoint, payload, parsing risposta di Gemini e DashScope)
vivono in `AiEvents-2.9.25` e non sono ancora state viste. `AiJob-0.28.0.html`
resta l'altro artefatto mancante.
