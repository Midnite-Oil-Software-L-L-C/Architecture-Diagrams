## Overview

These custom subagents audit and review Legends of Learning (LoL) games against the team's platform requirements and architecture. Each one runs on its own, has no way to ask questions mid-task, and returns one structured report.

All three are **read-only**. They never edit files, scenes, prefabs, assets, or settings, and they never trigger a recompile. Any fix is a separate step that you approve.


| Subagent                        | Purpose                                             | Model / effort             | Definition file                                     |
| ------------------------------- | --------------------------------------------------- | -------------------------- | --------------------------------------------------- |
| `lol-compliance-auditor`        | LoL submission constraints and build settings       | `claude-sonnet-5` / medium | `/.bezi/subagents/lol-compliance-auditor.md`        |
| `localization-coverage-scanner` | Hardcoded text and `language.json` key coverage     | `claude-sonnet-5` / medium | `/.bezi/subagents/localization-coverage-scanner.md` |
| `architecture-reviewer`         | Design and code review against the LoL architecture | `claude-opus-5-5` / high   | `/.bezi/subagents/architecture-reviewer.md`         |


**Sources they rely on:**

- [@ id="node:14ab595b-4ed2-4e20-bb44-a2c96143cc23" label="Project Requirements and Settings"]: platform constraints, used by the compliance auditor and the architecture reviewer.
- [@ id="node:b17ef131-c20e-41e6-a990-d50272eae994" label="Legends of Learning Architecture and Best Practices"]: architecture rules, used by the architecture reviewer.
- `Assets/StreamingAssets/language.json` and `Assets/_project/Scripts/Legends of Learning/UI/LocalizedText.cs`: localization references, used by the localization scanner.

Keep these pages and files accurate. The subagents treat them as the source of truth, and the compliance auditor and architecture reviewer stop if their required page is missing.

## Choosing the right subagent


| You want to...                                                                                  | Use                                                      |
| ----------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| Check whether the game is ready to submit to LoL                                                | `lol-compliance-auditor`                                 |
| Confirm a new input or UI change is touch-safe and has navigation set to None                   | `lol-compliance-auditor`                                 |
| Verify WebGL compression, WASM streaming, render pipeline, or Unity/SDK version                 | `lol-compliance-auditor`                                 |
| Find text that bypasses `LocalizedText` or `GetText`                                            | `localization-coverage-scanner`                          |
| Find keys missing from a locale, or referenced keys that don't exist                            | `localization-coverage-scanner`                          |
| Review a design before building a new system                                                    | `architecture-reviewer`                                  |
| Review a finished feature's scripts for ownership, lifetime, EventBus, and save/progress issues | `architecture-reviewer`                                  |
| Check formatting or naming style                                                                | None of these; use the `style-linter-pass` skill         |
| Check for missing references or broken prefabs                                                  | None of these; use the `scene-prefab-health-check` skill |


## lol-compliance-auditor

**What it checks**

- Installed Unity version and LoL SDK compared with the API version requirements.
- Render pipeline: no HDRP and no custom SRP.
- Keyboard input in project code and `.inputactions` assets. Editor-only code is ignored.
- Navigation mode None on every `Selectable` (Button, Toggle, Slider, Dropdown, TMP_InputField, Scrollbar) in scenes and prefabs.
- EventSystems use the Input System UI module rather than `StandaloneInputModule`.
- WebGL player settings: compression (Brotli preferred, GZip acceptable) and WASM streaming instantiation.
- The 30 MB uncompressed build limit. This stays **Unverified** unless a build report exists, but obvious size risks are noted.

**When to use it**

- Before every LoL submission.
- After changes to input, UI selectables, EventSystems, player settings, or build profiles.

**Example prompts**

- "Run the LoL compliance auditor on the project."
- "Check whether the Global Engine scene's UI meets the LoL navigation and input requirements."

**What you get back**

1. A summary with Pass, Fail, and Unverified counts and a readiness verdict.
2. A checklist table: Requirement | Status | Evidence | Smallest fix.
3. Failures ordered by how much they block submission.
4. A list of what it did not inspect.

**Limitations:** It cannot test on the live LoL platform or a real touch device. Those items are marked Unverified and need manual testing.

## localization-coverage-scanner

**What it checks**

- Keys missing between locales, and keys with empty values, in `language.json`. Section-header keys are skipped.
- Hardcoded player-facing strings in code, such as `.text =`, `SetText(`, dialogue or hint calls, and interpolated display text.
- Serialized text in TMP_Text and Text components in scenes and prefabs that isn't driven by `LocalizedText` or a runtime controller.
- Prose stored in ScriptableObject fields where a localization key is expected.
- Keys referenced in code or assets but missing from `language.json`.
- Keys in `language.json` never referenced anywhere, flagged as low confidence because keys may be built dynamically.

