---
title: "Ollama adapter drops max_tokens \u2014 model always generates to stop token"
description: "The ollamaRequest struct in the Ollama adapter has no Options field, so MaxTokens from ChatRequest is silently dropped and never converted to Ollama's num_predict parameter. Every request runs to the model's natural stop token regardless of the caller-specified max_tokens value.\\n\\nRepro Steps:\\nSend a chat completion request to the inference gateway with max_tokens=5. Observe the token count in the response.\\n\\nExpected:\\nThe Ollama request body includes options.num_predict=5. The response contains \u22645 completion tokens.\\n\\nActual:\\nOllama receives no num_predict field. The response returned 135 completion tokens despite max_tokens=5 being specified."
status: QuickDev
featureId: ""
slug: "ollama-adapter-drops-max-tokens-model-always-generates-to-st-7bed0ef2"
created_at: 2026-05-14T15:46:36Z
updated_at: 2026-05-14T15:46:36Z
quickdev_source: "lens-core-bugfix"
---

Bug report submitted via /lens-core-bugfix. Target: TargetProjects/terminus/inference/terminus-inference-gateway.