# Agent Note: pi-ai compaction admission marker

Status: implemented

English | [中文](2026-08-26-pi-ai-compaction-admission-marker.zh.md)

## Problem

The [compaction capability decision](2026-06-18-compaction-capability-seam.md) identifies Compact-basic's summarizer call with `GenerateOptions.purpose = 'compaction'`, but the pi-ai adapter discarded that transport-only purpose. A gateway that protects a recovery reserve therefore applied its ordinary hard input limit to the summarizer itself. Near the limit, the original request correctly triggered overflow recovery, then the recovery request could exceed the same limit and leave the session unable to compact.

## Decision

The pi-ai adapter maps only `purpose === 'compaction'` to `x-deepseek-harness-compact: 1` in the provider request headers, matching the direct DeepSeek adapter's existing wire contract. The marker does not enter the model-visible request body. It is Harness-owned alongside attribution headers: configured profile headers are filtered case-insensitively so a deployment cannot mark ordinary requests permanently or replace the marker value.

Gateways may use the exact marker to select a recovery-specific admission limit. They remain responsible for enforcing a physical context bound and output headroom; the marker describes request purpose, not permission to exceed the model window.

## Alternatives considered

**Raise the advertised model context or ordinary gateway input cap.** Rejected because it moves every agent turn into the recovery reserve and recreates the slow, difficult-to-compact state the reserve exists to prevent.

**Infer summarization from prompt text, request size, or user agent.** Rejected because prompt inspection is brittle and model-visible, while request size and attribution cannot distinguish a user turn from the internal summarizer call.

**Allow the marker through configured profile headers.** Rejected because profile headers apply to every request on a route and would silently disable the ordinary admission boundary.

## Consequences

Pi-ai routes can now cooperate with gateways that reserve context specifically for recovery, so an overflow-triggered summary has room to run while ordinary requests remain below the operational limit. Non-compaction requests are byte-for-byte unchanged except that a conflicting configured marker is removed. Gateways that ignore the header retain their previous behavior.
