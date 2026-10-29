# MoE expert route trace capture

The experimental `--moe-stream` path can optionally capture the actual expert IDs selected by the model router. This is intended to generate a real, layer-aware trace for KILN's `kiln-bench prefetch-replay` predictor evaluation.

## Capture

Set `LLAMA_MOE_STREAM_TRACE` to the output JSON path before starting inference:

```sh
LLAMA_MOE_STREAM_TRACE=/tmp/moe-route-trace.json \
  ./build/bin/llama-cli -m /path/to/model.gguf \
  --moe-stream --moe-stream-cache 4s \
  -p "A short reproducible prompt" -n 128
```

The trace is written when the MoE stream manager is destroyed, so the application must unload/close the model or exit cleanly before the file is complete. Recording is disabled by default. The recorder is bounded to 1,000,000 events per model manager; if the cap is exceeded, additional events are dropped and the final log reports the dropped count. The cap is currently fixed in code.

The JSON file is an array of objects such as:

```json
[
  {"layer": 0, "expert_id": 17},
  {"layer": 0, "expert_id": 4},
  {"layer": 1, "expert_id": 17}
]
```

Each selected expert ID is recorded in the order it appears in the router output, including repeated IDs across tokens. Both the standard remap path and multi-pass prefill path are instrumented. This records routing choices from the streaming-enabled model execution path; it does not by itself prove cache benefit or prefetch speedup.

## Replay in KILN

Copy the JSON trace to the KILN host and run:

```sh
cargo run -p kiln-bench -- prefetch-replay \
  --trace-file /path/to/moe-route-trace.json \
  --block-bytes 1048576 --top-k 2 \
  --max-pending 8 --max-inflight-mib 16 \
  --latency-requests 1 --json
```

The predictor replay currently treats the observed trace as its candidate universe and uses a uniform size per candidate. Use identical prompt, model, tokenizer, sampling settings and generation length for comparisons. Keep the trace and its model hash with the result. The simulated latency is measured in trace events, not real milliseconds, and this replay does not yet run I/O prefetch or establish end-to-end tokens/second.

## Integration boundary

This trace instrumentation is on the experimental `moe-stream` branch. The KILN repository's currently pinned `vendor/llama.cpp` gitlink does **not** point at this branch, so the feature is not active in the default KILN checkout yet. The branch is substantially behind current `master`; porting/validating its streaming implementation on a current base is a separate integration gate.
