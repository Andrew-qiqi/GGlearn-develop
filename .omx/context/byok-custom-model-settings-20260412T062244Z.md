## Task Statement

Fix the BYOK "My API" settings bug around custom OpenAI-compatible models in SlideTutor while preserving the current Settings layout.

## Desired Outcome

- Keep the current top-to-bottom Settings layout.
- Keep `Select Model` as the only model-selection entry point.
- For built-in Gemini models, show only the Gemini API key field.
- For built-in OpenAI-compatible models, show only the OpenAI-compatible API key field.
- For `Custom OpenAI-compatible`, show full editable credentials: API key, Base URL, and Model ID.
- Remove the current coupling where `Select Model` and `My API Credentials` can contradict each other.
- Fix the current UI bug where typing a custom model id makes the select appear to jump back to Gemini 3 Flash.

## Known Facts / Evidence

- `SettingsModal.tsx` currently encodes `providerId + endpointPreset + modelId` into the `Select Model` value.
- `Endpoint Preset` currently mutates both `selectedModel` and `aiAccess`.
- Custom model id currently writes directly into `selectedModel.modelId`.
- The select options only contain the sentinel custom option value `custom-openai-model`, so when the real custom model id replaces it, the `<select>` value no longer matches any option and visually falls back to the first item.
- Current docs and discussion converged on keeping the current layout and simplifying the logic rather than redesigning the whole surface.

## Constraints

- Preserve existing Settings surface layout as much as possible.
- No new dependencies.
- Keep diffs reviewable and reversible.
- Respect the current `Platform API` boundary: platform must not support custom OpenAI-compatible models.
- Run verification before claiming completion.

## Unknowns / Open Questions

- Whether capability-check flow for custom models should stay pending/unverified or be adjusted in the same implementation pass.
- Whether platform mode should hide or disable the custom option in the select, or just reject it at runtime plus preserve current behavior.

## Likely Codebase Touchpoints

- `SlideTutor-AI/src/components/SettingsModal.tsx`
- `SlideTutor-AI/src/config/models.ts`
- `SlideTutor-AI/src/store/uiStore.ts`
- `SlideTutor-AI/src/lib/api/apiClient.ts`
- `SlideTutor-AI/src/components/SettingsModal.test.tsx`
- `SlideTutor-AI/src/lib/api/apiClient.test.ts`
