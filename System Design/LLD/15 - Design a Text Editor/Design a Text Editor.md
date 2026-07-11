---
title: "Design a Text Editor (Undo/Redo)"
aliases: [Design a Text Editor, Undo Redo LLD, Command Pattern Text Editor]
tags: [system-design, lld, case-study, go, command-pattern]
status: reference-quality
---

# Design a Text Editor (Undo/Redo)

> [!abstract] What you'll be able to do after this chapter
> Implement undo/redo correctly with the Command pattern, know exactly why the redo stack must be cleared on every new edit, and explain precisely why per-command delta storage beats naive whole-document Memento snapshots.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design a text editor's core edit operations — insert, delete — with undo/redo support."

## Step 2 — Requirements

Insert text at a position. Delete a range. Undo the last operation. Redo an undone operation. Support a reasonably deep undo history.

## Step 3 — The bad first draft, and why it doesn't fit the usual shape

> [!warning] This one doesn't start as an if/else chain
> Most LLD chapters in this book start with a bad draft that's an if/else or type-switch chain violating OCP. This one is different: a naive text editor just mutates a string **in place**, with **no record of what changed at all**. Undo isn't a "this code is ugly" problem here — it's a "there is no way to reverse an action, period" problem. Worth recognizing this as a distinct category of bad design, not forcing the usual narrative onto it.

```go
// The bad draft — no history at all
type Document struct {
    content string
}

func (d *Document) Insert(pos int, text string) {
    d.content = d.content[:pos] + text + d.content[pos:]
}
// Undo()? There's nothing to undo *from* — the previous state is gone.
```

## Step 4 — Refactor: the Command pattern, with undo as the primary feature

> [!tip] Same pattern as the ATM chapter, different emphasis
> [[LLD/11 - Design an ATM/Design an ATM|The ATM chapter's]] `WithdrawCommand.Undo()` existed for a **rare failure-recovery** scenario. Here, every command's `Undo()` is exercised **routinely**, as a core, everyday feature — same pattern, genuinely different purpose. Worth contrasting explicitly rather than assuming the reader re-derives it.

Each edit operation is a `Command` object with `Execute()` and `Undo()`. A `DeleteCommand` must **capture what it deleted** at execution time — that's the only way its own `Undo()` can restore it later.

```go
// FILE: command.go
package texteditor

type Command interface {
    Execute()
    Undo()
}

type InsertCommand struct {
    doc      *Document
    position int
    text     string
}

func NewInsertCommand(doc *Document, position int, text string) *InsertCommand {
    return &InsertCommand{doc: doc, position: position, text: text}
}

func (c *InsertCommand) Execute() {
    c.doc.insertAt(c.position, c.text)
}

func (c *InsertCommand) Undo() {
    c.doc.deleteRange(c.position, c.position+len(c.text))
}

type DeleteCommand struct {
    doc         *Document
    start, end  int
    deletedText string // captured on Execute — the only way Undo can restore it
}

func NewDeleteCommand(doc *Document, start, end int) *DeleteCommand {
    return &DeleteCommand{doc: doc, start: start, end: end}
}

func (c *DeleteCommand) Execute() {
    c.deletedText = c.doc.content[c.start:c.end]
    c.doc.deleteRange(c.start, c.end)
}

func (c *DeleteCommand) Undo() {
    c.doc.insertAt(c.start, c.deletedText)
}
```

```go
// FILE: document.go
package texteditor

type Document struct {
    content string
}

func NewDocument() *Document {
    return &Document{}
}

func (d *Document) insertAt(position int, text string) {
    d.content = d.content[:position] + text + d.content[position:]
}

func (d *Document) deleteRange(start, end int) {
    d.content = d.content[:start] + d.content[end:]
}

func (d *Document) Content() string {
    return d.content
}
```

```go
// FILE: editor.go
package texteditor

type Editor struct {
    doc       *Document
    undoStack []Command
    redoStack []Command
}

func NewEditor() *Editor {
    return &Editor{doc: NewDocument()}
}

// execute is the single funnel every edit goes through — which is
// also the one place the redo stack gets cleared. Skipping this
// clear is a classic undo/redo bug: without it, an old "redo" branch
// could resurrect edits that no longer make sense after a new edit
// was made post-undo.
func (e *Editor) execute(cmd Command) {
    cmd.Execute()
    e.undoStack = append(e.undoStack, cmd)
    e.redoStack = nil // a new edit invalidates any pending redo history
}

func (e *Editor) Insert(position int, text string) {
    e.execute(NewInsertCommand(e.doc, position, text))
}

func (e *Editor) Delete(start, end int) {
    e.execute(NewDeleteCommand(e.doc, start, end))
}

func (e *Editor) Undo() {
    if len(e.undoStack) == 0 {
        return
    }
    n := len(e.undoStack) - 1
    cmd := e.undoStack[n]
    e.undoStack = e.undoStack[:n]

    cmd.Undo()
    e.redoStack = append(e.redoStack, cmd)
}

func (e *Editor) Redo() {
    if len(e.redoStack) == 0 {
        return
    }
    n := len(e.redoStack) - 1
    cmd := e.redoStack[n]
    e.redoStack = e.redoStack[:n]

    cmd.Execute()
    e.undoStack = append(e.undoStack, cmd)
}

func (e *Editor) Content() string {
    return e.doc.Content()
}
```

```go
// FILE: main.go
package main

import (
    "fmt"

    texteditor "example.com/texteditor"
)

func main() {
    editor := texteditor.NewEditor()

    editor.Insert(0, "Hello")
    editor.Insert(5, " World")
    fmt.Println(editor.Content()) // "Hello World"

    editor.Undo()
    fmt.Println(editor.Content()) // "Hello"

    editor.Redo()
    fmt.Println(editor.Content()) // "Hello World"

    editor.Delete(0, 6)
    fmt.Println(editor.Content()) // "World"

    editor.Undo()
    fmt.Println(editor.Content()) // "Hello World"
}
```

---

## Step 5 — Why not Memento?

> [!question] A real alternative worth ruling out explicitly, not silently
> **Memento** would mean snapshotting the *entire document state* before/after every edit. It's correct, but wasteful — every single keystroke would copy the whole document. The Command-based approach here stores only the **delta** each command needs to reverse itself (`DeleteCommand` remembers just the small deleted substring, not the whole document). That's a real, meaningful efficiency win, and it's why real editors use incremental command-based undo rather than naive Memento snapshotting. Memento is the right tool when an object's internal state is too encapsulated/complex to reconstruct from a delta — not the case here, where the delta is trivial to capture and replay.

## Interview Q&A

> [!question]- Why does clearing the redo stack on a new edit matter so much?
> Without it, undoing three edits then making one new edit would leave two *stale* "future" edits sitting in the redo stack — redoing would silently reapply edits that were made against a document state that no longer exists after the new edit. Silent corruption, not a crash — the dangerous kind of bug.

> [!question]- How would you support undo across multiple users editing the same document (like Google Docs)?
> Genuinely harder — a naive per-user undo stack breaks the moment edits interleave, since undoing "my last edit" may no longer be a valid inverse if someone else edited the same region since. Real collaborative editors use operational transformation or CRDTs rather than a simple Command undo stack — worth naming as the next level of this problem, out of scope for a single-user LLD chapter.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[LLD/11 - Design an ATM/Design an ATM|Design an ATM]] · [[Glossary/Command Pattern|Command Pattern]]*
