# Local agentic coding on macOS (oMLX + Cursor + Kilo Code)

## Target audience

This guide is for teams using **[Cursor](https://cursor.com/)** on a **company license** that would like to leverage the power of their Mac's hardware to handle local **agentic coding** given rising cloud AI costs.

The same **Kilo Code + oMLX** pattern works in **VS Code** or **IntelliJ** if you prefer those editors (not covered here).

## Goal

- Run a suitable model from disk with **oMLX**.
- Use **Kilo Code** only as the IDE client to **oMLX** on **`127.0.0.1`**. **Strictly prohibited:** any **model** traffic that sends **repository code or prompts** to **hosted**, **gateway**, or other **cloud** inference—**only** oMLX on loopback may perform inference for this workflow.
- Keep proprietary source code and prompts on your Mac for those **oMLX-only** model calls.

## What oMLX provides

**oMLX** is a native macOS app that runs an **MLX** inference server on your machine, with weights loaded from disk.

- **OpenAI-compatible HTTP API** under **`/v1`**, including **`/v1/chat/completions`** and **`/v1/models`**. **Kilo Code** uses this surface through its **custom provider**; follow [Cursor + Kilo Code integration](#cursor--kilo-code-integration) to configure it.
- **Operator tools:** a web **admin** dashboard at **`/admin`**, plus **menu bar** controls for the server and app lifecycle.
- **Session-friendly storage:** **disk-backed** cache directories for long runs; optionally **paged SSD KV caching** and **continuous batching** to improve throughput on long contexts and agent-shaped traffic.

## References

- [oMLX website](https://omlx.ai/)
- [oMLX on GitHub (`jundot/omlx`)](https://github.com/jundot/omlx)
- [Cursor documentation](https://docs.cursor.com/)
- [Kilo Code — custom models (OpenAI-compatible / local)](https://kilo.ai/docs/code-with-ai/agents/custom-models)

## Requirements

- Apple Silicon Mac with **macOS 26 Tahoe or newer**.
- [oMLX](https://omlx.ai/)
- **[Cursor](https://cursor.com/)** (company license is assumed; same extension marketplace as VS Code).
- [Kilo Code](https://kilo.ai/) (install from the marketplace **inside Cursor**). The extension requires **sign-in** to a **Kilo account** and ships **Kilo Gateway**–backed features (**autocomplete**, **background** helpers, **hosted** model lists) that do **not** use your **oMLX** URL. **Before you open confidential repositories**, open **Kilo Code → Settings** and **disable** those features so **all** chat and agent model calls go **only** to **oMLX** on loopback—see [Kilo Code: local inference only](#kilo-code-local-inference-only) and [Security and compliance](#security-and-compliance).

## Install oMLX (DMG)

1. Open [GitHub Releases](https://github.com/jundot/omlx/releases) for **jundot/omlx**.
2. Download the official **macOS `.dmg`**, open it, and install it by dragging **oMLX** into **Applications**.
3. Launch **oMLX** from **Applications** and complete the **welcome flow**: Defaults are fine (base directory, model directory and port). All you need is to set an API key (required to access the admin panel).
4. Once running, for ease of use, open the settings from the menu bar and select Launch oMLX at login and Start server automatically on launch.

## Download models

> **Warning:** Treat every download as **supply-chain sensitive** ([Security and compliance](#security-and-compliance)).

With **oMLX** running and the server on the default port, open the [oMLX admin model downloader](http://127.0.0.1:8000/admin/dashboard?tab=models&modelsTab=downloader).

Models of interest:

- [mlx-community/Qwen3.6-35B-A3B-4bit](https://huggingface.co/mlx-community/Qwen3.6-35B-A3B-4bit)
- [mlx-community/Qwen3.6-27B-4bit](https://huggingface.co/mlx-community/Qwen3.6-27B-4bit)
- [mlx-community/gemma-4-31b-it-4bit](https://huggingface.co/mlx-community/gemma-4-31b-it-4bit)
- [mlx-community/gemma-4-26b-a4b-it-4bit](https://huggingface.co/mlx-community/gemma-4-26b-a4b-it-4bit)

At the time this guide was written, each repo above distributed weights as **`*.safetensors`** and carried an **Apache 2.0** license on Hugging Face. **Licenses and file layouts can change**—always open the repo’s **`LICENSE`** and file listing before you download or pin a revision for production.

Coding and analysis scores along with speed and memory usage of these models:

| Benchmark                            | Qwen3.6-27B | Qwen3.6-35B-A3B | Gemma 4 31B | Gemma 4 26B A4B |
| :----------------------------------- | :---------- | :-------------- | :---------- | :-------------- |
| **SWE-bench Verified**               | **77.2%**   | 73.4%           | 52.0%       | 17.4%           |
| **SWE-bench Pro**                    | **53.5%**   | 49.5%           | 35.7%       | 13.8%           |
| **Terminal-Bench 2.0**               | **59.3%**   | 51.5%           | 42.9%       | 34.2%           |
| **LiveCodeBench v6**                 | 83.9%       | **84.8%**       | 82.27%      | 77.1%           |
| **Artificial Analysis Coding Index** | 34.9        | **35.1**        | 39.0        | 22.4            |
| **PP tok/s** (prompt, 4k ctx) [oMLX] | 475.6       | **2,219**       | 377.3       | 1,971           |
| **TG tok/s** (decode, 4k ctx) [oMLX] | 17.7        | **96.6**        | 13.9        | 70.9            |
| **Peak memory** (4k ctx) [oMLX]      | 17.3 GB     | 20.0 GB         | 20.2 GB     | **14.9 GB**     |

Speed and memory usage are from oMLX's community reports on a MacBook Pro 16" M5 Pro 20c 48GB: [Qwen3.6-27B](https://omlx.ai/benchmarks/cqxjdhf5), [Qwen3.6-35B-A3B](https://omlx.ai/benchmarks/jnqkxv37), [Gemma 4 31B](https://omlx.ai/benchmarks/op5f2ew0), [Gemma 4 26B A4B](https://omlx.ai/benchmarks/1x384qd3).

[Compare / explore](https://omlx.ai/compare) more submissions on [omlx.ai/benchmarks](https://omlx.ai/benchmarks).

All these models are well suited for agentic coding: they support **chain of thought (CoT)**, **tool calling**, and long-context workflows typical of agents.

**Qwen3.6-35B-A3B** stands out on **capabilities versus speed**: it is an efficient **mixture-of-experts (MoE)** design, so each decode step touches only a subset of parameters while still scoring strongly on coding and terminal-style benchmarks. On the cited M5 Pro numbers it delivers the **highest** prompt and decode throughput and moderate **peak memory** (~20 GB at 4k context)—a good default when that footprint fits your machine.

**Qwen3.6-27B** is the **strongest** of the four and uses a bit **less** peak memory, but with **lower** token throughput. It's a dense model that loads all parameters all the time—pick it when Qwen3.6-35B-A3B is having difficulty with complex tasks.

The **Gemma 4** series is less capable but worth keeping when you want a second opinion.

## Inference cost strategy (local first, then Cursor)

A practical way to **cut metered cloud spend** while keeping **Cursor** fully in play under your **company license**:

1. **Default — local oMLX:** Run day-to-day agent and chat work through **Kilo Code** with **`mlx-community/Qwen3.6-35B-A3B-4bit`** on **oMLX** ([Download models](#download-models)). It is a strong **MoE** coding model with high throughput on Apple Silicon; most routine refactors, explanations, and multi-step edits never need a cloud token.

2. **When the local model struggles — Cursor Composer 2:** Switch the same task to **Cursor’s** native **Composer** flow and choose **Composer 2**. Cursor has published that **Composer 2** builds on **Moonshot AI’s Kimi K2.5** as a base, then applies extensive **Cursor-side training** on real coding sessions—so it is not “raw Kimi,” but a **Cursor-hosted** coding model. Teams often report **frontier-class** results on hard repos at **much lower cost** than top-tier Claude or GPT; on some benchmarks and tasks it can **match or beat Claude Opus**–class models, though any model wins task-by-task.

3. **Rare stubborn problems — top cloud tiers:** For the few jobs where neither local **Qwen3.6-35B-A3B** nor **Composer 2** is enough, use **Cursor’s** strongest available models—e.g. **Claude Opus 4.7** or **GPT-5.5**.

## Security and compliance

### Trusted sources and safe weight formats

- Download models **only** from **trusted publishers** and mirrors you can tie to them: official org accounts on Hugging Face (for example **`mlx-community`**, **`Qwen`**, **`Google`**, **`meta-llama`**), your company’s artifact registry, or signed internal bundles approved by security.
- **Never** execute or import unknown **`.pickle`**, **`.pkl`**, or **`joblib`** blobs from the internet. **Do not** put them in your model directory.
- Stick to **safer weight formats**—for oMLX, use **`*.safetensors`** weights installed through **oMLX’s model downloader** in **`/admin`**. **Do not** use pickle-based weight or checkpoint formats from untrusted sources.

### Model scanning and sandboxing

- Run your organization’s **mandatory checks** after download (EDR, internal malware pipeline, checksums against a known-good manifest).
- For **first contact** with a new vendor or repo, use a **dedicated sandbox** with no production credentials before promoting the same artifact hash to a managed workstation.

### Network exposure and logging

- **Cursor:** Use **Cursor** under your **company license** as your organization provides it. This tutorial **adds** **Kilo Code** with a **custom provider** to **oMLX** so agent work you run through **Kilo** can use **local** weights and **reduce** metered cloud model use for that path.
- **oMLX:** In **`/admin`** and **`~/.omlx/settings.json`**, adjust diagnostics, updates, or logging if your deployment requires it.
- **Kilo Code:** **Disable autocomplete**, **background** cloud model use, **Auto Free**, **hosted** model selections, and any path that sends **code or prompts** to **Kilo’s gateway** or third-party APIs for this workflow ([Kilo Code: local inference only](#kilo-code-local-inference-only)). Sign-in still uses Kilo’s servers for **identity**; model inference for agents must remain on **oMLX** only.
- **Inference bind:** Keep the oMLX HTTP server on **`127.0.0.1`** only. Do **not** bind to **`0.0.0.0`**, **`::`**, or a routable interface except in a **network-isolated lab**; otherwise prompts and code may be reachable on the **LAN**.

### Licensing

- For **commercial or corporate** use, read the model **`LICENSE`** on Hugging Face (and any **use** / **acceptable use** terms from the publisher) and have **legal or open-source review** confirm **on-device inference** and your use case are allowed.
- **Apache 2.0** and **MIT** are often the simplest to clear; other licenses (copyleft, research-only, non-commercial, custom addenda) may need explicit approval—do not assume “open weights” implies enterprise use.

### Sensitive data and chat history

- **Cursor**, **Kilo Code**, and the oMLX **admin chat** often **persist conversation history in plain text** (plus workspace metadata and logs on disk). Treat those stores like confidential data: encryption at rest where possible, access controls, and secure deletion on hardware rotation.
- **Never** paste **passwords, API keys, session tokens, or raw customer PII** into prompts—even local models and logs can retain them; redact or use synthetic fixtures when testing.

## Check the API

oMLX has a mandatory API key for the admin portal. Do **not** reuse that admin key in API clients such as Kilo Code. Instead, open **[Settings](http://127.0.0.1:8000/admin/dashboard?tab=settings)**, go to **AUTH & INFO**, and add an **Additional API Key**. These keys are API-only and cannot be used for admin login.

When API key enforcement is enabled, every request must send an Additional API Key using an OpenAI-style header: **`Authorization: Bearer <your-api-key>`**. In Kilo Code’s **custom provider** dialog, put the same secret in the **API key** field so requests include that bearer token.

**List models** (replace `YOUR_API_KEY` with an Additional API Key from oMLX):

```bash
curl -s http://127.0.0.1:8000/v1/models \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY"
```

Note the exact **`id`** for your loaded model from that JSON (aliases in **`/admin`** are reflected in **`/v1/models`**).

**Chat completion** test (replace `YOUR_MODEL_ID` and `YOUR_API_KEY`):

```bash
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -d '{
    "model": "YOUR_MODEL_ID",
    "messages": [{"role": "user", "content": "Reply with one sentence: what are you?"}],
    "temperature": 0.7
  }'
```

## Cursor + Kilo Code integration

Connect **Kilo Code** to the **same** oMLX server so **Kilo’s** chat and agent flows use **local** weights. See **[Kilo Code custom models](https://kilo.ai/docs/code-with-ai/agents/custom-models)** for the **custom provider** dialog and **`kilo.jsonc`**.

**Cursor and Kilo Code:** Keep using **Cursor** as your licensed IDE. **Add** **Kilo Code** with a **custom provider** to **oMLX** so tasks you run through **Kilo** use **local** inference and **lower** Cursor’s **metered** cloud model spend for that work. The **Kilo → loopback** path is separate from Cursor’s own model plumbing (see [Why localhost must go through Kilo](#why-localhost-must-go-through-kilo-not-cursors-override-or-agent)).

### Why localhost must go through Kilo (not Cursor’s override or Agent)

**Kilo Code** sends HTTP to **`127.0.0.1`** from the **local extension host** (Cursor is VS Code–compatible here). **Do not** point Cursor’s **OpenAI base URL override** or built-in **Agent** at oMLX on loopback: **native** Cursor model traffic can be **proxied** through **remote** infrastructure that refuses **private networks**, which often surfaces as **“Access to private networks is forbidden”** and never hits your Mac. If you see that message, see [Access to private networks is forbidden](#access-to-private-networks-is-forbidden-or-similar-provider-error).

### Preconditions

- **oMLX** is running and **`curl http://127.0.0.1:8000/v1/models`** succeeds ([Check the API](#check-the-api)).
- You have **`YOUR_MODEL_ID`** from that JSON.

### Kilo Code: local inference only

This guide uses **Kilo Code** as the **IDE client** to **oMLX**. Kilo also exposes **cloud** and **gateway** features that must be **off** for confidential work: they send **code or prompts** outside your **oMLX** loopback path ([Autocomplete](https://kilo.ai/docs/code-with-ai/features/autocomplete), [Using Kilo for Free](https://kilo.ai/docs/getting-started/using-kilo-for-free), [Using the Kilo Code Provider](https://kilo.ai/docs/ai-providers/kilocode)).

**After sign-in** ([Setup & Authentication](https://kilo.ai/docs/getting-started/setup-authentication)), open **Kilo Code → Settings** and:

- **Autocomplete:** **Disable** it (including **automatic** ghost-text completions and any **manual** inline completion shortcut). Defaults route fill-in-the-middle requests through **Kilo’s gateway**, not your **oMLX** URL. Use the **status bar** control to **snooze** if you need a quick stop while hunting the permanent toggle.
- **Background helpers:** **Disable** or clear any **session title**, **summarization**, or **“small model”** feature that calls a **hosted** model; those requests do **not** go to **oMLX**.
- **Model picker:** Use **only** your **custom provider** + **`YOUR_MODEL_ID`** for agent chat. **Auto Free**, **hosted** catalog models, and other **cloud** endpoints are **strictly prohibited** for repository inference in this workflow.
- **Credits and gateway:** Ignore built-in **gateway** model lists for this workflow; they exist for Kilo’s cloud offering and are the wrong surface for **oMLX-only** inference.

Sign-in still uses Kilo’s servers for **authentication** only; **model** traffic for agents must stay on **`127.0.0.1`**.

### Install the extension

1. Open **Cursor** (company install is assumed).
2. Open the **Extensions** view (**`Cmd + Shift + X`**), search for **Kilo Code**, and install the extension published for that name (same marketplace flow as VS Code).

### Steps (custom OpenAI-compatible provider)

1. In **Cursor**, open **Kilo Code** (sidebar or command palette entry per your layout).
2. Open **Settings** from Kilo’s UI (**gear** icon), then go to the **Providers** tab.
3. Scroll to the bottom of the provider list and click **Custom provider**.
4. Fill the dialog:
   - **Provider ID:** a short unique id (letters, numbers, hyphens, underscores), e.g. **`omlx`**.
   - **Display name:** e.g. **oMLX (localhost)**.
   - **Base URL:** **`http://127.0.0.1:8000/v1`** (trailing **`/v1`** must match oMLX’s OpenAI-compatible root).
   - **API key:** **`YOUR_API_KEY`** — an **Additional API Key** from oMLX ([Check the API](#check-the-api)), not the admin login key. Leave empty only when API-key enforcement is **off** in oMLX and you accept **unauthenticated** loopback access.
   - **Models:** pick **`YOUR_MODEL_ID`** from the auto-fetched list after the base URL resolves, or type the **model id** yourself so it matches **`/v1/models`** exactly.
5. Save the provider, then choose **`provider_id/YOUR_MODEL_ID`** (Kilo’s **`provider_id/model_id`** form) in the **model picker** for sessions that should stay on loopback.

**Agent tooling:** For tool use (edits, terminal, and similar), set **`tool_call: true`** for that model in **`kilo.jsonc`** when the UI does not expose it—see [custom models](https://kilo.ai/docs/code-with-ai/agents/custom-models).

**Context limits:** For local models, Kilo recommends setting **`limit.context`** and **`limit.output`** under the model entry so compaction and caps match your oMLX model; otherwise defaults may not match the real context window.

### If Kilo Code fails against oMLX

- Confirm **`~/.omlx/logs/server.log`** shows incoming requests when you send a message; if nothing appears, the base URL or bind address is wrong.
- **API surface:** If **`curl`** from [Check the API](#check-the-api) works but an agent step fails, confirm oMLX implements the routes Kilo needs (typically **`/v1/chat/completions`**). Upgrade oMLX or narrow the agent feature if a route is missing.
- **Timeouts:** Very large prompts or slow decode can exceed default client timeouts; increase the custom provider **timeout** in **`kilo.jsonc`** per [Kilo Code custom models](https://kilo.ai/docs/code-with-ai/agents/custom-models).

### Cursor and Kilo privacy

**Cursor** follows your **company license** and settings as deployed. **Kilo Code** is separate: in **Kilo Code → Settings**, **disable** **autocomplete** and other **gateway** features so they do not send **code or prompts** off your **oMLX** path ([Kilo Code: local inference only](#kilo-code-local-inference-only)). Pointing **Kilo** at **oMLX** does **not** change Kilo’s defaults for you.

## Privacy rule for proprietary code

For repositories where **all** model calls must stay on **oMLX**:

- **Base URL:** `http://127.0.0.1:8000/v1` in the **Kilo Code** custom provider (another **loopback** port is fine if oMLX is configured there—**never** a wide-area host).
- **Model id:** exactly what **`GET /v1/models`** returns for your oMLX instance.
- **Autocomplete:** **Off**—it does **not** use the **oMLX** URL ([Kilo Code: local inference only](#kilo-code-local-inference-only)).
- **Background cloud models:** **Off**—session titling and similar must not call **hosted** models ([Using Kilo for Free](https://kilo.ai/docs/getting-started/using-kilo-for-free)).

Do **not** select **hosted** cloud models or OAuth-backed **cloud** tiers in **Kilo** for those repos—keep the active model on your **`omlx`** (or similarly named) custom provider only. In **Cursor**, **do not** use **OpenAI base URL override** or the **built-in Agent** expecting them to drive **oMLX** on loopback; use **Kilo Code** for that integration ([Why localhost must go through Kilo](#why-localhost-must-go-through-kilo-not-cursors-override-or-agent)).

## Final check

In **Cursor**, with **Kilo Code** using your **custom provider** model **`YOUR_MODEL_ID`**, send a short message or run a small **Kilo** agent task. Confirm in oMLX **`/admin`** metrics or **`~/.omlx/logs/server.log`** that the request hit **localhost**.

## Troubleshooting

### oMLX is not reachable

- Confirm the **oMLX app** shows the server as running and the port is **8000** (or whatever you configured).
- Run:

  ```bash
  curl -s http://127.0.0.1:8000/v1/models -H "Content-Type: application/json"
  ```

- If **`curl` fails**, check **`/admin`** for bind address and port, then inspect **`~/.omlx/logs/server.log`**.

### `API key required` or `authentication_error`

- Use **`Authorization: Bearer …`** in **`curl`** with the same secret oMLX expects ([Check the API](#check-the-api)), and the same value in Kilo Code’s custom provider **API key** field ([Cursor + Kilo Code integration](#cursor--kilo-code-integration)).
- Or disable API-key enforcement in **`/admin`** only when you intentionally accept **unauthenticated** access to the loopback API.

### Kilo Code cannot reach oMLX

- Confirm the custom provider **base URL** is exactly **`http://127.0.0.1:8000/v1`** and the **model id** matches **`/v1/models`**.
- If the server requires a key, the Kilo **API key** must be an **Additional API Key**, not the admin portal password ([Check the API](#check-the-api)).
- Remove typos in **Provider ID** / model picker strings; Kilo uses **`provider_id/model_id`** format.
- If **`curl`** to **`127.0.0.1`** works but Kilo still cannot connect, check **host firewall** or tools like **Little Snitch** that can block the extension host—even for loopback.
- **Kilo Code** can still open **network** connections for **sign-in** and any **gateway** features you have left enabled—**disable** those in **Kilo Code → Settings** when you need a strict **oMLX-only** path ([Security and compliance](#security-and-compliance), [Privacy rule for proprietary code](#privacy-rule-for-proprietary-code)). **Cursor** uses the network per your **company** deployment and license.

### “Access to private networks is forbidden” (or similar provider error)

You are almost certainly using **Cursor’s native** chat or **Agent** (or **Override OpenAI Base URL** toward oMLX), not **Kilo Code**. Those flows can block or mishandle **localhost** because **native** traffic may be **proxied** remotely (see [Why localhost must go through Kilo](#why-localhost-must-go-through-kilo-not-cursors-override-or-agent)). Switch to **Kilo Code** with the **custom provider** from [Cursor + Kilo Code integration](#cursor--kilo-code-integration) and try again; confirm **`~/.omlx/logs/server.log`** receives the request.

### Wrong model or unknown model

- Re-run **`/v1/models`** and align the Kilo model entry with a listed **`id`**.
- In **`/admin`**, confirm the model is **loaded** and check **aliases**.

### Memory pressure

- Use a **lower-bit** quantized build that fits unified memory.
- Adjust memory or concurrency limits in oMLX **`/admin`**.
- Close heavy apps (browsers, Docker, extra IDE windows) during large-repo work.
