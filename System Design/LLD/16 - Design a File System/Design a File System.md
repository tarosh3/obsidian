---
title: "Design a File System"
aliases: [Design a File System LLD, Composite Pattern File System]
tags: [system-design, lld, case-study, go, composite-pattern]
status: reference-quality
---

# Design a File System

> [!abstract] What you'll be able to do after this chapter
> Recognize the canonical Composite pattern use case on sight — a tree where leaves and containers must be treated uniformly — and implement it so recursive aggregation (like total size) needs zero type-checking.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design an in-memory file system supporting files and directories, navigation, and computing aggregate size."

## Step 2 — Requirements

Create files/directories in a hierarchy. Navigate the tree. Compute the total size of a directory, recursively including all nested files. List contents.

## Step 3 — The bad first draft

Separate `File` and `Directory` types with **no common interface**. Any code that needs to "process this node, whatever it is" — computing total size, for instance — needs explicit type-checking at every call site:

```go
// The bad draft — type-branching wherever the tree is touched
func computeSize(node interface{}) int64 {
    switch n := node.(type) {
    case *File:
        return n.size
    case *Directory:
        var total int64
        for _, child := range n.children {
            total += computeSize(child) // recurse, but only because we hand-wrote this branch
        }
        return total
    default:
        panic("unknown node type")
    }
}
```

This is the familiar type-switch smell from earlier chapters, but applied to a genuinely **tree-shaped, recursive** structure rather than a flat set of cases — every new node type (symlinks? mounted volumes?) means touching this function again.

## Step 4 — Refactor: the Composite pattern

> [!tip] The canonical textbook Composite use case
> A single `FileSystemNode` interface, implemented by **both** the leaf (`File`) and the composite (`Directory`). Calling code treats files and directories uniformly through the same interface — a directory's `Size()` simply sums its children's `Size()` calls recursively, and each child correctly contributes its own size regardless of whether it's a file or a nested directory. `Directory` never needs to know or check what kind of node a child is.

```go
// FILE: node.go
package filesystem

// FileSystemNode is the Composite abstraction — both File (leaf) and
// Directory (composite) implement it, letting calling code treat
// both uniformly without type-checking which one it has.
type FileSystemNode interface {
    Name() string
    Size() int64
}
```

```go
// FILE: file.go
package filesystem

type File struct {
    name string
    size int64
}

func NewFile(name string, size int64) *File {
    return &File{name: name, size: size}
}

func (f *File) Name() string { return f.name }
func (f *File) Size() int64  { return f.size }
```

```go
// FILE: directory.go
package filesystem

type Directory struct {
    name     string
    children []FileSystemNode
}

func NewDirectory(name string) *Directory {
    return &Directory{name: name}
}

func (d *Directory) Name() string { return d.name }

// Size recurses through every child — a File contributes its own
// size directly; a nested Directory recurses into ITS OWN Size(),
// which recurses further, all the way down. Directory never checks
// whether a given child is a File or another Directory — it just
// calls the shared interface method.
func (d *Directory) Size() int64 {
    var total int64
    for _, child := range d.children {
        total += child.Size()
    }
    return total
}

func (d *Directory) Add(node FileSystemNode) {
    d.children = append(d.children, node)
}

func (d *Directory) Children() []FileSystemNode {
    return d.children
}

// Find searches this directory's IMMEDIATE children by name — a
// simple building block. A full path-based lookup (e.g. "/a/b/c")
// would walk this method repeatedly, one path segment at a time.
func (d *Directory) Find(name string) (FileSystemNode, bool) {
    for _, child := range d.children {
        if child.Name() == name {
            return child, true
        }
    }
    return nil, false
}
```

```go
// FILE: main.go
package main

import (
    "fmt"

    filesystem "example.com/filesystem"
)

func main() {
    root := filesystem.NewDirectory("root")

    docs := filesystem.NewDirectory("docs")
    docs.Add(filesystem.NewFile("resume.pdf", 2048))
    docs.Add(filesystem.NewFile("cover_letter.pdf", 1024))

    photos := filesystem.NewDirectory("photos")
    photos.Add(filesystem.NewFile("vacation.jpg", 5_000_000))

    root.Add(docs)
    root.Add(photos)
    root.Add(filesystem.NewFile("readme.txt", 100))

    fmt.Println("Total size of root:", root.Size(), "bytes") // 5,003,172
    fmt.Println("Total size of docs:", docs.Size(), "bytes")  // 3,072

    if node, ok := root.Find("docs"); ok {
        fmt.Println("Found:", node.Name())
    }
}
```

---

## Step 5 — Why Composite, and where it stops being enough

> [!info] The one-line test for "is this Composite?"
> If your tree has **two or more node kinds** and calling code needs to treat them **the same way** for at least one operation (here: `Size()`), that's Composite. The moment operations start needing to know "am I a file or a directory" internally to behave differently is fine — that's normal polymorphism — the pattern is specifically about the *caller* never needing to branch.

**Where this simple model runs out:** symbolic links (a node that points to another node — recursion could cycle forever without cycle detection), permissions (each node needing an owner/ACL, likely via a Decorator wrapping any `FileSystemNode`), and mounted volumes (a `Directory` whose children actually live on a different physical device). Worth naming these as the natural follow-up questions this design invites, even without full implementations here.

## Interview Q&A

> [!question]- Why not just use a type switch like the bad draft — it's shorter?
> It's shorter for *one* operation. Every new operation on the tree (rename, delete, count-files-of-type, search) needs its own type switch repeated at every call site, and every new node type means revisiting every one of those switches. Composite pays a small upfront interface cost to make every future operation and every future node type additive, not invasive.

> [!question]- How would you support symlinks without infinite recursion in Size()?
> Track visited node identities during a single traversal (a `map` keyed by node pointer/ID) and skip a node already seen in the current call chain — the same cycle-detection idea used in graph traversal generally, applied here to a filesystem tree that's no longer strictly a tree once symlinks exist.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[LLD/12 - Design a Library Management System/Design a Library Management System|Design a Library Management System]] · [[Glossary/Composite Pattern|Composite Pattern]]*
