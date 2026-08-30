# AiWorld Runtime — distribuzione per i moduli

Due file, due modi di partecipare. Rigenerabili con `node dist/build.js`.

| File | Byte | A cosa serve |
|---|---|---|
| `aiworld-runtime.js` | 100.458 | il modulo **ospita** AiAgents: esegue task, verifica, persiste, tiene il gate |
| `aiworld-client.js` | 24.463 | il modulo **parla** con un nodo che ospita: solo AiBridge + client + guardrail |

## Modulo che ospita (come AiEvents 2.9.30)

```html
<script src="aiworld-runtime.js"></script>
<script>
  const world = AiWorld.boot({ nodeId: 'aievents', channelName: 'aiworld', persist: true });
  AiLanguageValidators.registerAll(world.eval);
  world.agents.registerExecutor({ id: 'EXT-...', capabilities_claimed: ['...'], run: async (task) => ({ ... }) });
  world.bridge.on('task.request', async (env) => {
    const res = await world.agents.run(env.payload, { meta: { source: env.source } });
    return AiBridge.envelope({ source: world.bridge.nodeId, target: env.source,
      type: res.success ? 'task.response' : 'task.error', run_id: res.run_id, payload: res });
  });
  AiBridgeClient.publishCapabilities(world);
</script>
```

## Modulo che parla soltanto (AiJob, AiStyle, una pagina qualunque)

```html
<script src="aiworld-client.js"></script>
<script>
  const bridge = AiBridge.createBridge({ nodeId: 'aijob', channelName: 'aiworld' });
  bridge.use(Constitutional.middleware);
  const client = AiBridgeClient.createClient({ bridge, target: 'aievents' });

  const res = await client.run({ intent: 'search_events', context: { prompt: '...' },
    constraints: { capabilities: ['search_events'] }, success_condition: ['C-14'] });
</script>
```

## Cosa vale per tutti, e cosa no

Sul **canale**: `BroadcastChannel` funziona **solo fra pagine dello stesso
origin**. Due moduli serviti da domini diversi non si vedono: per quel caso
serve l'adapter `remote` (Worker/A2A), che il bridge accetta ma che non è
ancora scritto.

Il **gate umano** e i guardrail P1–P6 girano su entrambi i lati: un modulo che
usa solo il client non può aggirarli, perché il middleware costituzionale
blocca già in partenza le azioni irreversibili senza `human_gate`.

`AiFold` vive **dove gira il runtime**: il modulo-client non ha uno stato
proprio, il run è persistito dal nodo che lo esegue. È voluto — due copie dello
stesso run che divergono sarebbero peggio di una sola.
