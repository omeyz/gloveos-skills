# gloveos-skills

Agent Skills for **GloveOS** (a GrapheneOS fork with Android 17 Computer Control turned on) and **Glove**, the on-device harness that lets the model you choose use your phone. The same `SKILL.md` folders work in Claude Code and in Codex.

| Skill | For | Status |
|---|---|---|
| `gloveos-build` | People building, signing, flashing and updating GloveOS, and pushing the Glove app over adb | usable |
| `glove-dev` | People changing the Glove harness app (tools, agent loop, model wire formats, speech) | usable |
| `glove-phone` | People driving a GloveOS phone from a laptop through the Glove MCP server | waiting on the MCP server |

## Install

Claude Code (as a plugin):

```
/plugin marketplace add omeyz/gloveos-skills
/plugin install gloveos@gloveos-skills
```

Codex (skills only): copy or symlink the folders under `skills/` into `~/.agents/skills/`, or into `.agents/skills/` inside the GloveOS checkout.

## Layout

```
.claude-plugin/marketplace.json     lists the "gloveos" plugin
plugins/gloveos/                    the Claude Code plugin; its skills/ is a symlink to ../../skills
skills/<name>/SKILL.md              the skills (Agent Skills format), plus references/ and agents/openai.yaml
```

The plugin ships no `.mcp.json` yet. When the phone-hosted MCP server lands, `plugins/gloveos/.mcp.json.example` becomes `.mcp.json`.

## Source

GloveOS and Glove source: link to follow when the repository is published.
