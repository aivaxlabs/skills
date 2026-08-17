# Generate Image

Use when the agent must build the request, call `POST /api/v1/generations/images`, and handle the response.

## Objective

Produce one or more images that match the user's prompt, with the right fidelity, cost, and downstream delivery.

## Preconditions

- The model is chosen (see `situations/choose-image-model.md`).
- The prompt is clear. A good prompt names the subject, the style, the composition, the lighting, and any constraints (no text, square, etc.).
- The account balance is sufficient.
- The agent has a place to store the generated images (S3, GCS, local disk, the user's CMS).

## Decision Tree

1. Is the task one image? Set `n: 1`.
2. Is the task a few options? Set `n` to a small number (2 to 4) and let the user pick.
3. Is the task a batch? Load `references/batch/situations/design-batch-workflow.md` and run the generation in batch.
4. Is the task an edit of an existing image? Pass the reference image and (if needed) a mask.
5. Is reproducibility important? Set `seed` to a stable value and record it.
6. Is the prompt negative (avoid X, no Y)? Use `negative_prompt` when supported.
7. Is the output style specific? Name the style in the prompt (e.g. "in the style of a 1990s anime", "shot on 35mm film").

## Construction

```text
{
  "model": "<chosen-image-model>",
  "prompt": "<textual description>",
  "size": "<size-or-aspect>",
  "quality": "standard",
  "n": 1,
  "metadata": { "trace_id": "tr_..." }
}
```

For image-to-image:

```text
{
  "model": "<chosen-image-model>",
  "prompt": "<what-to-change>",
  "reference_image": "<url or data:>",
  "metadata": { "trace_id": "tr_..." }
}
```

## Response Handling

1. The response is a JSON object with one or more image entries. Each entry has a URL or a base64 payload.
2. Download the image promptly. The hosted URL is short-lived.
3. Store the image in the user's storage of choice.
4. Pass the stable URL to the user or the downstream system.
5. Record the model, the prompt, the parameters, and the image URL in the trace.

## Validation

- At least one image was generated and matches the prompt.
- The image was downloaded and stored before the hosted URL expired.
- The trace ID is preserved.
- The cost is within the user's cap.

## Failure Modes

- The model returns no image: the prompt was rejected (moderation, format). Adjust the prompt or the model.
- The model returns a low-fidelity image: the prompt is too vague or the quality tier is too low.
- The hosted URL expires before download: re-request the image with the same seed. The model is deterministic when seeded.
- The cost is higher than expected: the user did not set a cap. Set one and re-run.

## Escalation

- The user wants many images: load `references/batch/`.
- The user wants a different style: load `situations/choose-image-model.md` and try a different model.
- The user wants to edit an image, not generate one: confirm the operation is supported by the chosen model and re-issue the request.
