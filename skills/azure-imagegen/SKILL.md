---
name: azure-imagegen
description: "Generate, edit, and batch-create images with Azure OpenAI v1 Image API. Use when GitHub Copilot CLI needs Azure-first image generation or image editing workflows for deployed image models, including prompt-to-image, masked edits, background extraction, transparent backgrounds, product shots, UI mockups, or batch JSONL runs, with API key or Entra ID authentication via the bundled CLI (`scripts/image_gen.py`)."
---

# Azure ImageGen

Generate or edit images with Azure OpenAI v1 by using the bundled CLI `scripts/image_gen.py`. Use deployment names instead of raw model names, prefer dry-runs before live calls, and keep the skill Azure-specific.

## Runtime Scope

- Target Azure OpenAI v1 only.
- Use the Image API only in v1.
- Exclude classic Azure `api-version` endpoints.
- Exclude a Responses API runtime path in v1.
- Infer gpt-image-2 behavior when the Azure deployment name contains `gpt-image-2`.

## Workflow

1. Decide `generate`, `edit`, or `generate-batch`.
2. Collect prompt(s), exact text, constraints, and any input image(s) or mask(s).
3. Resolve the plugin root and bundled script path before invoking the CLI.
4. Install runtime Python dependencies before first live use.
5. Resolve Azure configuration in this order:
   - direct CLI value (`--endpoint`, `--deployment`)
   - custom env-var-name flag (`--endpoint-env`, `--deployment-env`, `--api-key-env`)
   - default env var (`AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_DEPLOYMENT`, `AZURE_OPENAI_API_KEY`)
6. Keep the terminal in the user's working directory; invoke the script by absolute path or stored path variable instead of `cd`-ing into the plugin root.
7. If the CWD `.env` exists and contains populated `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_DEPLOYMENT`, and `AZURE_OPENAI_API_KEY` values, treat configuration as known-good and skip the dry-run unless the user asks for one.
8. Otherwise run the bundled CLI with `--dry-run` first.
9. Inspect outputs, confirm the output path clearly, and offer to open or display the generated image.

## Installed Path And Setup

- When working from this repository, the plugin root is the repo root and the script is `skills/azure-imagegen/scripts/image_gen.py`.
- After `copilot plugin install <owner>/azure-imagegen`, resolve the installed plugin root under the Copilot plugin cache, commonly `~/.copilot/installed-plugins/_direct/<owner>--azure-imagegen/`; on Windows use `$HOME\.copilot\installed-plugins\_direct\<owner>--azure-imagegen\`.
- The installed script path is normally `<plugin-root>/skills/azure-imagegen/scripts/image_gen.py`.
- If the exact cache path is uncertain, search for `skills/azure-imagegen/scripts/image_gen.py` under `~/.copilot/installed-plugins/` and use the discovered absolute path.
- Before first use in a Python environment, install runtime dependencies from the plugin root with `python -m pip install -e <plugin-root>`.
- For live Entra-authenticated calls, install the optional extra with `python -m pip install -e "<plugin-root>[entra]"` or install `azure-identity` separately.

## Environment Files

- The bundled CLI loads `.env` from the user's current working directory only.
- Always run the CLI from the user's desired output/project directory so CWD `.env` and relative output paths behave as expected.
- Existing process environment variables take precedence over values loaded from the CWD `.env`.
- If both environment variables and CWD `.env` are missing, offer to create a CWD `.env` with placeholders for `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_DEPLOYMENT`, and `AZURE_OPENAI_API_KEY`.
- Never ask the user to paste secrets into chat; for API-key auth, use `AZURE_OPENAI_API_KEY` or a custom env var referenced by `--api-key-env`.

## Command Selection

- If the user provides one or more input images, or asks to retouch, inpaint, mask, localize text, replace a background, or "change only X", use `edit`.
- If the user needs many prompts or many assets in one run, use `generate-batch`.
- If the user has a gpt-image-2 result on a flat key-color background and needs a transparent PNG, use `postprocess-transparent`.
- Otherwise use `generate`.

## Authentication

- Default to `--auth-mode api-key`.
- Use `--auth-mode entra` when the environment is configured for `DefaultAzureCredential`.
- Never ask the user to paste secrets into chat.
- If `--auth-mode api-key` is used, read the key from an environment variable, not a CLI secret flag.
- If `--auth-mode entra` is used, require `azure-identity` for live calls.
- For local configuration, use a CWD `.env` based on `.env.sample`; do not commit real secrets.

## Defaults And Rules

- Resolve or ask for the Azure deployment name instead of inventing one.
- Prefer the bundled CLI over ad hoc wrapper scripts.
- Keep prompt augmentation short and structural; do not invent new creative requirements.
- For edits, restate invariants every iteration.
- Use `quality=high` for text-heavy or detail-critical outputs.
- Use `input_fidelity=high` for identity-preserving or layout-sensitive edits.
- For gpt-image-2 deployments, use `--size auto` unless the user asks for exact dimensions; when dimensions are not exact, describe requested aspect ratio, framing, and resolution intent in the prompt instead of inventing pixel values.
- gpt-image-2 explicit sizes must use `WIDTHxHEIGHT` with dimensions aligned to multiples of 16 and at least 655,360 total pixels.
- gpt-image-2 requests above 8,294,400 pixels are allowed, but Azure may resize the final image to fit.
- gpt-image-2 does not support native `background=transparent`; use GPT-image-1/1.5 for native transparency, or generate on a flat key color and run `postprocess-transparent`.
- For gpt-image-2 key-color cutouts, request a pure flat background such as `#00FF00` or `#FF00FF`, no shadows, no reflections, crisp edges, and no use of the key color in the subject.
- Do not invent `smimage`, `image`, `xlimage`, or token-bucket CLI flags until Microsoft publishes official Image API parameter names.

