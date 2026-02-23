# Contributing to AqualinkD

## Commit Message Format

This project enforces a consistent commit message format via a `commit-msg` git hook.

### Setup

Install the git hooks after cloning:

```
make install-hooks
```

### Rules

**Subject line:**
- Start with a capital letter
- Use imperative mood ("Add feature" not "Added feature")
- Keep to 72 characters or fewer
- Do not end with a period
- Be specific — bare words like "Update" or "Fix" are rejected

**Body (optional):**
- Separate from subject with a blank line
- Wrap at 72 characters
- Explain *what* and *why*, not *how*

### Examples

Good:
```
Add MQTT discovery for variable-speed pumps

The previous implementation only published on/off state. This adds
RPM and watts attributes so Home Assistant can display pump speed.
```

```
Fix SWG percentage not updating after panel reset
```

Bad:
```
fix swg bug.          # lowercase, trailing period, vague
Updated the docs      # past tense, vague
Fix                   # single vague word
```
