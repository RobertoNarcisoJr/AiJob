# Come verificare P6 senza fidarsi di me

Tutto quello che segue e' riproducibile. Se un numero non torna, il numero
sbagliato e' il mio, non il tuo.

## Comandi

```
cd aiworld-core
node p6/test-p6.js            # suite Node
node p6/test-p6-browser.js    # suite Chromium (richiede playwright)
```

## Esiti attesi (sigilli deterministici)

| suite | casi | sigillo | evidenza |
|---|---|---|---|
| P6-AiEval (node) | 24/24 | `15585a7b` | `p6/evidence-p6-node.json` |
| P6-AiEval-browser | 6/6 | `492b3030` | `p6/evidence-p6-browser.json` |

Il sigillo e' FNV-1a su `{suite, counters, cases[{id,status}]}` con chiavi
ordinate. **Non dipende dai tempi**: due esecuzioni sane danno lo stesso valore.
Se il tuo sigillo differisce, un esito e' cambiato — e il gate te lo dice quale.

## Verificare l'evidenza senza rieseguire

```js
const Gate = require('./p6/aieval-gate.js');
const rep  = require('./p6/evidence-p6-node.json');
console.log(Gate.verify(rep));      // {valid:true, errors:[], recomputed:{...}}
```

`verify()` ricalcola i conteggi dai casi, ricalcola il sigillo e riderivà il
verdetto. Se avessi gonfiato i numeri, questo comando lo direbbe.

## Prova che il gate non e' decorativo

```js
const rep = JSON.parse(JSON.stringify(require('./p6/evidence-p6-node.json')));
rep.counters.passed = 99; rep.counters.total = 99;
Gate.verify(rep);   // valid:false — "AiEval-01: nessun totale scritto a mano"
```

## Totale progetto

`205/205` non e' un numero che ho scritto: e' la somma delle voci in
`MODULE_REGISTRY.json > suites`, ognuna prodotta da una suite eseguibile.

| suite | comando |
|---|---|
| base 38 | `node test/run-tests.js` |
| p1 17 | `node p1/test-p1.js` |
| p2 12 | `node p2/test-p2.js` |
| idempotenza 7 | `node p2/test-idem.js` |
| p3 14 | `node p3/test-p3.js` |
| p4 22 | `node p4/test-p4.js` |
| p5 18 | `node p5/test-p5.js` |
| browser 12 | `node p2r/test-p2r.js` |
| integrazione 9 | `node p1/real/aievents/test-integrazione.js AiEvents-2.9.32-aiworld.html` |
| swipe 9 | `node p1/real/aievents/test-swipe.js AiEvents-2.9.32-aiworld.html` |
| runtime 10 | `node p1/real/aievents/test-runtime.js AiEvents-2.9.32-aiworld.html` |
| multimodulo 7 | `node dist/test-multimodulo.js` |
| p6 node 24 | `node p6/test-p6.js` |
| p6 browser 6 | `node p6/test-p6-browser.js` |

## Integrita' del pacchetto

```
sha256sum -c MANIFEST.sha256
```
