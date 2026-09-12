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

The request accepts only:

- `input` (required): a non-empty prompt string or a non-empty array of prompt strings.
- `count` (optional): an integer from 1 to 4; defaults to 1 and applies to each prompt.
- `model` (required): an image model identifier from `GET /api/v1/information/image-models.json`.
- `referenceImages` (optional): up to four safe absolute HTTP(S) image URLs. Use this only when the selected model has the reference-image capability flag in the catalog.

```json
{
  "input": ["<prompt-1>", "<prompt-2>"],
  "count": 1,
  "model": "<image-model>",
  "referenceImages": ["https://example.com/reference.jpg"]
}
```

The response groups generated image URLs by input prompt:

```json
{
  "data": [
    {
      "input": "<prompt-1>",
      "images": ["https://example.com/generated-image.jpg"]
    }
  ]
}
```

Do not assume a base64 response or a URL expiration policy not stated by the endpoint.

## Models

Use `GET /api/v1/information/image-models.json` as the image model catalog. Check the selected model's flags before sending `referenceImages`, and expose only models returned by this public catalog.

## Cost Awareness

Image generation is priced per image, not per token. Set a cost cap with the user before increasing `count` or sending many prompts. For batch use, load `references/batch/situations/design-batch-workflow.md`.

## Storage And Delivery

Treat each returned image URL as endpoint output. Download or persist it only when the user's delivery or storage requirements call for that step; do not infer undocumented URL lifetime guarantees.

## Validation

- `data` contains one `{input, images}` group for each supplied prompt.
- Each `images` value is an array of generated image URLs.
- The generated images match their corresponding prompts.

## Limits

- Image generation has a per-account quota. High-volume generation can hit the quota. Load `references/cost-monitoring/` and `references/resilience/situations/rate-limit-429.md` for the rate-limit response.
- `referenceImages` is accepted only for models whose catalog flags declare that capability.
- Public keys (`pk-aiv-`) can call image generation. They cannot call integrated text models.

## Escalation

- The model is producing poor results: load `situations/choose-image-model.md` and try a different model.
- The cost is too high: reduce `count`, send fewer prompts, or switch to a cheaper model.
- The task is to produce many images: load `references/batch/`.
- The user wants the model to reason about an image, not generate one: load `references/multimodal/situations/image-input.md`.
