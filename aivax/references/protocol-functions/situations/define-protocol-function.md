# Define Protocol Function

Use when the agent must write the model contract for a protocol function: name, description, schema, and inline vs. source placement.

## Objective

Produce a function definition the model calls for the right triggers and skips otherwise, with arguments AIVAX can validate before invoking the callback.

## Preconditions

- The user action the function exposes is stable, narrow, and owned by your app.
- The gateway that will expose the function is known.
- The callback URL and authentication approach exist or are planned (never shown to the model).

## Decision Tree

1. Inline `protocolFunctions` or `protocolFunctionSources`? Inline for a stable gateway-owned list; sources (array of URL strings returning `{ "functions": [...] }`) for a remotely managed or shared catalog.
2. What is the name? A short English action in valid JavaScript identifier form (minimum 3 characters): `search_user`, not `search`, `pesquisa-usuario`, or `search user`.
3. What is the description? Action plus trigger plus limitation: what it does, when to call, when not to call, expected input, and result kind.
4. What is the schema? `contentFormat` as JSON Schema: required fields, types, formats. Functions without arguments use an empty object schema.
5. Does the name stay unique after inline, sourced, MCP, built-in, and raw tools combine? Rename on collision.

## Construction

Inline form on the gateway:

```json
{
  "name": "my-ai-gateway",
  "parameters": {
    "protocolFunctions": [
      {
        "name": "view_client",
        "description": "Use this tool to obtain details and orders of a client via its ID. Do not call it when the user only provides a name; ask for the client ID first.",
        "callbackUrl": "https://my-external-api.com/api/scp/users",
        "contentFormat": {
          "type": "object",
          "properties": { "user_id": { "type": "string", "format": "uuid" } },
          "required": ["user_id"]
        }
      }
    ]
  }
}
```

Sourced form: `protocolFunctionSources` holds URL strings; each endpoint answers `GET` with `{ "functions": [...] }` using the same object shape (`name`, `description`, `callbackUrl`, optional `headers`, `contentFormat`).

## Validation

- The description names the trigger, the expected input, and at least one negative case.
- The schema requires exactly the fields the callback needs, no more.
- The function name is a valid JS identifier and unique in the combined tool surface.
- A test conversation calls the function for trigger phrases and skips it otherwise.

## Failure Modes

- Vague description ("Search customers"): the model never calls it or calls it for the wrong inputs. Rewrite with trigger and limitation.
- Over-broad schema: the model sends unusable arguments. Tighten required fields and formats.
- Too many functions: reasoning token growth and wrong-tool choice. Cut to the smallest set and write per-tool trigger rules in the gateway instructions.

## Limits

- The model sees only name, description, and schema; callback URL and headers stay hidden.
- Sourced listings are cached 10 minutes per account and source URL.
- A non-success listing response fails inference preparation; keep the endpoint fast, stable, and small.
