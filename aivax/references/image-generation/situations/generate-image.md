# Generate Image

Use when the agent must build the request, call `POST /api/v1/generations/images`, and handle the response.

## Objective

Generate images from one or more prompts and handle their URLs according to the user's delivery needs and budget.

## Preconditions

- The model is chosen from `GET /api/v1/information/image-models.json` (see `situations/choose-image-model.md`).
- Each prompt names the subject, style, composition, and important constraints. Set framing with `size` when it matters; the model may still adjust the dimensions.
- The account has a positive balance, and the intended number of prompts and images fits the agreed budget.
- If references are needed, the model's catalog flags allow them and each reference is a safe absolute HTTP(S) URL or an image data URI.

## Decision Tree

1. For one image, send one `input` prompt with `count: 1` (or omit `count`).
2. For alternatives, increase `count` within the current [Limits](https://docs.aivax.net/docs/limits) and agreed budget; an array of prompts produces a separate group for each input.
3. For many independent requests, load `references/batch/situations/design-batch-workflow.md`.
4. For a reference-guided change, use `referenceImages` within the current request limit, with a model that supports them. Each entry is an HTTP(S) URL or a `data:image/png`, `data:image/jpeg`, `data:image/webp`, or `data:image/gif` data URI. The endpoint has no mask or separate editing operation.
5. Put style and exclusions in the prompt. Send optional `size`, `format` (`png`, `jpeg`, or `webp`), and `quality` (1–100, used for `jpeg`/`webp` encoding and ignored for `png`). This endpoint does not accept `seed` or `negative_prompt`; do not promise reproducible reruns.

## Construction

Send `POST /api/v1/generations/images` with a JSON body:

```json
{
  "input": "<textual description>",
  "count": 1,
  "model": "<chosen-image-model>",
  "size": "1024x1024",
  "format": "webp",
  "quality": 80
}
```

Omit `size`, `format`, and `quality` to keep the model default (1080x1920, the model's own format). `count` must be an integer from 1 to 4.

For reference-guided generation, choose a reference-capable model:

```json
{
  "input": "<what-to-change>",
  "count": 1,
  "model": "<reference-capable-model>",
  "referenceImages": ["https://example.com/reference.jpg"]
}
```

`input` can also be a non-empty array of non-empty prompt strings. Do not invent fields beyond `input`, `count`, `model`, `referenceImages`, `size`, `format`, and `quality`.

## Response Handling

1. Read `data` from the JSON response. It contains one `{ "input", "images", "outputs", "cost" }` group per prompt; image entries are URLs, not base64 payloads.
2. Check each group's `outputs`: one entry per requested image with `url`, `cost` and `error`. A failed image has a `null` URL, zero cost and an `error` message; do not assume every successful response has `count` images.
3. Deliver the returned URLs, or download and store the images if the user's workflow requires durable storage. The endpoint does not specify a URL expiration time.
4. Record the model, prompts, `count`, reference URLs when used, and returned URLs in the caller's own trace if needed; the endpoint does not accept `metadata` or a trace ID in its body.

## Validation

- `data` has one group per input prompt, and each generated image is represented by a URL in `images`.
- Inspect the output against the prompt before delivery or publication; investigate empty or short image arrays.
- If storage was requested, confirm that the images were saved and the delivered locations work.
- Check actual usage against the agreed budget.

## Failure Modes

- Invalid prompts, `count`, model, `size`, `format`, `quality`, or reference images return a validation error: fix the reported field before retrying.
- An empty or short `images` array: read `outputs[].error` for each failed image before deciding whether to retry.
- The output misses the brief: refine the prompt, change `size`, or try another catalog model. `quality` only sets the `jpeg`/`webp` encoding, not the generation quality.
- A returned URL is unavailable: report the failed retrieval rather than claiming a seeded rerun will reproduce the image.
- Usage exceeds the estimate: stop further generation and recheck the selected model's catalog pricing and request volume.

## Escalation

- The user wants many images: load `references/batch/`.
- The user wants a different style: load `situations/choose-image-model.md` and try a different model.
- The user needs a controlled edit or inpainting mask: explain that this endpoint supports reference-guided generation but not a mask or edit-specific controls.
