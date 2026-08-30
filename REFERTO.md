# AiEvents 2.9.25 / 2.9.26 / 2.9.27 — referto di verifica

Tre file ricevuti ed eseguiti. Tutto quanto segue è **misurato**, non dedotto.

## 1. Identità e genealogia

| File | Byte | Righe | SHA-256 |
|---|---|---|---|
| AiEvents2.9.25.html | 334.898 | 5.384 | `bb290dc1a8826e70580484a89618e24e42c85da5484e2477774e26ada275408b` |
| AiEvents2.9.26.html | 336.473 | 5.404 | `67cc5ce4d31c93b245e3c449170bb04cd698efb05aecb1dc4780c558cdf2a409` |
| AiEvents2.9.27.html | 347.807 | 5.604 | `2933cd54ce5dc041e3876edb958beb124a01aeeb5324bfa2e3d2d333e91f4e3a` |

Cosa cambia fra le versioni:

- **2.9.25 → 2.9.26** — 26 righe: correzioni OAuth Google (`[FIX-OAUTH]`), fra cui il caso `file://` in cui `location.origin` vale la stringa `"null"` e il `redirect_uri` diventa `null/C:/...` → 400 da Google.
- **2.9.26 → 2.9.27** — 208 righe: gestione chiavi agente, `TavilySearch`, UI Gemini/agenti, GSync.

## 2. Salute strutturale: tutte e tre CONFORMI

```
12 blocchi <script> inline · tutti superano node --check
nessuna classe/const dichiarata due volte
6/6 asserzioni · GREEN
```

Nessuna traccia del difetto che aveva rotto `noPassphrase.html`.

## 3. B1 — NON PRESENTE, confermato su tutte e tre

```
AiEvents2.9.25.html → B1: NON RILEVATO
AiEvents2.9.26.html → B1: NON RILEVATO
AiEvents2.9.27.html → B1: NON RILEVATO
```

`window.ConsentManager = {` alla riga 3825. Qwen aveva ragione. Il rilevatore ha
controllato **ogni** identificatore letto via `window.`, non solo `ConsentManager`:
nessun altro caso.

## 4. Il ritrovamento che cambia P1: AiAccess non esiste come modulo

`AiAccess` compare 5–7 volte nel file, e **sono tutte etichette di interfaccia**:
il pill di stato, il titolo del modale, `openAiAccess()`. Non c'è nessun
`AiAccess.ask()`, nessun `contractVersion`, nessuna busta v1.0.

**Il facade AiAccess vive solo nel banco di prova. In AiEvents non è mai stato
integrato.** Quindi P1 non era un'estrazione: è un'integrazione ancora da fare.
La ricostruzione, la riconciliazione e il debito dichiarato partivano tutti dal
presupposto sbagliato che il facade fosse già dentro l'app.

Assenti anche: `AiBridge` (0), `AiAgentsCore` (0), `contractVersion` (0),
`AiTelemetry` (0), `SYSTEM_PROMPTS` (0). Presenti: `DeviceVault` (8),
`BaseGraph` (8), `PeopleGraph` (3), `AI_RULES` (3).

## 5. Il percorso AI reale — e perché «funziona sempre e solo Gemini»

`callQwenAI` (riga 4545 in 2.9.27) è **identico nelle tre versioni** (stesso
md5): il percorso AI non è mai stato toccato da 2.9.25 a 2.9.27.

```js
if (getProxyCfg().url) {
  result = await callCloudProxy(prompt, '...');          // ← se c'è il proxy, SOLO il proxy
} else if (sessionAgentKeys && sessionAgentKeys.gemini) {
  result = await callAgentCore('gemini', prompt, {...}); // ← altrimenti SOLO Gemini
} else {
  ...dashscope...                                        // ← altrimenti SOLO DashScope
}
```

### F1 — nessun fallback: è una scelta esclusiva, non una cascata (bloccante)

`if / else if / else` sceglie **un** layer in base alla configurazione, non al
risultato. Se il proxy risponde 429, 500 o va in timeout, `callCloudProxy`
ritorna `{error}` e `callQwenAI` **restituisce quell'errore**: Gemini e
DashScope non vengono mai provati, benché configurati.

È il «manca il fallback provider dopo un errore 429 da Gemini» del backlog —
ancora presente in 2.9.27 — ed è la spiegazione meccanica di «funziona sempre e
solo Gemini»: chi ha una chiave Gemini e nessun proxy finisce sempre sullo
stesso ramo, qualunque cosa succeda.

### F2 — nessun contratto di risposta

`callQwenAI` ritorna `{text}` oppure `{error: 'HTTP 500 ...'}`. Nessun
`contractVersion`, nessun `success`, nessun `retryable`, nessun `sources`,
nessun `groundingApplied`. Il Response Contract v1.0 non è implementato qui.

### F3 — nessuna classificazione degli errori

`{ error: 'HTTP ' + r.status }`: 429, 401 e 500 sono la stessa cosa per il
chiamante. Nessuno può decidere se ritentare, cambiare motore o chiedere una
chiave. Senza F3, F1 non è nemmeno riparabile: per fare fallback bisogna prima
sapere **perché** si è fallito.

### F4 — chiave di cache = solo il prompt

`AI_CACHE._key = SHA-256(version + '::' + prompt)`. Il system prompt non entra
nella chiave, e i due rami usano system prompt diversi. La stessa domanda posta
con il proxy e poi senza restituisce la risposta del primo ramo.

### F5 — la cache non perde le fonti (B2 assente qui)

`AI_CACHE.get` restituisce l'oggetto salvato così com'era: se `callAgentCore`
aveva prodotto `sources`, sopravvivono. Il difetto B2 del banco **non** si
riproduce nel file reale: là c'era un `_wrap(cached.text, 'cache', null, false, [])`
che qui non esiste. La cache scrive solo in caso di successo
(`if (result && !result.error)`), ed è corretto.

## 6. Conseguenze sulla roadmap

1. **P1 va riclassificato**: non «AiAccess estratto», ma «AiAccess **da
   integrare**». Il modulo `p1/aiaccess.js` resta valido come contratto — ora
   però è chiaro che sostituirebbe `callQwenAI`, non lo avvolgerebbe.
2. **F1 è il difetto con più impatto d'uso di tutta la lista.** È anche il più
   piccolo da chiudere: `callQwenAI` diventa un ciclo sui layer configurati
   invece di un `if/else if`, e questo è esattamente ciò che `AiAccess.ask()`
   già fa e che i test AX3/AX4 già coprono.
3. **F3 precede F1**: la classificazione degli errori è il prerequisito del
   fallback. `AiAccess.parseError` esiste già ed è testato (AX5).
4. Il fix va applicato **una sola volta**, integrando il facade: rattoppare
   `callQwenAI` sul posto ricrea due percorsi divergenti da riconciliare dopo.

## 7. Cosa NON ho verificato

- Il comportamento a runtime con chiavi vere: i tre file non sono stati eseguiti
  in un browser con provider reali, quindi F1 è letto nel codice, non osservato
  in esecuzione.
- Il bug swipe (navigazione laterale) resta aperto: presente il codice, non
  indagato in questa passata.
