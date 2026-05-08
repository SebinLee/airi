# AIRI repository architecture analysis task

Analyze the local repository `/workspace/airi` for explaining how the AI VTuber AIRI source code is structured and how the listed features work in code.

Use OMC/Claude Code with ralph-style autonomous analysis. Do not modify production source code. You may create concise report artifacts under `.agent_reviews/` only.

Write one Markdown report per worker under `.agent_reviews/omc_worker_<role>.md` with file/directory evidence and exact paths. Keep findings grounded in code.

Final user-facing analysis must support Korean output and include these topics:

1. Overall System Architecture with a diagram.
2. Whether AI Provider can be changed (whether LLMs besides Grok can be used; if yes, which files/directories are involved and what to modify).
3. Persona settings for the AI VTuber, and whether persona can be updated during broadcast based on content.
4. How chat, donations, and other user interactions are collected and how AIRI reacts.
5. Avatar control, separated into 2D (Live2D) and 3D (VRM/OBX or equivalent supported formats).
6. Which existing files/directories appear removable or legacy/unused, with caution level.

Suggested worker split:
- architect: monorepo/system architecture and package/app boundaries.
- analyst: provider/persona/memory configuration flows.
- code-reviewer: interactions/chat/donations/reaction pipeline.
- planner: avatar/2D/3D/rendering pipeline and supported formats.
- critic: removable/legacy parts and cross-check gaps.

Important repository conventions from AGENTS.md:
- Current desktop is Electron (`apps/stage-tamagotchi`); `crates/` is legacy Tauri.
- `packages/stage-ui` is core stage business UI/composables/stores.
- `packages/i18n` holds translations.
- Use Eventa for IPC/RPC, injeca for DI, UnoCSS for styling.

Return evidence-focused, high-level overview first, then directory/file role details for each topic.
