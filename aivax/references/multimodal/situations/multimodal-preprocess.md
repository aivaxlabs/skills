# Multimodal Preprocess

Use when the model does not support the input modality, or when the agent wants the main model to receive a text description of media instead of the raw media.

## Objective

Convert media into a textual description that the main model can reason over, with the right balance between fidelity, cost, and latency.

## Preconditions

- A media input (image, audio, video, file) is in the request.
- The main model is text-only or the agent wants the model to receive a description.
- The account balance meets the minimum for the chosen modality (image/audio $0.10, file/video $0.50).

## Decision Tree

1. What modalities need preprocessing? Use one of the flags: `Image`, `Audio`, `Video`, `File`, `OtherFiles`, or `All`. Multiple flags are not combinable; use a single flag that covers all required modalities, or send two requests.
2. Does the user want the main model to see the media directly? Do not preprocess. The preprocess is for text-first models.
3. Is the same media sent repeatedly? The resolver caches descriptions by content hash. Re-sending the same content is cheaper on the second call.
4. Is the description quality critical? Use a higher-quality describe model via `references/text-tools/situations/describe-media.md` for the critical parts, and the default preprocess for the rest.

## Construction

```text
{
  "model": "<text-only-model>",
  "messages": [
    { "role": "user", "content": [
      { "type": "text", "text": "<prompt>" },
      { "type": "image_url", "image_url": { "url": "<url>" } }
    ] }
  ],
  "multimodal_preprocess": "Image"
}
```

## Validation

- The main model received a textual description that includes the relevant facts.
- The preprocess tokens are accounted for in `usage`.
- The trace ID is preserved.
- The description is faithful to the media. Spot-check on a small sample.

## Failure Modes

- Preprocess fails (modality unsupported, balance too low, content unreachable): the inference fails. Inspect the error and the resource.
- Preprocess produces a poor description: switch to a more specific describe model or send the media directly to a multimodal model.
- Preprocess changes the description between calls: the cache key changed. The content is the same; the cache key is the content hash, not the URL. If the URL serves different content on each call, the cache miss is expected.

## Escalation

- The description is too long for the main model's context: load `references/text-tools/situations/segment-text.md` and segment the description.
- The preprocess model is not available: load `references/text-tools/situations/describe-media.md` and use a direct describe call instead.
- The cost of preprocess exceeds the main inference cost: switch the main model to a multimodal model that supports the input directly.
