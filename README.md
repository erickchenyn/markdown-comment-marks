# Markdown Comment Marks

A system for adding inline comments to markdown documents. Comments are anchored to specific text spans using lightweight markup, stored alongside the document, and viewable in a browser-based reader.

## How It Works

1. **Mark** text in a `.md` file using one of 4 supported formats
2. **Store** comment metadata (id, quoted text, content) in a companion `comments.csv`
3. **View** the annotated document in `reader.html` with highlighted spans and a comments sidebar

## Comment Mark Formats

| Format | Syntax | Example |
|--------|--------|---------|
| `start-end-mark` | `<comment-start data-id="ID" />text<comment-end data-id="ID" />` | Paired HTML-style tags |
| `moxt-mark` | `<moxt-comment data-id="ID">text</moxt-comment>` | Single wrapping tag |
| `markdown-link` | `[text](comment:ID)` | Disguised as a markdown link |
| `markdown-note` | `text[^ID]` + CSV lookup | Footnote-style, relies on CSV `quoted_text` for anchor matching |

Each format is demonstrated in its own sample file (`start-end-mark.md`, `moxt-mark.md`, `markdown-link.md`, `markdown-note.md`).

## Reader

Open `reader.html` in a browser. It provides:

- Drag-and-drop or file picker for `.md` files
- Format selector to match the comment mark style used in the document
- Optional `.csv` loading for comment content display
- Highlighted comment spans with click-to-focus interaction
- Sidebar listing all comments with bidirectional navigation (click a highlight to focus its card, click a card to scroll to its highlight)

## CSV Format

```csv
id,quoted_text,content
<uuid>,"<the marked text>","<comment body>"
```

- `id`: unique identifier matching the mark in the markdown
- `quoted_text`: the text span being commented on
- `content`: the comment itself (can be empty)

## Claude Code Skills

Three skills for programmatic comment manipulation:

- `/add-comment` - Insert a new comment mark and CSV entry
- `/delete-comment` - Remove a comment mark and its CSV entry
- `/read-comments` - List all comment IDs and their data from a file
