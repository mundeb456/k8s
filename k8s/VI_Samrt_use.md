# Vim / Vi Cheat Sheet for CKA

> A quick reference for the most commonly used `vi` commands during the CKA exam.

---

# 1. Modes in Vim

| Mode             | Purpose                     |
| ---------------- | --------------------------- |
| **Normal Mode**  | Navigation and commands     |
| **Insert Mode**  | Typing and editing text     |
| **Command Mode** | Save, quit, search, replace |

> **Tip:** Press `Esc` anytime to return to **Normal Mode**.

---

# 2. Enter Insert Mode

| Key | Action                      |
| --- | --------------------------- |
| `i` | Insert before cursor        |
| `I` | Insert at beginning of line |
| `a` | Append after cursor         |
| `A` | Append at end of line       |
| `o` | Open a new line below       |
| `O` | Open a new line above       |

---

# 3. Save & Exit

| Command | Description                 |
| ------- | --------------------------- |
| `:w`    | Save file                   |
| `:q`    | Quit                        |
| `:wq`   | Save and Quit               |
| `ZZ`    | Save and Quit (Normal Mode) |
| `:q!`   | Quit without saving         |
| `:x`    | Save only if modified       |

---

# 4. Cursor Movement

## Basic Navigation

| Key | Action     |
| --- | ---------- |
| `h` | Move Left  |
| `j` | Move Down  |
| `k` | Move Up    |
| `l` | Move Right |

## Word Navigation

| Command | Action              |
| ------- | ------------------- |
| `w`     | Next word           |
| `b`     | Previous word       |
| `e`     | End of current word |

## Line Navigation

| Command | Action                    |
| ------- | ------------------------- |
| `0`     | Beginning of line         |
| `^`     | First non-space character |
| `$`     | End of line               |

## File Navigation

| Command | Action         |
| ------- | -------------- |
| `gg`    | Top of file    |
| `G`     | Bottom of file |
| `10G`   | Go to line 10  |

---

# 5. Delete

| Command | Description           |
| ------- | --------------------- |
| `x`     | Delete character      |
| `dd`    | Delete current line   |
| `2dd`   | Delete 2 lines        |
| `5dd`   | Delete 5 lines        |
| `dw`    | Delete one word       |
| `d$`    | Delete to end of line |
| `D`     | Delete to end of line |

---

# 6. Copy (Yank)

| Command | Description       |
| ------- | ----------------- |
| `yy`    | Copy current line |
| `3yy`   | Copy 3 lines      |
| `yw`    | Copy one word     |

## Paste

| Command | Description               |
| ------- | ------------------------- |
| `p`     | Paste below/after cursor  |
| `P`     | Paste above/before cursor |

---

# 7. Undo / Redo

| Command    | Description |
| ---------- | ----------- |
| `u`        | Undo        |
| `Ctrl + r` | Redo        |

---

# 8. Search

Search for text:

```vim
/nginx
```

| Command | Description    |
| ------- | -------------- |
| `n`     | Next match     |
| `N`     | Previous match |

---

# 9. Replace

Replace first occurrence in current line:

```vim
:s/old/new/
```

Replace all occurrences in current line:

```vim
:s/old/new/g
```

Replace throughout the file:

```vim
:%s/old/new/g
```

Replace with confirmation:

```vim
:%s/old/new/gc
```

---

# 10. Indentation (Useful for YAML)

| Command | Description           |
| ------- | --------------------- |
| `>>`    | Indent current line   |
| `<<`    | Unindent current line |
| `5>>`   | Indent next 5 lines   |

---

# 11. Visual Mode

| Command    | Description              |
| ---------- | ------------------------ |
| `v`        | Character-wise selection |
| `V`        | Line-wise selection      |
| `Ctrl + v` | Block (column) selection |

After selecting:

| Key | Action   |
| --- | -------- |
| `y` | Copy     |
| `d` | Delete   |
| `>` | Indent   |
| `<` | Unindent |

---

# 12. Copy Entire File

```vim
ggVGy
```

Meaning:

* `gg` → Top of file
* `V` → Visual line mode
* `G` → Select until end
* `y` → Copy

---

# 13. Delete Entire File

```vim
ggdG
```

---

# 14. Auto Indent Entire File

```vim
gg=G
```

---

# 15. Show Line Numbers

Enable:

```vim
:set number
```

Disable:

```vim
:set nonumber
```

---

# 16. Read Another File

Insert another file into the current file:

```vim
:r file.yaml
```

---

# 17. Open a File at a Specific Line

```bash
vi +25 deployment.yaml
```

Opens `deployment.yaml` at line **25**.

---

# 18. Useful CKA Workflow

Generate YAML:

```bash
kubectl create deployment nginx \
  --image=nginx \
  --dry-run=client \
  -o yaml > deploy.yaml
```

Edit YAML:

```bash
vi deploy.yaml
```

Search directly to the `spec` section:

```vim
/spec
```

---

# 19. Paste Without Breaking YAML Indentation

Enable paste mode:

```vim
:set paste
```

Paste the content.

Disable paste mode:

```vim
:set nopaste
```

---

# 20. Most Important Commands for CKA

| Command         | Purpose               |
| --------------- | --------------------- |
| `i`             | Insert                |
| `o`             | New line              |
| `Esc`           | Return to Normal mode |
| `:wq`           | Save & Quit           |
| `:q!`           | Quit without saving   |
| `yy`            | Copy line             |
| `dd`            | Delete line           |
| `p`             | Paste                 |
| `u`             | Undo                  |
| `/text`         | Search                |
| `n`             | Next match            |
| `gg`            | Top of file           |
| `G`             | Bottom of file        |
| `>>`            | Indent                |
| `<<`            | Unindent              |
| `:%s/old/new/g` | Replace all           |
| `gg=G`          | Auto-indent file      |

---

# Quick Revision

```text
i      -> Insert
Esc    -> Normal Mode
:wq    -> Save & Quit
:q!    -> Quit without Saving
yy     -> Copy Line
dd     -> Delete Line
p      -> Paste
u      -> Undo
Ctrl+r -> Redo
gg     -> Top of File
G      -> Bottom of File
0      -> Beginning of Line
$      -> End of Line
w      -> Next Word
b      -> Previous Word
>>     -> Indent
<<     -> Unindent
/text  -> Search
n      -> Next Search Result
:%s/a/b/g -> Replace All
gg=G   -> Auto Indent Entire File
```
