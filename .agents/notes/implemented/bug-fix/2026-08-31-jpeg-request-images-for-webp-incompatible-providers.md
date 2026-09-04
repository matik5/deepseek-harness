# Agent Note: JPEG request images for WebP-incompatible providers

Status: implemented

English | [中文](2026-08-31-jpeg-request-images-for-webp-incompatible-providers.zh.md)

## Problem

An image-capable OpenAI-compatible provider may accept PNG and JPEG while its media decoder rejects WebP with HTTP 400. Durable normalization can legitimately store WebP, and the route request transform also produced WebP whenever an alpha-bearing PNG needed resizing or recompression. A successful `read_image` result therefore entered session history before the next model request failed; every later turn replayed the same rejected media. Provider model metadata declares the image modality but does not declare accepted image media types, so the adapter cannot negotiate the format before the first request.

## Decision

The local attachment backend keeps durable normalization provider-independent: clean in-budget PNG, JPEG, and WebP still pass through, and normalization that must preserve alpha still uses WebP. Model-request projection has a separate compatibility rule. An in-budget PNG or JPEG passes through unchanged, while every WebP attachment and every image requiring route resizing or recompression is encoded as JPEG through the 85/75/60 quality ladder. JPEG conversion composites transparent pixels over white. The request transform version is `request-image-v6`; its identity records the JPEG qualities and alpha background, so version-five WebP cache entries are not reused.

The conversion runs before an adapter serializes inline base64 or uploads a provider file. Frontends, including DSHmux, do not inspect or rewrite attachment bytes: internal tool results and outbound model requests live in the Harness process, and request projection is the one path shared by chat uploads, `read_image`, compaction, direct LLM calls, and provider Files.

This decision partially supersedes the request-encoding part of the [alpha-routed image quality ladders note](2026-08-24-alpha-routed-image-quality-ladders.md). Its alpha-routed durable normalization, fixed quality ladder, byte-target semantics, and total-pixel sizing remain unchanged. It also updates the deterministic request-version facts in the [unified image request pipeline note](../feature/2026-08-20-unified-image-request-pipeline.md).

## Alternatives considered

**Convert inside DSHmux.** Rejected: DSHmux relays browser traffic to DSH, but an internal tool image and DSH's provider request never cross that relay. Intercepting them would require an outbound provider proxy with image-decoder native dependencies and would still miss non-DSHmux Harness clients.

**Retry only after the provider returns the generic media-decoder 400.** Rejected: the provider response does not identify which image or media type failed, an invisible retry would make the successful model input depend on an unlogged provider response, and one extra rejected request would be spent for every new variant.

**Convert WebP to PNG and preserve alpha.** Rejected: the observed endpoint has also rejected request images whose durable reference was PNG because route resizing re-encoded them as WebP, and JPEG is the common denominator required by this deployment. Durable history continues to preserve transparency for later routes.

## Consequences

- OpenAI-compatible backends need JPEG decoding for transformed request images and no longer need WebP decoding for Harness-generated requests.
- A transparent WebP, or any transparent image that exceeds a route request budget, reaches the model composited over white. Its durable normalized attachment retains alpha and remains available to a future projection policy.
- Existing request-image caches regenerate under `request-image-v6`; durable attachment ids and session logs do not change.
- Focused attachment tests cover in-budget WebP conversion, alpha compositing metadata, cache validation, route resizing, and the unchanged normalized-image path. The Blender reproduction stores `render_full.png` as WebP but projects it as a 1280×960 JPEG before pi-ai serialization.
