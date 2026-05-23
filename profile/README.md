<h1 align="center">Keel</h1>

<p align="center">
  <strong>Production-grade, vendor-neutral reliability for multi-model LLM apps.</strong><br/>
  Composable libraries — not a framework. All on PyPI.
</p>

<p align="center">
  <a href="https://pypi.org/project/keel-llm-reliability/"><img src="https://img.shields.io/pypi/v/keel-llm-reliability.svg?label=keel-llm-reliability&color=2b8a3e" alt="PyPI - keel-llm-reliability"></a>
  <a href="https://pypi.org/project/keel-llm-protocol/"><img src="https://img.shields.io/pypi/v/keel-llm-protocol.svg?label=keel-llm-protocol&color=2b8a3e" alt="PyPI - keel-llm-protocol"></a>
  <a href="https://pypi.org/project/keel-circuit-breaker/"><img src="https://img.shields.io/pypi/v/keel-circuit-breaker.svg?label=keel-circuit-breaker&color=2b8a3e" alt="PyPI - keel-circuit-breaker"></a>
  <br/>
  <a href="https://pypi.org/project/keel-llm-adapter-openai/"><img src="https://img.shields.io/pypi/v/keel-llm-adapter-openai.svg?label=adapter-openai&color=495057" alt="PyPI - keel-llm-adapter-openai"></a>
  <a href="https://pypi.org/project/keel-llm-adapter-anthropic/"><img src="https://img.shields.io/pypi/v/keel-llm-adapter-anthropic.svg?label=adapter-anthropic&color=495057" alt="PyPI - keel-llm-adapter-anthropic"></a>
  <a href="https://pypi.org/project/keel-llm-adapter-google/"><img src="https://img.shields.io/pypi/v/keel-llm-adapter-google.svg?label=adapter-google&color=495057" alt="PyPI - keel-llm-adapter-google"></a>
  <br/>
  <img src="https://img.shields.io/badge/python-3.11%2B-blue" alt="Python 3.11+">
  <img src="https://img.shields.io/badge/mypy-strict-blue" alt="mypy --strict">
  <img src="https://img.shields.io/badge/license-MIT-blue" alt="MIT">
</p>

---

## Why Keel

Building on multiple LLM providers means hand-writing the same fragile plumbing — retries, failover, rate-limit handling, circuit breaking — and usually getting it subtly wrong. *(The classic bug: treating a `429` as a failure and circuit-breaking a model that was merely throttled.)*

Keel is that plumbing, done right:

- **Vendor-neutral — no lock-in.** One interface across OpenAI, Groq, Anthropic, Gemini, Mistral, vLLM, local. Swap or mix providers without touching your code.
- **Right by default.** A rate-limited model *defers* instead of failing — measured to move a throttled model from **3/10 → 10/10 availability**.
- **Transparent, never a black box.** Every retry / failover / skip is visible data. Trust at scale because you can *see* what it did.
- **A library, not a framework.** You keep your control loop. No LangChain-style runtime to adopt.

## Quickstart — reliable multi-model calls in ~10 lines

```bash
pip install keel-llm-reliability keel-llm-adapter-openai
```

```python
import asyncio
from keel_llm_reliability import ResilientClient, Request
from keel_llm_adapter_openai import OpenAIAdapter
from keel_llm_protocol import user

# Any OpenAI-compatible endpoint works (OpenAI, Groq, OpenRouter, Mistral, vLLM, …).
groq  = OpenAIAdapter(model="llama-3.3-70b-versatile", api_key="gsk_…",
                      base_url="https://api.groq.com/openai/v1", provider="groq")
local = OpenAIAdapter(model="llama-3.1-8b", base_url="http://localhost:11434/v1",
                      provider="local")

async def main() -> None:
    result = await ResilientClient([groq, local]).failover(
        Request(messages=[user("One-line summary of TCP.")]))
    if result.succeeded:
        print(result.response.text)
    for a in result.attempts:                  # every decision is visible data
        print(a.model_key, a.outcome, f"{a.latency_ms}ms")

asyncio.run(main())
```

A `429` on `groq` is treated as backpressure (no circuit break), a `5xx` fails over to `local`, and you can *see* exactly what happened. Want an ensemble instead of failover? Call `client.fan_out(...)` — same client, returns every model that answered.

## Is this for you? *(honest segmentation)*

These libraries are sharp for a specific shape — not every product should adopt them. Read across:

<table>
<thead>
<tr><th align="left">Package</th><th align="left">Adopt when</th><th align="left">Skip when</th></tr>
</thead>
<tbody>
<tr>
  <td><a href="https://pypi.org/project/keel-llm-reliability/"><code>keel-llm-reliability</code></a></td>
  <td>Multi-model apps; production traffic where rate-limit handling matters; operators who'll <em>read</em> the visible-degradation trail.</td>
  <td>Single-provider apps; you already have working in-tree reliability; prototypes/scripts; you need a runtime/framework.</td>
</tr>
<tr>
  <td><a href="https://pypi.org/project/keel-llm-protocol/"><code>keel-llm-protocol</code></a></td>
  <td>Building tooling across multiple providers (router, evaluator, observability); writing a vendor-neutral adapter; want a shared error vocabulary.</td>
  <td>Single-provider apps (use the SDK); already invested in LangChain's <code>BaseChatModel</code> / LlamaIndex's <code>LLM</code>.</td>
</tr>
<tr>
  <td>Adapters <em>(<a href="https://pypi.org/project/keel-llm-adapter-openai/">openai</a> · <a href="https://pypi.org/project/keel-llm-adapter-anthropic/">anthropic</a> · <a href="https://pypi.org/project/keel-llm-adapter-google/">google</a>)</em></td>
  <td>Consuming via <code>keel-llm-protocol</code> / <code>keel-llm-reliability</code> — adapters map provider failures to the typed taxonomy.</td>
  <td>Direct provider calls in a simple app — use the official <code>openai</code> / <code>anthropic</code> / <code>google-generativeai</code> SDKs (wider surface).</td>
</tr>
<tr>
  <td><a href="https://pypi.org/project/keel-circuit-breaker/"><code>keel-circuit-breaker</code></a></td>
  <td>Any flaky external call (HTTP, DB, queue, LLM); want a small, zero-dep, keyed-by-string breaker.</td>
  <td>You already use <code>pybreaker</code> / <code>circuitbreaker</code> / <code>aiobreaker</code> and it works.</td>
</tr>
</tbody>
</table>

### The deciding test

> *"Does this deliver a capability my codebase genuinely lacks — or could I get the same outcome with the SDK / a library I already trust?"*

If the SDK + a basic retry handles your case, skip. Where these tip toward adoption is **multi-model**, **vendor-neutrality matters**, or **you need typed errors + visible degradation as a contract**.

## Quality bar

- **All on PyPI**, version-pinnable, OIDC trusted-publisher releases.
- **`mypy --strict` clean** across every package. PEP 561 `py.typed`.
- **Async-native.** Python 3.11+.
- **Zero-to-low dependencies** (substrate packages are zero-deps).
- **MIT licensed.**

---

<p align="center">
  <sub>Composable bricks for products that ship.</sub>
</p>
