## Original User Task

Use an OMX team-style workflow to audit and implement a fix for the SlideTutor BYOK custom OpenAI-compatible settings bug, with Gemini contributing the frontend implementation guidance.

## Final Prompt Sent To Gemini CLI

You are implementing a frontend/settings bugfix in the SlideTutor repo at `c:\Users\hoo\Documents\z_cqmeng_file\local_repository\SlideTutor-AI-main`. Read these files and propose an exact implementation patch plan with concrete code-level changes, preserving current Settings layout:

- `SlideTutor-AI/src/components/SettingsModal.tsx`
- `SlideTutor-AI/src/config/models.ts`
- `SlideTutor-AI/src/store/uiStore.ts`
- `SlideTutor-AI/src/lib/api/apiClient.ts`
- `SlideTutor-AI/src/components/SettingsModal.test.tsx`
- `SlideTutor-AI/src/lib/api/apiClient.test.ts`

Bug and required product behavior:

- Keep Select Model as the only model-selection entry point.
- Built-in Gemini selected -> show only Gemini API key.
- Built-in OpenAI-compatible selected -> show only OpenAI-compatible API key.
- Custom OpenAI-compatible selected -> show API key + Base URL + Model ID.
- Do not change the overall layout or the `My API Credentials` heading.
- Platform API must still not support custom OpenAI-compatible models.

Technical direction to follow:

- Keep `accessMode + selectedModel + aiAccess` overall structure.
- `selectedModel` stays the catalog selection only.
- For custom selection, `selectedModel` must remain fixed at the sentinel custom option instead of storing the real user-entered model id.
- Add separate `aiAccess.openAiCompatible` custom fields for the real custom base URL and custom model id.
- Built-in OpenAI-compatible models must derive `endpointPreset/baseURL/modelId` from `selectedModel/config` and must not expose editable `endpoint/baseURL/model` fields in the UI.
- `apiClient` must resolve runtime request params from `selectedModel + aiAccess`, using `customModelId/customBaseURL` only when `selectedModel` is custom.

Please output:

1. file-by-file change list
2. important edge cases
3. any test cases to add
4. representative code snippets for the tricky parts

## Gemini Output (Raw)

```text
Based on the code investigation, here is the implementation patch plan to fix the frontend settings bug and fulfill the product requirements.

1. File-by-File Change List
- Update models.ts AiAccessSettings with customBaseURL/customModelId and normalize defaults.
- Update apiClient.ts to resolve baseURL/modelId differently for built-in vs custom OpenAI-compatible selections.
- Update SettingsModal.tsx to remove redundant endpoint preset editing and show only the fields needed for the selected model type.

2. Important Edge Cases
- Empty custom model id should block generation.
- Platform mode must not allow custom models.
- Switching between built-in and custom must preserve saved custom values.

3. Test Cases To Add
- Built-in OpenAI-compatible hides base URL/model id inputs.
- Custom selection shows base URL/model id inputs.
- Custom runtime request replaces sentinel model id with customModelId.
- Platform mode rejects custom.

4. Representative Code Snippets
- Suggested a runtime resolver in apiClient and conditional rendering in SettingsModal for built-in vs custom paths.
```

## Concise Summary

Gemini's recommendation matched the intended UI direction: keep the current layout, keep `Select Model` as the sole entry point, move real custom OpenAI-compatible identity into dedicated `aiAccess` fields, and derive built-in OpenAI-compatible runtime config from the selected catalog model.

## Action Items / Next Steps

- Add failing tests first for the modal rendering and runtime request assembly.
- Incorporate critic-review gaps before implementation:
  - migration for legacy saved custom state
  - one shared effective OpenAI-compatible resolver
  - explicit custom runtime selection in capability-check and request assembly
  - platform invalid-state handling for custom
