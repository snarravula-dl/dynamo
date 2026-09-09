<!--
SPDX-FileCopyrightText: Copyright (c) 2026 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

# Solar Open2 250B NVFP4 Recipes

Dynamo + vLLM serving recipes for
[Solar-Open2-250B-Nota-NVFP4](https://huggingface.co/nota-ai/Solar-Open2-250B-Nota-NVFP4)
on B200, in aggregated and disaggregated configurations.

Full documentation — prerequisites, deployment, configuration notes and measured
performance — is published at
<https://docs.nvidia.com/dynamo/latest/recipes/model-recipes/solar-open2-250b>.

| Path | Contents |
|---|---|
| [`model-cache/`](model-cache) | PVC definition and checkpoint download Job |
| [`vllm/agg-b200-chat/`](vllm/agg-b200-chat) | Aggregated profile, 2 replicas, KV-aware routing |
| [`vllm/disagg-b200-chat/`](vllm/disagg-b200-chat) | Disaggregated profile, 1P:2D, round-robin routing |
| [`perf/`](perf) | AIPerf benchmark Jobs for both profiles |
