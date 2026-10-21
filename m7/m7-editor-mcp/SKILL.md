---
name: m7-editor-mcp
description: Use the m7-editor MCP tools to inspect, create, edit, render, save, and back up danmakus in a running m7-editor session, including rendering behavior and practical production techniques.
---

# m7-editor MCP

Use this skill when a task should be carried out through the running m7-editor MCP session. The desktop app owns the project state; MCP is a separate client-facing service that calls that session.

Before editing, read the current state with `get_editor_state`. Ordinary field and toolbar edits act on the current selection. Read the relevant danmaku first when the requested target or values are not clear. After an edit, use the returned result and, when useful, `list_danmakus` or `get_frame` to verify it. Successful edits are separate undo steps; a generated batch is one edit.

For exact tool arguments, danmaku object shape, creation-rule modes, expression scopes, and tool behavior, read [references/mcp-api.md](references/mcp-api.md). Use the active MCP tool schemas as the final authority if the running server differs from the reference. Read [references/rendering-rules.md](references/rendering-rules.md) when positioning or styling danmakus, and [references/technique.md](references/technique.md) when translating general M7 production techniques into MCP operations.

Use the number of danmakus that best expresses the result; do not let broad performance concerns unnecessarily limit the design. Very large numbers of simultaneously moving danmakus or rapid frame-by-frame node creation and removal can make preview playback stutter. Frame-by-frame export may take longer, but does not reduce final quality. Use generated batches when their shared formulas fit. After each meaningful stage of editing a folder project, promptly save or back it up with the MCP project tools; local work still needs recovery points. Those tools only support folder projects.

Feature descriptions and user-facing terminology are maintained in `src/components/preference/about.vue`. When changing this skill after product changes, compare it with that file and the relevant MCP/generator/renderer source. This skill documents interface behavior, rendering semantics, and practical mappings from common animation techniques to available MCP operations.

This skill is designed for m7-editor_v1.11.0. If necessary, you can go to [repository](https://github.com/qmwNEbrVt2233/m7-editor) to view the specific definition and behavior of the source code.