**Excluded:** log and exception messages, editor-only code, commented-out code, confirmed runtime-overwritten placeholder text, and third-party or SDK folders.

**When to use it**

- After adding UI, dialogue, hints, or content data.
- Before a submission, alongside the compliance auditor.
- When a `--missing--` string appears in play mode.

**Example prompts**

- "Scan localization coverage for the whole project."
- "Check the hint assets and Global Engine scene for hardcoded text."

**What you get back**

1. A summary with counts per category and the locales found.
2. Hardcoded strings with location, text, a suggested key name, and confidence.
3. Missing locale keys.
4. Referenced but undefined keys.
5. Unused JSON keys.
6. A list of what it did not inspect.

**Limitations:** It checks structure only. It can't judge translation quality or accuracy.

## architecture-reviewer

**What it checks**

- **Reuse:** whether existing managers, base classes, prefabs, events, or config types already cover the need. It flags duplicate services and parallel bootstrap paths.
- **Layer ownership:** SDK, save, progress, or language handling leaking into gameplay, and shared code depending on concrete game code.
- **Lifetime:** application versus scene versus attempt lifetime, duplicate guards, stale scene references, and reset on retry.
- **Singletons:** new MonoBehaviour singletons derive from `SingletonMonoBehaviour<T>`, and scene-local systems such as hints stay non-persistent.
- **Single completion path:** it traces each outcome through to the game manager, progress, and save, and flags double processing or a missing guard against repeated completion.
- **Data separation:** config ScriptableObjects aren't changed at runtime, and saved data stays serializable and compatible with existing saves.
- **Communication:** EventBus events are structs with public constructors and no interface, and subscribe/unsubscribe calls are paired correctly.
- **Readiness:** no arbitrary delays or reliance on the order of unrelated `Start()` calls. New timers, tweens, input, and audio handle platform pause and resume.
- **Hints and narration:** hints use context interfaces, are held back during dialogue and tutorials, and narration stays separate from progression.
- **Platform fit:** touch-first input and WebGL size or performance risks.

**When to use it**

- **Before implementation:** describe the proposed system or paste the design so problems are caught early.
- **After implementation:** name the scripts, folder, or feature to review.
- Whenever a change touches save data, completion, progress, or the EventBus.

**Example prompts**

- "Have the architecture reviewer assess this design for a plate boundary scoring system: ..."
- "Review the scripts under `Assets/_project/Scripts/Tectonics` for architecture issues."
- "Trace the level completion path and check that progress is saved only once."

**What you get back**

1. A verdict: Approve, Approve with changes, or Rework.
2. Findings ordered by severity (Blocker, Major, Minor), each with the location, the rule and page section it breaks, why it matters, and the smallest compliant change.
3. A trace of the completion and progress path, when it's in scope.
4. Open questions: decisions the architecture page doesn't settle.
5. A list of what it did not inspect.

**Limitations:** It separates confirmed violations from risks that depend on code it couldn't see. It doesn't review formatting or naming, and it leaves full localization key audits to the scanner.

## How to run them

1. **Ask for the task in chat.** Bezi picks the subagent from its description, or you can name it directly, for example "Use the architecture-reviewer to...".
2. **Keep the scope narrow when you can.** Naming a scene, folder, feature, or design gives faster, more accurate reports than a whole-project scan.
3. **Run them in parallel before a submission.** Ask Bezi to run all three together (up to three at once) and combine their findings into one readiness report.
4. **Apply fixes separately.** Review the report, then ask Bezi in Agent mode to fix specific findings. The subagents themselves never change the project.

**Suggested workflow for a new feature:**

1. `architecture-reviewer` on the design.
2. Implement the feature.
3. `architecture-reviewer` on the changed scripts, plus `localization-coverage-scanner` on the new UI and content.
4. `lol-compliance-auditor` if input, UI, or build settings changed.

## Maintaining the subagents

- Definitions live in `/.bezi/subagents/`. The file name must exactly match the `name` field in the frontmatter, or the subagent won't load.
- Don't put unquoted colons in frontmatter values; they break the file's parsing.
- If a model name isn't recognized, Bezi silently picks a model instead. Update `modelIds` if you want a specific model or a different cost.
- The `/.bezi/subagents` folder isn't covered by checkpoints, so edits there can't be undone from the revert UI. Keep a backup before large changes.
- After editing a definition, test it on a known problem, such as a planted hardcoded string or a button with navigation turned on, to confirm it still catches it.
- When the requirements or architecture pages change, the subagents pick up the new rules automatically on their next run. Update a definition only when its checks, scope, or report format need to change.