## Prompt Augmentation

Reformat user prompts into a short production-style spec. Include only lines that are relevant.

```text
Use case: <taxonomy slug>
Asset type: <where the asset will be used>
Primary request: <user prompt>
Scene/background: <environment>
Subject: <main subject>
Style/medium: <photo/illustration/3D/etc>
Composition/framing: <wide/close/top-down; placement>
Lighting/mood: <lighting + mood>
Color palette: <palette notes>
Materials/textures: <surface details>
Text (verbatim): "<exact text>"
Constraints: <must keep/must avoid>
Avoid: <negative constraints>
```

## Use-Case Taxonomy

Generate:
- `photorealistic-natural`
- `product-mockup`
- `ui-mockup`
- `infographic-diagram`
- `logo-brand`
- `illustration-story`
- `stylized-concept`
- `historical-scene`

Edit:
- `text-localization`
- `identity-preserve`
- `precise-object-edit`
- `lighting-weather`
- `background-extraction`
- `style-transfer`
- `compositing`
- `sketch-to-render`

## Output Conventions

- Use `tmp/imagegen/` for temporary JSONL files or scratch assets.
- Write final outputs under the user's CWD, normally `./output/imagegen/`, by passing `--out-dir` for a directory or `--out` for a specific filename.
- Never use `--output`; it is not a supported flag and can be confused with `--output-format` or `--output-compression`.
- Keep filenames stable and descriptive by setting `--out` or `--out-dir`.

## Runtime Expectations

- gpt-image-2 image generations commonly take a few minutes.
- Complex or busy generations can take several minutes, and extreme cases may take up to 10 minutes.
- For live API calls, use a long command wait/timeout window, typically 180-600 seconds.
- Do not assume a live call has failed just because it appears quiet for a few minutes; wait for the CLI result unless the process exits, reports an error, or clearly exceeds the expected window.

## Dependencies

- Runtime dependencies: `openai` and `pillow`.
- Dotenv support uses `python-dotenv` to load CWD `.env` files when present.
- Add `azure-identity` only for live `--auth-mode entra` runs.
- Add ImageMagick only when using local `postprocess-transparent`; it shells out to `magick` on PATH.
- Do not require `rembg`; mention it only as an optional semantic background-removal tool for hair, glass, soft edges, or non-flat backgrounds.
- If the repo root is available, prefer its `pyproject.toml` for dependency installation. Otherwise install equivalent packages in the active Python environment.

## Reference Map

- `references/cli.md`: command catalog and CLI recipes
- `references/azure-auth.md`: API key and Entra auth setup
- `references/prompting.md`: prompting principles and iteration tips
- `references/sample-prompts.md`: copy/paste prompt recipes
- `references/limitations.md`: explicit v1 boundaries and non-goals
