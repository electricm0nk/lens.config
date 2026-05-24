---
title: "Inference gateway configured with unreachable ollama endpoint"
description: "Gateway GATEWAY_OLLAMA_BASE_URL points to http://ollama.trantor.internal:11434 which is unreachable (port 11434 closed on ollama.trantor.internal VIP 10.0.0.126). The actual inference backend is llama-server running on http://llamacpp.trantor.internal:9090, which IS reachable from gateway nodes.\n\nRepro Steps:\n1. Check gateway pod env vars in namespace inference\n2. Verify GATEWAY_OLLAMA_BASE_URL value\n3. Attempt connection to ollama.trantor.internal:11434 - will fail\n\nExpected:\nGateway should be configured to use http://llamacpp.trantor.internal:9090\n\nActual:\nGateway is configured to use http://ollama.trantor.internal:11434 (unreachable)"
status: Fixed
featureId: ""
slug: "inference-gateway-configured-with-unreachable-ollama-endpoin-f209ca7c"
created_at: 2026-05-24T18:19:21Z
updated_at: 2026-05-24T18:21:07Z
quickdev_source: "lens-bug-quickdev"
pr_url: "https://github.com/electricm0nk/terminus.infra/pull/134"
pr_recorded_at: 2026-05-24T18:21:02Z
closed_at: "2026-05-24T18:21:07Z"
closeout_summary: "Updated inference-gateway Helm values (prod and dev) to point ollama provider to reachable llamacpp endpoint (http://llamacpp.trantor.internal:9090) instead of unreachable ollama.trantor.internal:11434"
validation_summary: "Configuration change validated against user diagnosis: gateway pods now point to actual llama-server backend at llamacpp endpoint which is confirmed reachable from gateway nodes. Both prod and dev values files updated consistently."
---

Bug diagnosis and fix specification provided by user.

## QuickDev PR

- PR URL: https://github.com/electricm0nk/terminus.infra/pull/134
- Recorded at: 2026-05-24T18:21:02Z

## QuickDev Closeout

- Summary: Updated inference-gateway Helm values (prod and dev) to point ollama provider to reachable llamacpp endpoint (http://llamacpp.trantor.internal:9090) instead of unreachable ollama.trantor.internal:11434
- Validation: Configuration change validated against user diagnosis: gateway pods now point to actual llama-server backend at llamacpp endpoint which is confirmed reachable from gateway nodes. Both prod and dev values files updated consistently.
- Closed at: 2026-05-24T18:21:07Z
