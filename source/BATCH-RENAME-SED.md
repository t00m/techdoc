---
Category: Procedure
Command: sed, find, grep
Date: 2026-04-16
DocType: How-to guide
OS: Linux
Product: KB4IT
Scope: Linux Administration
Tag: batch, rename, in-place, text
Topic: Knowledge Management, Troubleshooting
---

# Batch rename with sed

## Goal

Replace a property name across hundreds of Markdown files in a single operation - for example, renaming `Updated:` to `Date:` throughout an entire KB4IT document repository.

## Prerequisites

- A POSIX-compatible shell (bash, sh, ksh)
- GNU `sed` (required for the `-i` in-place flag)
- Read/write access to the target directory

## Steps

### 1. Dry Run - Identify Affected Files

Before making any changes, list every file that contains the old property name:

```bash
grep -rl 'Updated:' /path/to/docs --include="*.md"
```

Review the output to confirm the scope matches your expectations.

### 2. Single Directory

If all files are in one flat directory, run:

```bash
sed -i 's/Updated:/Date:/g' *.md
```

The `-i` flag edits each file in-place. The `g` flag replaces all occurrences per line, though a property name typically appears only once per file.

### 3. Recursive - Multiple Subdirectories

For a directory tree, use `find` with the `+` terminator to batch files into a single `sed` invocation (more efficient than one process per file):

```bash
find /path/to/docs -name "*.md" -exec sed -i 's/Updated:/Date:/g' {} +
```

### 4. Verify the Result

Confirm no occurrences of the old name remain:

```bash
grep -r 'Updated:' /path/to/docs --include="*.md"
```

An empty result means the rename is complete.

> **WARNING:** The `-i` flag modifies files permanently. Ensure you have a backup or the repository is under version control (e.g. Git) before running the command.

> **TIP:** On macOS, GNU `sed` is not the default. Either install it via Homebrew (`brew install gnu-sed`) and use `gsed`, or append a backup suffix: `sed -i '' 's/Updated:/Date:/g' *.md`.

