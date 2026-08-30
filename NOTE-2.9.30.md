# AiEvents 2.9.30 — swipe corretto, AiAccess integrato, AiWorld operativo

Catena: `2.9.27` (base) → `2.9.28` AiAccess → `2.9.29` swipe → `2.9.30` runtime AiWorld
458.726 byte · SHA-256 nel MODULE_REGISTRY

## 1. Bug swipe — causa e correzione

**Causa (riprodotta, non dedotta).** La valutazione del gesto *durante* il
movimento non controllava `blocked`:

```js
document.addEventListener('pointermove', e => {
  if (x0 === null) return;        // ← manca il controllo di blocked
  ...
  if (dt <= 800 && Math.abs(dx) >= 48 ...) go(dir);
```

`finish()` (al rilascio) rispettava le zone escluse, `pointermove` no. Con la
soglia di 48px superata **durante** il gesto, uno swipe dentro la mappa, la chat
agenti, la galleria o un campo di testo cambiava schermata. Traccia del difetto
catturata in Chromium:

```
["BLOCCO: zona INPUT", "MOVE:OK-ON-MOVE (dx-70 81ms)"]
 ↑ zona riconosciuta come esclusa      ↑ ...e navigato lo stesso
```

**Cinque correzioni:**

| | Difetto | Correzione |
|---|---|---|
| 1 | `pointermove` ignorava `blocked` (root cause) | ora esce se `blocked` |
| 2 | un secondo dito riazzerava l'origine: il pinch navigava | gesto tracciato per `pointerId`, il secondo puntatore lo annulla |
| 3 | `blocked` restava vero fino al gesto successivo | azzerato in `finish()` e su `pointercancel` |
| 4 | il ramo modale non azzerava `x0` | azzerato |
| 5 | toast `[swipe] percorso: ...` mostrato all'utente | resta in `window.__swipePath`, niente toast |

Preservato di proposito: un carosello che **non** scorre continua a lasciar
passare lo swipe (era una correzione precedente, verificata da S3b).

**Esito: 9/9.** Sul file 2.9.27 la stessa suite dava 5/9.

## 2. AiWorld dentro AiEvents

Un blocco `<script>` aggiunto in fondo, generato da `bundle.js` (11 moduli, tutti
UMD: la concatenazione basta, nessun bundler). L'app diventa un **nodo** del bus.

- I tre motori di AiAccess sono registrati come esecutori: la chiamata resta di
  AiAccess, **sopra** ci va l'orchestrazione — routing, retry, verifica, gate,
  persistenza.
- `EXT-CLAUDE-01` è un `browser_agent` in modalità testo: l'handoff viene reso
  in formato AL sigillato e copiato negli appunti; la risposta rientra dal parse.
  Le sue capability nascono **UNVERIFIED**.
- `AiEvents` risponde alle task che arrivano dal bus, e pubblica le proprie
  capability.

Uso, dalla console:

```js
AiWorldRuntime.agents.run({ intent:'search_events', context:{prompt:'...'},
  constraints:{capabilities:['search_events']}, success_condition:['C-14','C-15'] })

AiWorldRuntime.__claude.adapter.execute(handoff, {})   // rende il testo e attende
AiWorldRuntime.__claude.testo()                        // il blocco da incollare
AiWorldRuntime.__claude.rispondi(rispostaIncollata)    // chiude il giro
```

**Esito: 10/10**, fra cui: il gate umano regge dentro l'app (un `deploy` senza
gate si ferma su P4), un altro modulo sullo stesso bus riceve risposta, e se
Claude altera un vincolo nell'handoff il runtime lo rileva (`VERIFY_FAILED`).

## 3. Un difetto trovato di rimbalzo, non corretto

Migliorando la sonda dei duplicati è emerso che **`openKeyModal` è dichiarata
due volte a colonna 0** — righe 2890 e 3771 di 2.9.27, già presente in 2.9.25.
Non è un crash: in JavaScript la seconda dichiarazione vince in silenzio. Ma la
prima (2890) è **codice morto**: chi la modifica non vede alcun effetto. La
seconda è la versione più recente, quella che gestisce `agents-save` e
`agents-unlock`.

Non l'ho toccata: è una decisione di prodotto, non un fix meccanico.

## 4. Cosa NON è verificato

- **Provider reali.** Proxy e DashScope sono stub. La cascata è dimostrata, le
  firme HTTP vere no.
- **Layer Gemini.** `sessionAgentKeys` è un `let`: non raggiungibile
  dall'esterno per essere sostituito in un test. Va provato sul dispositivo.
- **Swipe con dita vere.** I gesti sono Pointer Events di Chromium headless:
  reali come eventi, non come tocco su vetro. Il multi-touch è sintetico.
- Il peso del file cresce da 348 KB a 459 KB (+32%).
