# P6 — AiEval: la verifica diventa un gate, non una dichiarazione

## 1. Modale key orfano — rimosso (AiEvents 2.9.32)

Rimossi da `AiEvents-2.9.31`:

| elemento | dove era | perche' |
|---|---|---|
| markup `#modal-key` | riga 1208-1214 | unico chiamante di `confirmKeyModal()` |
| `confirmKeyModal()` | riga 3695-3739 | zero chiamanti dopo la rimozione del markup |
| `openKeyModal(mode)` | riga 3741-3756 | zero chiamanti gia' in 2.9.31 |
| `const hasBlob` | riga 3278 | variabile locale mai usata |

Fatto rilevato durante la rimozione: **il flusso era gia' inerte prima di oggi**.
L'input della passphrase era `type="hidden"`, quindi `pass` era sempre `''` e
`confirmKeyModal` usciva sempre su "Passphrase troppo corta". Non era codice
morto in attesa di essere chiamato: era codice che non poteva funzionare.

Percorso vivo, invariato: `saveAgentKeys()` -> `AgentVault` -> `DeviceVault`
(chiave legata al dispositivo, nessuna passphrase).

### Correzione a una mia affermazione (segnalata da Qwen)
Nella prima stesura di questa nota ho scritto *"SecureStorage resta: lo usa
ConsentManager"*. **E' sbagliato.** Verificato sul file:

- `ConsentManager` di AiEvents 2.9.32 (riga 3727) e' una versione **in-file v1.0**
  che scrive `aitrip_consents` in **localStorage in chiaro**: nessuna cifratura,
  nessun DeviceVault, nessun SecureStorage.
- `SecureStorage` (riga 2862) e' usato da `SecurityUtils.encryptData/decryptData`
  (riga 3797), i cui **unici chiamanti sono la suite di autotest interna**
  (riga 4054). Dopo la rimozione del modale key, `SecureStorage` non ha piu'
  chiamanti di produzione: e' raggiungibile solo dall'autotest.

Quindi la divergenza MD-07 esiste, ma non e' "DeviceVault vs SecureStorage":
e' **DeviceVault (kit H1, `p0/consent-manager.js` v1.1) vs localStorage in
chiaro (AiEvents 2.9.32)**. Va riconciliata prima di ratificare MD-07.
Non l'ho fatto in questo giro: e' una modifica al comportamento del consenso,
quindi gate umano.

