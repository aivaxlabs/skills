# Multimodal Preprocess

Use when the model does not support the input modality, or when the agent wants the main model to receive a text description of media instead of the raw media.

## Objective

Convert media into a textual description that the main model can reason over, with the right balance between fidelity, cost, and latency.

## Preconditions

- A media input (image, audio, video, file) is in the request.
- The main model is text-only or the agent wants the model to receive a description.
- The selected path meets the account requirements in [Limits](https://docs.aivax.net/docs/limits); check [Pricing](https://docs.aivax.net/docs/pricing) before processing at scale.

## Decision Tree

1. Which media types need preprocessing? Set their engines in `multimodal_resolver`; configure several types in the same object when needed. Null or omitted engines keep those types unchanged. See `references/multimodal/SKILL.md` for accepted engines.
2. Is the task text extraction? Prefer `imageEngine: "Ocr"` or `fileEngine: "Ocr"`; for spoken words use `audioEngine: "Stt"`. Use `InferenceLow` or `InferenceHigh` when the task needs a media description rather than only extracted text.
3. Does the user want the main model to see media directly? Leave those engines null; ensure the main model supports the remaining raw media. An empty request object disables inherited preprocessing.
4. Is the same media sent repeatedly? Cache reuse depends on the account, content-part hash, and selected engine. Do not promise a cache hit or fresh remote content.
5. Is description fidelity critical? Consider `InferenceHigh` for that media type, compare it with direct multimodal input, and validate the relevant facts.

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
  "multimodal_resolver": { "imageEngine": "InferenceLow" }
}
```

The request resolver replaces, rather than merges with, the gateway resolver. Preserve all intended engines when overriding it. It takes precedence over deprecated `multimodal_preprocess`; absent request overrides, the gateway's `parameters.multimodalResolverParameters` applies.

## Validation

- The main model received a textual description that includes the relevant facts.
- The preprocess tokens are accounted for in `usage`.
- The trace ID is preserved.
- The description is faithful to the media. Spot-check on a small sample.

## Failure Modes

- Preprocess fails (modality unsupported, balance too low, content unreachable): the inference fails. Inspect the error and the resource.
- Preprocess produces a poor description: switch to a more specific describe model or send the media directly to a multimodal model.
- Unexpected reuse or changed descriptions: compare the content part and selected engine. URL-backed cache keys include the supplied URL, not a freshness check of the downloaded bytes; a changed resource at the same URL does not guarantee a cache miss.

## Escalation

- The description is too long for the main model's context: load `references/text-tools/situations/segment-text.md` and segment the description.
- The preprocess model is not available: load `references/text-tools/situations/describe-media.md` and use a direct describe call instead.
- The cost of preprocess exceeds the main inference cost: switch the main model to a multimodal model that supports the input directly.
