<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Solar Open2 250B NVFP4 recipe

This recipe serves [`nota-ai/Solar-Open2-250B-Nota-NVFP4`](https://huggingface.co/nota-ai/Solar-Open2-250B-Nota-NVFP4)
with Dynamo and vLLM on B200 GPUs. It is an aggregated deployment with two
independent TP2 workers behind the Dynamo KV-aware router.

## Configuration

| Setting | Value |
| --- | --- |
| GPU | 4x B200 total; 2x B200 per worker |
| Topology | Aggregated, two worker replicas |
| Parallelism | TP2, DP1, expert parallel disabled |
| Weight precision | NVFP4 |
| KV-cache precision | FP8 |
| Routing | KV-aware, vLLM prefix-cache events |
| Context | 1,048,576 tokens |
| Reasoning parser | `solar_open2` |

Two workers are intentional: a single worker can publish KV events, but the
router needs at least two candidates before KV-aware placement does anything.

Solar Open2 is a hybrid architecture, 12 grouped-query attention layers and 36
linear-attention layers. vLLM accounts for the linear-attention state as a Mamba
page, so it raises the attention block size to 2128 tokens to keep the attention
page at least as large as the Mamba page. The recipe therefore does not set
`--block-size`; any value passed is overridden.

## Deployment

```bash
kubectl apply -f recipes/solar-open2-250b/model-cache/model-cache.yaml
kubectl apply -f recipes/solar-open2-250b/model-cache/model-download.yaml
kubectl wait --for=condition=complete job/model-download --timeout=2h

kubectl apply -f recipes/solar-open2-250b/vllm/agg-b200-chat/deploy.yaml
kubectl wait --for=condition=Ready \
  dynamographdeployment/solar-open2-250b-vllm-b200-agg-chat --timeout=30m
```

## Performance

Measured on a 15% subset of the `nim_turbo` 8k/1k 70kv chat trace: 1805 requests,
mean ISL 37.4k tokens, mean OSL 1008 tokens.

| Workload | Recipe | Framework | SKU | GPUs | Concurrency | System output tok/s/GPU | User output tok/s (P50) | TTFT P50 (ms) |
|---|---|---|---|---:|---:|---:|---:|---:|
| Chat (15% subset) | Aggregated, 2 replicas, KV-aware routing | vLLM | B200 | 4 | 20 | 214.98 | 52.16 | 319 |
| Chat (15% subset) | Disaggregated 1P:2D, round-robin | vLLM | B200 | 8 | 44 | 203.96 | 51.71 | 2395 |

Benchmark jobs: [`vllm/agg-b200-chat/perf.yaml`](vllm/agg-b200-chat/perf.yaml) and
[`vllm/disagg-b200-chat/perf.yaml`](vllm/disagg-b200-chat/perf.yaml).

### Aggregated

Two TP2 replicas behind a KV-aware router. Prefill and decode share the same workers,
so there is no KV transfer between them and TTFT is an order of magnitude lower than
the disaggregated profile. Single run.

### Disaggregated 1P:2D

One TP4 expert-parallel prefill worker and two TP2 decode workers, connected over
NIXL/RDMA, with round-robin routing. Routing is round-robin rather than KV-aware
because a single prefill worker gives a prefix-aware router nothing to choose
between; KV-aware and load-aware were both measured and neither helped.

The 1:2 prefill-to-decode ratio is not the rate-matched ratio, which would be 1:3.
It is chosen deliberately. With three decode workers the single prefill worker runs
at about 87% utilisation at the concurrency needed to saturate them, and at that
point queueing amplification makes p50 TTFT swing between 4.7 and 6.6 seconds across
runs against a 5 second target. With two decode workers the prefill worker runs at
about 65% utilisation, which keeps TTFT near 2.5 seconds with roughly 2.5 seconds of
margin. The 1P:2D shape trades a few percent of per-GPU throughput for a latency
figure that reproduces.

Figures are the mean of runs made with `perf.yaml` in this directory, which resets
the KV cache before profiling so each run starts from a cold prefix cache:

| Run | tok/s/GPU | User tok/s (P50) | TTFT (P50) |
|---|---:|---:|---:|
| 1 | 204.15 | 52.05 | 2.621 s |
| 2 | 203.77 | 51.45 | 2.322 s |
| 3 | 203.96 | 51.64 | 2.243 s |
| **mean** | **203.96** | **51.71** | **2.395 s** |

Throughput across these three runs spans 0.19%, so the figure is reproducible on a
fresh deployment.

Benchmarks that do not reset the cache between runs report several percent higher
throughput and materially lower TTFT, because the 70% KV-reuse trace leaves a warm
prefix cache behind. Use `perf.yaml` as written if you want figures comparable to
these.

