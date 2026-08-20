# SERVICE36_STRICT_QOS

## Architecture

This deployment uses two physically separate KV slots:

- slot 0: Service36 Bot realtime inference
- slot 1: OpenCode, OpenClaw, and manual background agents

Start `llama-server` with:

```text
--qos-strict --qos-realtime-slot 0
```

Without `--qos-strict`, the server uses the original continuous batching behavior. When strict QoS is enabled and slot 0 is processing a request, background slots do not contribute tokens to the next compute batch. Their request, sampler, streaming state, and KV cache remain intact. Background prompt processing uses a 512-token scheduler quantum while the realtime slot is idle.

Priority takes effect between scheduler iterations. It does not interrupt a `llama_decode()` call that is already running.

## Protected host prompt cache

`--cache-ram` limits the entire serialized prompt cache in host RAM. It does not reserve VRAM and does not change either slot's KV allocation.

`--cache-realtime-ram` is an optional protected sub-limit inside that same host-RAM cache. It is an admission guard, not an additional allocation or reservation:

```text
--cache-ram 4096 --cache-realtime-ram 2048
```

The example permits at most 2 GiB of realtime cache entries within the 4 GiB global cache, leaving capacity for normal background entries.

Prompt cache entries record their owner slot and priority. Entries saved by `--qos-realtime-slot` are `REALTIME`; all other entries are `NORMAL`.

- eviction removes the oldest `NORMAL` entry first;
- a `NORMAL` request cannot evict a `REALTIME` entry;
- any slot may restore a matching `REALTIME` entry, but that restore retains the protected entry in host cache;
- if only protected entries remain, new cache admissions are rejected while the LLM request continues normally;
- a realtime admission is also rejected when it exceeds `--cache-realtime-ram`.

Restoring a host-cache entry still transfers serialized state back into VRAM, but avoids GPU prompt prefill for the matched prefix.

## Client slot ownership

Clients must always send an explicit `id_slot`. Do not use `id_slot: -1`; automatic slot selection can send a request to either slot.

### OpenCode

OpenCode must send `id_slot: 1` for the local model. For OpenCode 1.18.14, set it in the existing model-level options so the OpenAI-compatible request body contains the slot ID:

```json
{
  "models": {
    "qwen36-cmp": {
      "options": {
        "id_slot": 1
      }
    }
  }
}
```

The resulting request body must contain:

```json
{
  "id_slot": 1
}
```

Verify routing in the server journal:

```text
selected slot by id (1)
```

Use the same request-body setting for OpenClaw and manual background agents.

### Service36 Bot

All Service36 Bot requests must send `id_slot: 0`:

```json
{
  "model": "qwen36-cmp",
  "id_slot": 0,
  "messages": []
}
```

Set the slot ID in the central OpenAI-compatible transport layer:

```python
extra_body={
    "id_slot": 0,
}
```

Do not configure the slot independently in Planner, RAG, or Final Writer. Central transport ownership prevents one Service36 request path from falling back to automatic slot selection.

## Metrics

With `--metrics`, `/metrics` exports:

- `llamacpp:qos_preemptions_total`
- `llamacpp:qos_background_pauses_total`
- `llamacpp:qos_background_paused_seconds`
- `llamacpp:qos_realtime_slot`
- `llamacpp:qos_enabled`

QoS logs are emitted only when strict mode enters or exits:

```text
QOS STRICT enter: slot 0 active, background compute paused
QOS STRICT exit: slot 0 idle, background resumed
```
