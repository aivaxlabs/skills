# Choose Image Model

Use when the agent must select an image generation model for a task.

## Objective

Pick the image generation model that best matches the task's style, fidelity, and budget.

## Preconditions

- The task is known: text-to-image, image-to-image, inpainting, style transfer, or a specific style (photorealism, illustration, anime, etc.).
- The plan and balance are known.
- The agent has access to `aivax_list_models` or the public model listing.

## Decision Tree

1. What is the output style?
   - Photorealistic: a model trained on real-world images. Most modern models can do this; the difference is fidelity on small details.
   - Illustration / stylized: a model known for the target style. The catalog usually names the style.
   - Text in image: a model with strong text rendering. Few models do this well; verify with `aivax_search_context`.
2. What is the aspect ratio or size requirement? Verify the model supports the target size.
3. What is the quality tier? `standard` is usually enough for prototypes. `high` is for production assets.
4. What is the cost budget? Image generation is priced per image, not per token. A high-quality image can cost 10x to 50x more than a standard image.
5. Does the task require image-to-image, inpainting, or reference images? Verify the model supports the operation.
6. Among the candidates that pass all floors, pick the cheapest. If two models tie on cost, pick the one with stronger text rendering or the better style match.

## Recommendation Output

- The chosen model name.
- A one-line rationale naming the binding constraint (cheapest with text rendering, photorealistic with the right aspect ratio, etc.).
- Cost per image at the chosen quality tier.
- The dominant alternative and when to switch to it.

## Validation

- The model is available on the user's plan.
- The model supports the intended operation (text-to-image, image-to-image, inpainting, etc.).
- A small smoke test produces an image that matches the prompt.
- The cost is within the user's cap.

## Escalation

- The catalog has no model that meets the floors: surface the binding constraint and ask the user to relax one.
- The model is deprecated or preview: load `references/cost-monitoring/situations/optimize-spend.md` for the rotation plan.
- A swap is needed in production: load `references/platform-rules/safe-mutations.md` and confirm approval.
