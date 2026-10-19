# Choose Image Model

Use when the agent must select an image generation model for a task.

## Objective

Pick an available image model that fits the prompt, reference-image requirement, and budget.

## Preconditions

- The task and desired visual style are known, including whether the user has reference images.
- The account has a positive balance.
- The agent can consult `GET /api/v1/information/image-models.json`.

## Decision Tree

1. Consult `GET /api/v1/information/image-models.json` for current model names, descriptions, `flags`, and `pricingDescription`.
2. Does the task require an existing image? Choose a model with the reference-image flag before sending `referenceImages`. References guide the output; do not assume a separate mask or inpainting control.
3. Compare the model descriptions with the requested style and text-rendering needs. Treat these as selection clues, then inspect the generated result.
4. Is the requested size or aspect ratio important? Describe the desired framing in `input` and verify the output; this endpoint has no size or aspect-ratio parameter.
5. Compare the catalog's pricing descriptions with the budget and the planned number of prompts and images per prompt. They are estimates, not a guaranteed charge or a quality-tier selector.
6. Select a candidate that meets the reference requirement and budget; if the first result misses the brief, refine the prompt or switch models.

## Recommendation Output

- The chosen model name.
- A one-line rationale naming the binding constraint (reference support, style, or estimated cost).
- The relevant catalog pricing description, without treating it as a guaranteed total.
- An alternative model and when to try it, if available.

## Validation

- The selected model appears in the current catalog and its flags allow reference images if requested.
- Review a generated image against the prompt and the requested framing.
- Check actual usage against the agreed budget rather than assuming the estimate is the final charge.

## Escalation

- No catalog model meets the reference-image or budget requirement: explain the constraint and ask the user which requirement to relax.
- A model carries the `Deprecated` flag: choose another catalog model when possible; it may be removed, and the media MCP no longer lists it.
