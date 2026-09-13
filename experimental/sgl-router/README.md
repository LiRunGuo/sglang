# sgl-router

Slim, KV-aware, OpenAI-compatible router for SGLang workers.

Serves a single model and routes across its workers. Exposes
`/v1/tokenize`, `/v1/detokenize`, `/v1/models`, `/v1/chat/completions`
(buffered and SSE), plus `/healthz` / `/readyz` and `/metrics`. Worker
pools come from either a static URL list or Kubernetes EndpointSlice
discovery.

## Building

```bash
cd experimental/sgl-router
cargo build --release
```

## Running

The router is configured entirely through CLI flags (run
`sgl-router --help` for the full list). It serves exactly one model, so
`--model-id` is required, along with exactly one discovery backend.
`--tokenizer-path` is optional: give it a local `tokenizer.json` path or a
HuggingFace repo id, and when omitted the router downloads the tokenizer
for `--model-id` from HuggingFace (honoring `HF_TOKEN` / `HF_HOME`).

Static worker list:

```bash
sgl-router \
  --host 0.0.0.0 --port 30000 \
  --model-id qwen3 \
  --tokenizer-path /models/qwen3/tokenizer.json \
  --worker-urls http://10.0.0.1:30000 http://10.0.0.2:30000
```

Kubernetes EndpointSlice discovery:

```bash
sgl-router \
  --host 0.0.0.0 --port 30000 \
  --model-id qwen3 \
  --tokenizer-path /models/qwen3/tokenizer.json \
  --service-discovery \
  --service-discovery-namespace prod \
  --selector app=engines-qwen3
```

Omit `--service-discovery-namespace` to watch all namespaces (requires
cluster-wide RBAC). For prefill/decode disaggregation, replace `--selector`
with `--prefill-selector` and `--decode-selector`.

External KV indexer as the cache-aware signal source:

```bash
sgl-router \
  --model-id qwen3 \
  --tokenizer-path /models/qwen3/tokenizer.json \
  --worker-urls http://10.0.0.1:30000 http://10.0.0.2:30000 \
  --policy cache_aware \
  --cache-prefix-provider indexer \
  --kv-indexer-endpoint http://10.0.0.10:50051 \
  --kv-indexer-query-timeout-ms 100 \
  --kv-indexer-query-max-inflight 32
```

The Indexer replaces the Router-local radix tree as the native Cache-Aware
signal. Query timeouts and local concurrency are bounded by the two Indexer
options, which default to 100 ms and 32 respectively.

## Chat rendering and compatibility

For text chat, the router renders and tokenizes with Dynamo and forwards the
result as `input_ids`, retaining the original messages and request options.
SGLang uses these IDs as the prompt and skips its own rendering and tokenization.
This applies to all routing policies, including requests with tools, reasoning
history, and `chat_template_kwargs`.

Compatibility with requests sent directly to SGLang is incomplete. In particular:

- Final assistant turns and `continue_final_message` can differ: the router
  currently always requests a generation prompt.
- Per-request `chat_template`, top-level reasoning controls (`reasoning` and
  `reasoning_effort`), and `task` are not fully connected to the Dynamo renderer.
- Tools and tool history use Dynamo's formatting; SGLang's tool selection,
  legacy `functions`, and message normalization may produce a different prompt.
- Worker template overrides, default template kwargs, reasoning defaults, and
  tokenizer configuration are not automatically synchronized with the router.

These options do not block ID forwarding. Retaining them in the request does
not make the engine apply them to the already-rendered prompt; an option may
therefore be ignored during rendering or differ from direct SGLang behavior.
Use matching model/tokenizer files on the router and workers. Completing the
request adapter and testing token parity against SGLang are follow-up work.

Caller-provided `input_ids` are preserved. Messages with non-text content parts
(images, audio, video, or unknown part types) use engine-side preprocessing;
injecting IDs would bypass media extraction. Text-only content arrays and
tool-call messages with null or omitted content remain eligible. If rendering
or tokenization fails, or no formatter is available, the router forwards the
original prompt without injecting IDs. Raw-text fallback tokens are used only
for routing.

## Upgrading from `cache_aware_zmq`

The `cache_aware_zmq` policy has been removed. Configurations using it should
select `--policy cache_aware` and choose a native cache-prefix source: the
Router-local radix tree (the default), or the external Indexer shown above.

The legacy `--cache-threshold`, `--balance-abs-threshold`, and
`--balance-rel-threshold` flags have also been removed. They do not have
one-to-one replacements; remove them and review the current `sgl-router
--help` output when tuning Cache-Aware routing.

## License

Apache-2.0.
