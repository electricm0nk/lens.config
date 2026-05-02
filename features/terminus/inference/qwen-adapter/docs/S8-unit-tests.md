---
feature: terminus-inference-qwen-adapter
story_id: S8
title: Unit Tests
epic: E2
sprint: 2
points: 5
status: not-started
assignee: null
depends_on: [S1]
blocks: [S9]
updated_at: '2026-04-26T14:55:00Z'
---

# S8 — Unit Tests

## As a
Developer

## I want
pytest unit tests covering `warmup.py` with mocked HTTP requests and filesystem operations

## So that
The warmup logic can be validated in CI without a running ollama instance or GPU

## Context

The warmup script (S1) is the most critical piece of new logic. Unit tests ensure correctness of each function, proper exit code behavior, and edge-case handling — all without requiring a real GPU or ollama running.

## Acceptance Criteria

1. Tests are in `tests/test_warmup.py` (or equivalent test module location)
2. `test_wait_for_ollama_success`: mock `requests.get /api/tags` to return 200 on first call — asserts function returns `True`
3. `test_wait_for_ollama_timeout`: mock `requests.get` to always raise `ConnectionError` — asserts function raises `TimeoutError` (or returns `False`) after timeout
4. `test_pull_model_success`: mock `requests.post /api/pull` to return `{"status": "success"}` — asserts function returns without error
5. `test_pull_model_failure`: mock `requests.post /api/pull` to return 500 — asserts function raises an exception
6. `test_generation_success`: mock `requests.post /api/generate` to return a valid generate response — asserts function returns without error
7. `test_generation_timeout`: mock `requests.post /api/generate` to time out — asserts function raises `TimeoutError`
8. `test_gpu_memory_check_pass`: mock `subprocess.run nvidia-smi` to return 32768 MB free — asserts check passes
9. `test_gpu_memory_check_fail`: mock `subprocess.run nvidia-smi` to return 16384 MB free — asserts check fails fast with clear error
10. `test_main_success`: mock all sub-functions to succeed — asserts `/tmp/warmup.ready` is written and process exits 0
11. `test_main_warmup_failure`: mock `test_generation` to raise an exception — asserts process exits 1 and `/tmp/warmup.ready` is NOT written
12. All tests pass in CI (no GPU or ollama required — all external calls mocked)

## Technical Notes

```python
# Example test structure using pytest and unittest.mock
import pytest
from unittest.mock import patch, MagicMock
from pathlib import Path
import warmup  # or from app.warmup import ...

def test_wait_for_ollama_success():
    with patch("requests.get") as mock_get:
        mock_get.return_value.status_code = 200
        result = warmup.wait_for_ollama(timeout=5)
        assert result is True

def test_main_success(tmp_path, monkeypatch):
    monkeypatch.setenv("OLLAMA_MODEL", "qwen2.5:14b")
    ready_file = tmp_path / "warmup.ready"
    with patch.multiple(
        warmup,
        wait_for_ollama=MagicMock(return_value=True),
        check_gpu_memory=MagicMock(return_value=True),
        pull_model=MagicMock(return_value=True),
        test_generation=MagicMock(return_value=True),
        READY_MARKER=str(ready_file)
    ):
        warmup.main()
    assert ready_file.exists()
```

## Definition of Done

- [ ] All 12 test cases implemented
- [ ] All tests pass in CI without GPU or ollama
- [ ] `warmup.py` functions are structured for easy mocking (dependency injection or module-level mocking)
- [ ] Test coverage for warmup.py is ≥90%
- [ ] Code reviewed and merged to feature branch
