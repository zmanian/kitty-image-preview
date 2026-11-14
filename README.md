# Kitty Image Preview

An agent skill for displaying clear screenshots and images in graphics-capable terminals, with a measured sizing workflow for Rex. Works with Codex and Claude Code.

The skill uses [Showme](https://github.com/sandwichfarm/showme) with its explicit Kitty graphics backend. The Kitty application itself is not required when the terminal supports the protocol.

## What it does

- Measures the destination pane, including terminal cell pixel dimensions.
- Calculates image geometry from the source dimensions while preserving proportions within terminal-cell rounding.
- Reserves space for the shell prompt and rerenders after pane resizing or maximizing.
- Keeps rendering commands out of busy agent panes and distinguishes successful rendering commands from visual confirmation.

This is an instruction skill, not an interactive image viewer. Showme renders static images once; the agent must remeasure and rerender after resizing. It does not add drag-to-pan, scroll-to-zoom, or a continuous resize watcher.

## Install

For Codex:

```sh
mkdir -p ~/.codex/skills
git clone https://github.com/zmanian/kitty-image-preview.git ~/.codex/skills/kitty-image-preview
```

For Claude Code:

```sh
mkdir -p ~/.claude/skills
git clone https://github.com/zmanian/kitty-image-preview.git ~/.claude/skills/kitty-image-preview
```

If the destination already exists, review it before replacing it. Start a new agent session if the skill is not discovered immediately.

## Use

Ask the agent to use `kitty-image-preview` to show a screenshot in a correctly sized terminal pane. In Codex, you can explicitly invoke `$kitty-image-preview`.

Showme and a terminal supporting Kitty graphics must be available. Rex users also need the Rex CLI. The documented workflow was checked against Showme 0.1.1 and Rex CLI 0.1.0-preview.18; the skill instructs agents to inspect installed capabilities before relying on them. On macOS, `sips` can read image dimensions without changing the source.

See [SKILL.md](SKILL.md) for the full workflow.
