# Local agentic coding on macOS (oMLX + Cursor + Kilo Code)

## Target audience

This guide is for teams using **[Cursor](https://cursor.com/)** on a **company license** that would like to leverage the power of their Mac's hardware to handle local **agentic coding** given rising cloud AI costs.

The same **Kilo Code + oMLX** pattern works in **VS Code** or **IntelliJ** if you prefer those editors (not covered here).

## Goal

- Run a suitable model from disk with **oMLX**.
- Point **Kilo Code** to oMLX's local endpoint.
- Keep proprietary source code on your Mac.

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
- [Kilo Code](https://kilo.ai/) (install as the **Kilo Code** extension from the marketplace **inside Cursor**; full steps are in [Cursor + Kilo Code integration](#cursor--kilo-code-integration).)

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

## Security and compliance

### Trusted sources and safe weight formats

- Download models **only** from **trusted publishers** and mirrors you can tie to them: official org accounts on Hugging Face (for example **`mlx-community`**, **`Qwen`**, **`Google`**, **`meta-llama`**), your company’s artifact registry, or signed internal bundles approved by security.
- **Never** execute or import unknown **`.pickle`**, **`.pkl`**, or **`joblib`** blobs from the internet. **Do not** put them in your model directory.
- Prefer **`*.safetensors`** trees typical of Hugging Face MLX repos used with oMLX. If policy mandates another format, confirm support for your **exact** oMLX build before copying files.

### Model scanning and sandboxing

- Run your organization’s **mandatory checks** after download (EDR, internal malware pipeline, checksums against a known-good manifest).
- For **first contact** with a new vendor or repo, use a **dedicated sandbox** with no production credentials before promoting the same artifact hash to a managed workstation.

### Telemetry, logging, and network exposure

- **oMLX:** review **`/admin`** and **`~/.omlx/settings.json`** for diagnostics, updates, or logging you must disable under policy.
- **Cursor:** review **Privacy**, **Network**, telemetry, indexing, and any **cloud** features your security team requires you to configure. **Kilo Code** calling **`127.0.0.1`** for inference does **not** make the whole IDE air-gapped—Cursor’s **native** AI features are separate.
- **Kilo Code:** review the extension’s settings and [Kilo](https://kilo.ai/) documentation for optional online features, updates, and telemetry that must align with policy.
- **Inference bind:** keep the HTTP server on **`127.0.0.1`**. Do **not** bind to **`0.0.0.0`** or a routable interface without an **isolated lab** and explicit approval—otherwise prompts and code may be exposed on the **LAN**.

### Licensing

- For **commercial or corporate** use, read the model **`LICENSE`** on Hugging Face and have **legal or open-source review** confirm internal inference is allowed.

### Sensitive data and chat history

- **Cursor**, **Kilo Code**, and the oMLX **admin chat** can **persist** conversation text, workspace metadata, and logs on disk. Treat those like confidential data: encryption, access controls, secure deletion on hardware rotation.
- **Never** paste **passwords, API keys, session tokens, or raw customer PII** into prompts—even local models and logs can retain them.

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

Connect **Kilo Code** to the **same** oMLX server so **Kilo’s** chat and agent flows use **local** weights. **[Kilo Code custom models](https://kilo.ai/docs/code-with-ai/agents/custom-models)** is the authoritative reference for the **custom provider** dialog and optional **`kilo.jsonc`** fields—UI labels can shift between releases, so prefer the doc over screenshots.

**Two AI surfaces:** Cursor ships **native** chat, agent, and model features. For the workflow in this guide, use **only Kilo Code** with your **oMLX** custom provider when the repo or policy requires **local** inference. Accidentally using **Cursor’s** built-in agent or a **hosted** model can send context off-machine and will be billed accordingly.

### Why localhost must go through Kilo (not Cursor’s override or Agent)

**Kilo Code** sends HTTP to **`127.0.0.1`** from the **local extension host** (Cursor is VS Code–compatible here). **Do not** point Cursor’s **OpenAI base URL override** or built-in **Agent** at oMLX on loopback: **native** Cursor model traffic can be **proxied** through **remote** infrastructure that refuses **private networks**, which often surfaces as **“Access to private networks is forbidden”** and never hits your Mac. If you see that message, see [Access to private networks is forbidden](#access-to-private-networks-is-forbidden-or-similar-provider-error).

### Preconditions

- **oMLX** is running and **`curl http://127.0.0.1:8000/v1/models`** succeeds ([Check the API](#check-the-api)).
- You have **`YOUR_MODEL_ID`** from that JSON.

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
   - **API key:** **`YOUR_API_KEY`** — an **Additional API Key** from oMLX ([Check the API](#check-the-api)), not the admin login key. Leave empty only if oMLX key enforcement is disabled and policy allows it.
   - **Models:** pick **`YOUR_MODEL_ID`** from the auto-fetched list after the base URL resolves, or add it manually so the **model id** matches **`/v1/models`** exactly.
5. Save the provider, then choose **`provider_id/YOUR_MODEL_ID`** (Kilo’s **`provider_id/model_id`** form) in the **model picker** for sessions that should stay on loopback.

**Agent tooling:** For tool use (edits, terminal, and similar), enable **`tool_call: true`** for that model in **`kilo.jsonc`** if the UI does not set it—see [custom models](https://kilo.ai/docs/code-with-ai/agents/custom-models).

**Context limits:** For local models, Kilo recommends setting **`limit.context`** and **`limit.output`** under the model entry so compaction and caps match your oMLX model; otherwise defaults may not match the real context window.

### If Kilo Code fails against oMLX

- Confirm **`~/.omlx/logs/server.log`** shows incoming requests when you send a message; if nothing appears, the base URL or bind address is wrong.
- **API surface:** If **`curl`** from [Check the API](#check-the-api) works but an agent step fails, check whether your oMLX build implements every route Kilo calls (often centered on **`/v1/chat/completions`**). Upgrade oMLX or narrow the agent feature if a route is missing.
- **Timeouts:** Very large prompts or slow decode can exceed default client timeouts; increase provider **timeout** in **`kilo.jsonc`** if Kilo documents that option for your provider type.

### Cursor and Kilo privacy

Tune **Cursor** under **Settings** (including **Privacy** / **Network** as your org requires) and review **Kilo Code**’s own options and documentation so the **stack** matches policy. Local inference through Kilo does **not** automatically disable Cursor telemetry or other Cursor features.

## Privacy rule for proprietary code

When you intend to stay **fully local** for sensitive repositories:

- **Base URL:** `http://127.0.0.1:8000/v1` in the **Kilo Code** custom provider (change only after security review if you use another **loopback** port—do not point at a wide-area host).
- **Model id:** exactly what **`GET /v1/models`** returns for your oMLX instance.

Do **not** select **hosted** cloud models, OAuth-backed tiers, or any provider you do not control in Kilo while handling sensitive code—keep the active model on your **`omlx`** (or similarly named) custom provider only. In **Cursor**, also avoid **native** cloud models and **built-in Agent** for those same repos unless your security team has explicitly approved that path (see [Why localhost must go through Kilo](#why-localhost-must-go-through-kilo-not-cursors-override-or-agent) for why Cursor’s override and Agent are the wrong path for localhost oMLX).

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
- Or disable API-key enforcement in **`/admin`** if policy allows anonymous loopback access.

### Kilo Code cannot reach oMLX

- Confirm the custom provider **base URL** is exactly **`http://127.0.0.1:8000/v1`** and the **model id** matches **`/v1/models`**.
- If the server requires a key, the Kilo **API key** must be an **Additional API Key**, not the admin portal password ([Check the API](#check-the-api)).
- Remove typos in **Provider ID** / model picker strings; Kilo uses **`provider_id/model_id`** format.
- If **`curl`** to **`127.0.0.1`** works but Kilo still cannot connect, check **host firewall** or tools like **Little Snitch** that can block the extension host—even for loopback.
- Remember **Cursor** and **Kilo Code** may still use **other** network features per their own settings ([Security and compliance](#security-and-compliance), [Privacy rule for proprietary code](#privacy-rule-for-proprietary-code)).

### “Access to private networks is forbidden” (or similar provider error)

You are almost certainly using **Cursor’s native** chat or **Agent** (or **Override OpenAI Base URL** toward oMLX), not **Kilo Code**. Those flows can block or mishandle **localhost** because **native** traffic may be **proxied** remotely (see [Why localhost must go through Kilo](#why-localhost-must-go-through-kilo-not-cursors-override-or-agent)). Switch to **Kilo Code** with the **custom provider** from [Cursor + Kilo Code integration](#cursor--kilo-code-integration) and try again; confirm **`~/.omlx/logs/server.log`** receives the request.

### Wrong model or unknown model

- Re-run **`/v1/models`** and align the Kilo model entry with a listed **`id`**.
- In **`/admin`**, confirm the model is **loaded** and check **aliases**.

### Memory pressure

- Use a **lower-bit** quantized build that fits unified memory.
- Adjust memory or concurrency limits in **`/admin`** (or your build’s settings).
- Close heavy apps (browsers, Docker, extra IDE windows) during large-repo work.
