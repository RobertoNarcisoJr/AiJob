# AiEvents 2.9.28 — integrazione di AiAccess

Base: `AiEvents2.9.27.html` · SHA-256 `2933cd54ce5dc041e3876edb958beb124a01aeeb5324bfa2e3d2d333e91f4e3a`

## Cosa è cambiato

**Una sola sostituzione**, nello stesso blocco `<script>`: il corpo di
`callQwenAI` (riga 4545) è stato rimpiazzato dal facade `AiAccess` più un
adattatore retrocompatibile. **Nessun chiamante è stato modificato** — le
quattro chiamate a `callQwenAI` sono identiche riga per riga, solo spostate.

### Prima

```js
if (getProxyCfg().url)            result = await callCloudProxy(...);
else if (sessionAgentKeys?.gemini) result = await callAgentCore('gemini', ...);
else                               /* dashscope */
```

Sceglie **un** layer in base alla configurazione. Se quel layer fallisce,
la richiesta finisce lì.

### Dopo

```js
for (const layer of this._layers()) {
  if (!(await layer.enabled())) continue;
  const r = await layer.call(prompt, system);
  if (r && !r.error && r.text) return this._wrap(...);
  lastError = this.parseError(r.error);      // e si prosegue
}
```

Cascata vera: prosegue finché un layer risponde. Ordine invariato
(proxy → gemini → dashscope), quindi chi funzionava prima continua a
funzionare allo stesso modo.

## Cosa risolve

| # | Difetto | Stato |
|---|---|---|
| F1 | nessun fallback: un 429 sul proxy chiudeva la richiesta | **chiuso** |
| F2 | nessun contratto di risposta | **chiuso** — `AiAccess.ask()` ritorna la busta v1.0 |
| F3 | errori non classificati (`HTTP 500` grezzo) | **chiuso** — `parseError` con priorità alla transitorietà |
| F4 | chiave di cache = solo il prompt | **chiuso** — la chiave include un tag del system prompt |

## Cosa NON è cambiato

- `callQwenAI(prompt)` ritorna ancora `{text}` oppure `{error}` — più, in
  aggiunta, `provider`, `model`, `sources` quando ci sono.
- Il messaggio del gate consenso è identico, carattere per carattere.
- `needsSetup: true` e il testo del wizard sono preservati.
- `AI_CACHE`, `ConsentManager`, `callCloudProxy`, `callAgentCore`,
  `getApiKeySecure` non sono stati toccati.

## Verifica eseguita (Chromium reale, confronto prima/dopo)

Scena identica sui due file: proxy configurato che risponde 429, DashScope
disponibile.

```
CONFRONTO   proxy tentato / dashscope tentato / esito
  2.9.27         1 / 0 / ERRORE: AiGateway HTTP 429 rate limit
  2.9.28         1 / 1 / risposta DashScope
```

9/9 GREEN: forma legacy preservata, `needsSetup` conservato, busta v1.0
disponibile, errori classificati, cache che distingue i system prompt,
traccia della cascata ispezionabile, `health()` che distingue configurato da
attivo.

Struttura: 12 blocchi `<script>`, tutti superano `node --check`; nessuna
dichiarazione duplicata; B1 non rilevato.

## Come tornare indietro

Il file originale non è stato modificato: `AiEvents2.9.27.html` è intatto.
Il rollback è sostituire il file, niente altro.

## Cosa resta da fare

- **Prova con chiavi vere.** Qui i provider sono stati sostituiti da stub:
  la cascata è dimostrata, le firme HTTP reali di Gemini e DashScope no.
- Il layer `gemini` non è stato esercitato nel test (nessuna chiave in
  sessione): `sessionAgentKeys` è un `let`, non raggiungibile dall'esterno
  per essere stubbato. Va provato sul target con una chiave reale.
- Il bug della navigazione laterale (swipe) resta aperto: non toccato.
