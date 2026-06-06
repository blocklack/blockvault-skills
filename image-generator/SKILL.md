---
name: image-generator
description: Generate AI images from text prompts using the BlockVault delegate API (Imagen 4).
metadata:
  tool: generate_image
  category: media
  secrets:
    - key: DELEGATE_JWT
      description: BlockVault delegate API session token, issued via SIWE wallet sign-in.
      siwe:
        provider: delegate
        blockchain: ethereum
---

# Image Generator

Generate one or more images from a text description and return them inline as Markdown so the chat surface can render them directly.

## Instructions

### When to use

Trigger this skill when the user:

- Asks for an image, illustration, picture, drawing, mockup or visual.
- Wants a variation on a prior image (use a new prompt that references it).

### Authentication

The first call may prompt the user to sign in with their wallet to authorize the delegate API. This is automatic — no action required on your side. If sign-in is cancelled, surface the error to the user and stop.

### Call

Invoke `run_js`:

- **function**: `generate_image`
- **data**: `{"prompt": "<vivid English description>", "aspect_ratio": "<1:1|3:4|4:3|9:16|16:9>", "number_of_images": <1-4>, "negative_prompt": "<optional>", "enhance_prompt": <true|false>, "seed": <optional int>}`

Only `prompt` is required. Reasonable defaults:

- `number_of_images: 1`
- `aspect_ratio: "1:1"`
- `enhance_prompt: true`

### Response

The tool returns a JSON object with a `markdown` field already containing `![alt](<url>)` references to images that have been saved to the device — paste it verbatim into your reply so the user sees them inline. Optionally add a one-line caption above.

The `images` array in the result lists `path`, `url`, `bytes` and safety metadata for each image, in case you need to reason about what was produced. **Never echo the URLs as plain text or try to reconstruct base64** — the markdown the tool produced is the only thing that should appear in your reply.

If `rendered` is `0`, all images were blocked by the safety policy — tell the user briefly and suggest rephrasing.

## Constraints

- Write prompts in English even when the user writes in another language — the model performs better that way. Then reply to the user in their language.
- Keep prompts concrete: subject, style, lighting, framing. Avoid vague wishes ("make it nice").
- Do not request more than 4 images per call.
- Inform the user that each image consumes credits from their delegate balance.
- Never include base64 image data in your reply; always rely on the markdown the tool already produced.
