# Azure ImageGen Plugin

`azure-imagegen` is a GitHub Copilot CLI plugin that packages an Azure-first image generation skill plus a bundled Python CLI for Azure OpenAI v1 image workflows.

The installable unit is the repository root. GitHub Copilot CLI discovers the root [`plugin.json`](./plugin.json) manifest and loads the skill from [`skills/azure-imagegen`](./skills/azure-imagegen).

## What it includes

- GitHub Copilot CLI plugin manifest at [`plugin.json`](./plugin.json)
- Copilot skill at [`skills/azure-imagegen`](./skills/azure-imagegen)
- bundled CLI at [`skills/azure-imagegen/scripts/image_gen.py`](./skills/azure-imagegen/scripts/image_gen.py)
- repository documentation assets under [`assets/`](./assets)
- validation tests and CI smoke checks

## Scope

- Azure OpenAI v1 only
- Image API workflows only
- `generate`, `edit`, and `generate-batch`
- API key or Entra ID authentication

## Install As A Plugin

Install from the repository root while developing locally:

```powershell
copilot plugin install .
```

Install from GitHub:

```powershell
copilot plugin install openassistuk/azure-imagegen
```

List installed plugins:

```powershell
copilot plugin list
```

Update the plugin:

```powershell
copilot plugin update azure-imagegen
```

Uninstall the plugin:

```powershell
copilot plugin uninstall azure-imagegen
```

## Dependency Setup

Python 3.11 is the CI baseline.

Install runtime dependencies from the plugin root:

```bash
python -m pip install -e .
```

Add optional Entra authentication support:

```bash
python -m pip install -e ".[entra]"
```

Install development dependencies for validation and tests:

```bash
python -m pip install -e ".[dev,entra]"
```

If you use `uv`, the equivalent workflow is:

```bash
uv sync --extra dev --extra entra
```

The runtime dependency set is:

- `openai`
- `pillow`
- `python-dotenv`
- optional `azure-identity` for live Entra-authenticated runs
- optional ImageMagick `magick` CLI for local transparent-background post-processing

## Quick Start

From the plugin root:

```powershell
python .\skills\azure-imagegen\scripts\image_gen.py generate `
  --endpoint "https://example.openai.azure.com" `
  --deployment "gpt-image-prod" `
  --prompt "Minimal ceramic mug on a clean studio background" `
  --dry-run
```

That performs a zero-network configuration smoke test. For live calls, use your Azure endpoint and deployment or set:

```text
AZURE_OPENAI_ENDPOINT
AZURE_OPENAI_DEPLOYMENT
AZURE_OPENAI_API_KEY
```

For local development, copy the root `.env.sample` values into a root `.env` file. The CLI loads that file when it exists, while already-exported process environment variables remain authoritative.

For deeper CLI usage and prompt recipes, use the bundled skill references instead of this README:

- [CLI reference](./skills/azure-imagegen/references/cli.md)
- [Azure auth reference](./skills/azure-imagegen/references/azure-auth.md)
- [Prompting guidance](./skills/azure-imagegen/references/prompting.md)
- [Sample prompts](./skills/azure-imagegen/references/sample-prompts.md)

## GPT-image-2

This plugin supports Microsoft Foundry GPT-image-2 deployments through the same Azure OpenAI v1 Image API path. The CLI infers GPT-image-2 behavior when the deployment name contains `gpt-image-2`.

- Omit `--size` for GPT-image-2 to let Azure's routing layer select the generation configuration.
- Pass explicit sizes such as `3840x2160`, `2160x3840`, `1024x1024`, `1536x1024`, `1024x1536`, or another `WIDTHxHEIGHT` value with both dimensions aligned to multiples of 16.
- Explicit GPT-image-2 sizes must be at least 655,360 pixels. Requests over 8,294,400 pixels are allowed with a warning because Azure may resize the final output.
- The Microsoft announcement names legacy size tiers and token buckets, but this plugin does not expose guessed flags for them until Microsoft publishes official Image API parameter names.
- GPT-image-2 does not support native `background=transparent`. Generate on a flat key color such as `#00FF00` and run `postprocess-transparent` with ImageMagick, or use a GPT-image-1/1.5 deployment for native transparent PNG output.

Example GPT-image-2 cutout post-process:

```powershell
python .\skills\azure-imagegen\scripts\image_gen.py postprocess-transparent `
  --input ".\output\imagegen\product-keyed.png" `
  --out ".\output\imagegen\product-transparent.png" `
  --key-color "#00FF00" `
  --fuzz 6 `
  --trim
```

## Compatibility And Limitations

- Azure-only: no direct non-Azure OpenAI endpoint support
- v1-only: no classic `api-version` Azure endpoint mode
- Image API only: no Responses API runtime path in this version
- local Python environment required for the bundled CLI

See [limitations](./skills/azure-imagegen/references/limitations.md) for the explicit boundary list.

## Validation And Release

Local validation:

```bash
python -m pip install -e ".[dev,entra]"
pytest
```

If GitHub Copilot CLI is installed locally, verify plugin discovery from the repository root:

```powershell
copilot plugin install .
copilot plugin list
copilot plugin uninstall azure-imagegen
```

GitHub Actions runs packaging validation and dry-run smoke tests on pull requests, pushes to `main`, and version tags matching `v*`.

GitHub tags and release archives are the intended distribution format. Because the repository root is the plugin root, a checkout or release archive can be installed directly without an extra packaging step.