### Punto aperto lasciato al gate umano
`aiConfigured()` considera ancora `localStorage['aitrip2_agent_keys_enc']` come
prova che l'AI sia configurata. Quel blob non e' piu' scrivibile ne' apribile.
Non ho toccato la funzione: il comportamento e' identico a prima della rimozione
(il modale era gia' inerte), quindi non e' una regressione introdotta oggi. Se
vuoi, il passo successivo e' togliere quel termine, cosi' un utente con un blob
legacy viene mandato al setup invece di vedere "AI attiva".

**File:** `AiEvents-2.9.32-aiworld.html` — 454405 byte
**SHA-256:** `2a8f0419d08e7feb44d98902ccf61027091077df6608d23b623bfe2dd65977c4`

Suite rieseguite sul file nuovo: P0 CONFORME, swipe 9/9, integrazione 9/9,
runtime 10/10, multimodulo 7/7.

---

## 2. P6 AiEval — cosa e' stato costruito

Ordine di verifica rispettato: test deterministici -> schema -> conteggi
generati -> browser -> secondo esecutore. Nessun LLM-giudice (DEC-20260825-07).

| file | responsabilita' |
|---|---|
| `p6/schema.js` | validatore di schema deterministico, zero dipendenze |
| `p6/aieval-runner.js` | esegue le suite e produce evidenza firmata |
| `p6/aieval-gate.js` | **il gate**: rifiuta ogni GREEN non dimostrato |
| `p6/ailanguage-eval.js` | AL-6 EvaluationRequest, AL-7 EvaluationReport |
| `p6/aieval-adapter.js` | innesto per motori esterni + crossCheck |

### Il gate, in concreto

La frase "un agente non puo' dichiarare GREEN senza evidenza machine-readable"
e' diventata codice che **non si fida del report**:

1. **schema chiuso** — un campo in piu' e' il report di un'altra cosa;
2. **conteggi ricalcolati dai casi** (AiEval-01): scrivere `27/27` a mano
   produce ora un errore che dice dichiarato / ricalcolato;
3. **sigillo** su esiti + conteggi: riscrivere un `FAIL` in `PASS` lo rompe;
4. **verdetto derivato**, non dichiarato;
5. **determinismo**: una suite che cambia esito tra due passate identiche e'
   RED, con le divergenze elencate. Un test instabile non e' "quasi verde".

Tre casi della suite (E7, E8, E9) sono esattamente i tre modi in cui un agente
puo' mentire su un test. Tutti e tre vengono rifiutati.

### Schema: cio' che non e' implementato viene rifiutato
Un vincolo ignorato in silenzio e' peggio di un vincolo assente, perche' da'
l'impressione di essere applicato. `validate()` valida prima **lo schema**:
una chiave non implementata e' un errore del progettista, non un no-op.

---

## 3. EXT-MATRAIX-01 — registrato, non integrato

```json
{"id":"EXT-MATRAIX-01","type":"external_evaluation_accelerator","status":"PROPOSED",
 "integration":"AiEvalAdapter","core_dependency":false,"priority":"H3",
 "human_evidence_required":true,"capabilities_verified":[],"mapping_status":"UNVERIFIED"}
```

Il core non e' stato toccato: il caso **E24** verifica meccanicamente che
nessun file di `aiagents/ailanguage/aibridge/aifold/aieval/executors/index`
importi qualcosa da `p6/`.

Tre vincoli resi meccanici:
- l'esecutore nasce con **zero capability verificate** (E19) e non vince un
  match `strict` (B5);
- un report esterno nasce **OBSERVED**, mai un verdetto; passare a
  `HUMAN_REVIEWED` richiede una firma umana (E17);
- **senza mapping verificato non esegue** (E18): `assertMappingUsabile` lancia.

### Onesta' sul contratto
**Non ho dati sufficienti** sullo schema I/O reale di MatrAIx: non l'ho
verificato in questa sessione. Per questo AL-6/AL-7 sono progettati **neutri**
(popolazione a segmenti con quote e seed, scenari con criteri di successo,
metriche dichiarate, findings con evidence_ref e riproducibilita') e la voce
`MAPPINGS['EXT-MATRAIX-01']` e' marcata `UNVERIFIED` con i campi a `"?"`.
Nessun campo del contratto e' presentato come "formato MatrAIx".

Per rendere il mapping VERIFIED serve da te: la specifica reale dei payload di
input/output (o due esempi reali, richiesta e risposta). A quel punto il
mapping si compila e il driver esegue.

Nel frattempo il contratto e' esercitato end-to-end da un driver locale
deterministico (stessa seed, stesso sigillo — caso E15), che fa anche da
**secondo esecutore** per `crossCheck`: il disaccordo viene riportato, non
mediato (E22), e lo stesso esecutore due volte non conta come controllo
indipendente (E23).

---

## 4. Esiti

| suite | esito |
|---|---|
| P6 Node | 24/24 |
| P6 Chromium | 6/6 |
| **totale progetto** | **205/205 GREEN** |

Evidenza (i conteggi qui sopra escono dal runner, non dalla mia penna):
- `aieval://P6-AiEval/15585a7b` -> `p6/evidence-p6-node.json`
- `aieval://P6-AiEval-browser/492b3030` -> `p6/evidence-p6-browser.json`

Il caso **B4** non e' auto-referenziale: una valutazione AL-6 pilota il DOM vero
di AiEvents 2.9.32 con gesti reali e misura lo stato dell'app — il segmento
"deciso" naviga, il segmento "esitante" (gesto da 20px) no.
