# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an Obsidian vault containing notes for Golang technical interview preparation. Notes are written in Russian and cover topics needed to successfully pass technical interviews for Golang positions.

## Repository Structure

This is an Obsidian vault, which means:
- All `.md` files in the root and subdirectories are Obsidian notes
- `.obsidian/` directory contains Obsidian configuration (workspace settings, plugins, themes)
- Notes use Obsidian's wiki-link syntax `[[Note Name]]` for internal linking
- Notes may contain YAML frontmatter for metadata

## Working with Notes

### Creating New Notes
- Create markdown files directly in the root or organize into subdirectories by topic
- Use descriptive filenames in Russian (e.g., `Горутины и каналы.md`, `Интерфейсы в Go.md`)
- Obsidian supports nested folders for organization

### Linking Notes
- Use wiki-link syntax: `[[Target Note]]` for internal links
- For links with custom text: `[[Target Note|display text]]`
- Obsidian automatically maintains backlinks between notes

### Common Topics for Organization
Expected content areas for Golang interview preparation:
- Language fundamentals (syntax, types, structs)
- Concurrency (goroutines, channels, sync primitives)
- Interfaces and polymorphism
- Error handling
- Standard library
- Testing
- Performance and optimization
- Common algorithms and data structures
- System design patterns

## Version Control

The vault uses the `obsidian-git` plugin for automatic Git integration:
- Plugin handles automatic commits and syncing
- Workspace files (`.obsidian/workspace.json`) may change frequently during normal use
- Plugin configuration is in `.obsidian/plugins/obsidian-git/`

When committing changes manually:
- Commit `.md` note files
- Generally avoid committing `.obsidian/workspace.json` unless intentionally sharing workspace layout
- `.obsidian/community-plugins.json` and plugin directories should be committed to preserve setup

## Language

All notes content is in Russian, matching the interview preparation context for Russian-language technical interviews.
