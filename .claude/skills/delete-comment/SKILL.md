---
name: delete-comment
description: Read the given file, find the comment mark with the given id that needs to be deleted.
---

- Read the given file, find the comment mark with the given id that needs to be deleted.
  - Throw an error if not found, and skip next.
- Edit the `comments.csv` file, find and delete the row with the comment id (uuid).
  - Throw an error if not found, and skip next.
- Edit the given file, remove the comment mark (recognize how to do this in the specific file) with the given comment id. Do not delete the content that was commented on.
