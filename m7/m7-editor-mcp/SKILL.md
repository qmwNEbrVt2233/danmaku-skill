---
name: m7-editor-mcp
description: Use the m7-editor MCP tools to inspect and edit a running m7-editor session. Covers tool input structures, creation rules, expressions, variables, and rendering behavior; does not prescribe artistic creation techniques.
---

# m7-editor MCP

Use this skill when a task should be carried out through the running m7-editor MCP session. The desktop app owns the project state; MCP is a separate client-facing service that calls that session.

Before editing, read the current state with `get_editor_state`. Ordinary field and toolbar edits act on the current selection. Read the relevant danmaku first when the requested target or values are not clear. After an edit, use the returned result and, when useful, `list_danmakus` or `get_frame` to verify it. Successful edits are separate undo steps; a generated batch is one edit.

For exact tool arguments, danmaku object shape, creation-rule modes, expression scopes, and renderer semantics, read [references/mcp-api.md](references/mcp-api.md). Use the active MCP tool schemas as the final authority if the running server differs from the reference.

Feature descriptions and user-facing terminology are maintained in `src/components/preference/about.vue`. When changing this skill after product changes, compare it with that file and the relevant MCP/generator/renderer source. This skill documents interface behavior and data semantics; it intentionally leaves artistic production techniques to the user.

This skill is designed for m7-editor_v1.11.0. If necessary, you can go to [repository](https://github.com/qmwNEbrVt2233/m7-editor) to view the specific definition and behavior of the source code.