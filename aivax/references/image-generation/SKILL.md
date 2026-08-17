---
name: aivax-image-generation
description: Generate, edit, or transform images with AIVAX — model selection, prompt design, parameters, output storage, and result handling. Load when the user wants the model to create images, edit existing images, or produce visual assets.
---

# Image Generation

This sub-skill owns text-to-image generation and image-editing through AIVAX. The endpoint is `POST /api/v1/generations/images`. The discipline is matching the model to the task, designing the prompt, and handling the result.

## Operating Files

- `situations/choose-image-model.md`: pick the right image generation model for the task.
- `situations/generate-image.md`: build the request, call the endpoint, handle the response.

## When To Use Image Generation

Use this sub-skill when the user wants the model to:

- Create an image from a text prompt.
- Edit or transform an existing image (variation, inpainting, style transfer).
- Produce visual assets for a product, a presentation, a social post, or a document.

Do not use this sub-skill for:

- Reasoning about an image (load `references/multimodal/situations/image-input.md`).
- Transcribing text inside an image (use the same multimodal path).
- Generating PDF or HTML pages (load `references/text-tools/` or the gateway built-in tools).

## Endpoint

`POST /api/v1/generations/images`.

The request accepts:

- `model`: an image generation model from `aivax_list_models`. Verify the model supports the intended task (text-to-image, image-to-image, inpainting, etc.).
- `prompt`: a textual description of the desired image.
- `negative_prompt` (when supported): a description of what to avoid.
- `size` or `aspect_ratio`: the output dimensions or aspect ratio. Verify the model's supported sizes.
- `quality`: the quality tier (e.g. `standard`, `high`). Higher quality costs more.
- `n`: the number of images to generate. Each image costs.
- `seed`: a deterministic seed for reproducibility.
- `reference_image` (when supported): a base image for editing or style transfer.
- `mask` (when supported): a mask for inpainting.
- `metadata`: a JSON object of string key/value pairs for trace correlation.

The response is a JSON object with one or more images. Each image has a URL, a base64 representation, or a hosted location depending on the model. The hosted location is a short-lived URL — download the image promptly.

## Models

Models vary widely in style, fidelity, latency, and cost. Verify with `aivax_list_models`. Common axes:

- Photorealism vs. illustration.
- Text rendering inside images (some models are better).
- Aspect ratio support.
- Cost per image at each quality tier.
- Whether the model supports image-to-image, inpainting, or reference images.

## Cost Awareness

Image generation is priced per image, not per token. A single high-quality image can cost more than a long text inference. Set a cost cap with the user before generating many images in batch. For batch use, load `references/batch/situations/design-batch-workflow.md`.

## Storage And Delivery

AIVAX does not store generated images indefinitely. The response includes a hosted location that is short-lived. The agent must:

1. Download the image promptly.
2. Store it in the user's storage of choice (S3, GCS, local disk).
3. Pass the stable URL to the user or the downstream system.
4. Never paste the image bytes or a long data URL in the final response.

## Validation

- The response includes at least one image.
- The image matches the prompt at the expected fidelity tier.
- The hosted location was downloaded and stored before the URL expired.
- The trace ID is preserved.

## Limits

- Image generation has a per-account quota. High-volume generation can hit the quota. Load `references/cost-monitoring/` and `references/resilience/situations/rate-limit-429.md` for the rate-limit response.
- Some models support only a subset of the parameters. Verify with `aivax_search_context` before assuming a parameter is supported.
- Public keys (`pk-aiv-`) can call image generation. They cannot call integrated text models.

## Escalation

- The model is producing poor results: load `situations/choose-image-model.md` and try a different model.
- The cost is too high: lower the quality, reduce `n`, or switch to a cheaper model.
- The task is to produce many images: load `references/batch/`.
- The user wants the model to reason about an image, not generate one: load `references/multimodal/situations/image-input.md`.
