---
name: italy-lab
description: Italian language learning knowledge base and processing pipeline. Use when working on the Italy-lab repository — processing lesson transcripts, extracting vocabulary and mistakes, creating exercises, managing inbox workflow, or any Italian learning content.
---

# Italy-lab Skill

## Rules

- **Conversation with the user is in Slovak**
- All Italian learning content is Italian ↔ Slovak (not English)
- YAML front matter in files uses English keys
- File names use English conventions
- Never delete raw user data from `00_inbox/`
- Never invent new folder structures — use existing ones
- If something is ambiguous, add `<!-- TODO: ... -->` or `<!-- NOTE: ... -->`, never silently improvise
- Always use templates from `templates/` when creating new content files
- Every processed output must reference its source in YAML `source` field

## Repository

**Path:** `D:/personal/Italy-lab/`

## Structure

```
Italy-lab/
├── 00_inbox/          Raw inputs — transcripts, notes, vocab dumps
├── 01_grammar/        Grammar rules and explanations
├── 02_vocabulary/     Word lists and expressions
├── 03_exercises/      Exercises and practice
├── 04_lessons/        Processed lessons
├── 05_mistakes/       Error log and corrections
├── 06_dialogues/      Dialogues and conversations
├── 07_writing/        Written texts
├── 08_speaking/       Pronunciation, speaking
├── 09_exports/        Exports (Anki, PDF, ...)
├── templates/         Markdown templates for all content types
├── scripts/           Processing scripts (planned)
├── docs/              Rules, workflows, documentation
│   └── workflows/     Step-by-step processing workflows
└── config/            Configuration files
```

## Naming Conventions

### Inbox (raw files)

Format: `YYYY-MM-DD_type_raw.md`

Examples:
- `2026-04-08_lesson_transcript_raw.md`
- `2026-04-08_vocab_dump_raw.md`
- `2026-04-08_notes_raw.md`

### Processed files

| Folder | Name format | Example |
|---|---|---|
| `01_grammar/` | `topic_name.md` | `passato_prossimo.md` |
| `02_vocabulary/` | `YYYY-MM-DD_source_vocab.md` | `2026-04-08_lesson_vocab.md` |
| `03_exercises/` | `YYYY-MM-DD_type_exercises.md` | `2026-04-08_review_exercises.md` |
| `04_lessons/` | `YYYY-MM-DD_lesson.md` | `2026-04-08_lesson.md` |
| `05_mistakes/` | `YYYY-MM-DD_mistakes.md` | `2026-04-08_mistakes.md` |
| `06_dialogues/` | `YYYY-MM-DD_topic_dialogue.md` | `2026-04-08_ristorante_dialogue.md` |
| `07_writing/` | `YYYY-MM-DD_topic.md` | `2026-04-08_lettera_amico.md` |
| `08_speaking/` | `YYYY-MM-DD_topic.md` | `2026-04-08_pronunciation_r.md` |

### YAML front matter

All processed files use:

```yaml
---
title: "Title"
type: lesson | grammar | vocabulary | exercise | mistake | dialogue | writing | speaking
status: draft | review | done
date: YYYY-MM-DD
source: "00_inbox/filename_raw.md"
tags: [tag1, tag2]
---
```

## Workflows

### Lesson transcript processing

**Input:** `00_inbox/YYYY-MM-DD_lesson_transcript_raw.md`

**Outputs (4 files):**

1. `04_lessons/YYYY-MM-DD_lesson.md` — summary, vocab, grammar, corrections, homework
2. `05_mistakes/YYYY-MM-DD_mistakes.md` — error log, patterns, action items
3. `02_vocabulary/YYYY-MM-DD_lesson_vocab.md` — new words and expressions
4. `03_exercises/YYYY-MM-DD_review_exercises.md` — practice based on lesson content

**Steps:**
1. Read the full raw transcript
2. Create lesson summary using `templates/lesson_template.md`
3. Extract all mistakes using `templates/mistake_template.md`
4. Extract vocabulary using `templates/vocab_template.md`
5. Create review exercises using `templates/exercise_template.md`
6. Set `source` in all output files to the raw input path
7. Set `status: draft` in all outputs
8. Do NOT delete the raw file

**Full workflow doc:** Read `docs/workflows/lesson_transcript.md` in the repo.

### Inbox cleanup

1. List all files in `00_inbox/`
2. Identify type of each file
3. Process according to type (lesson transcript → 4 outputs, vocab dump → vocabulary, etc.)
4. Use appropriate template for each output
5. Fully processed raw files may be manually moved to `00_inbox/archived/`

**Full workflow doc:** Read `docs/workflows/inbox_cleanup.md` in the repo.

### Weekly review

1. Collect all mistakes from the week (`05_mistakes/`)
2. Identify recurring error patterns
3. Collect new vocabulary from the week (`02_vocabulary/`)
4. Create `03_exercises/YYYY-MM-DD_weekly_review.md` with targeted exercises
5. If a grammar rule is repeatedly violated, create/update entry in `01_grammar/`

**Full workflow doc:** Read `docs/workflows/weekly_review.md` in the repo.

## Templates

Before creating any content file, read the appropriate template:

| Type | Template |
|---|---|
| Lesson | `templates/lesson_template.md` |
| Grammar | `templates/grammar_template.md` |
| Vocabulary | `templates/vocab_template.md` |
| Mistakes | `templates/mistake_template.md` |
| Exercise | `templates/exercise_template.md` |

## Important constraints

- Do not create files outside the defined folder structure
- Do not move existing files without explicit user request
- Do not add unnecessary external dependencies
- Keep everything git-friendly (no large binaries)
- Prefer simplicity — no over-engineering
- Everything must be readable without AI
