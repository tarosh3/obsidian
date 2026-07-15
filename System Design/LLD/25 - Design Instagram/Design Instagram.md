---
title: "Design Instagram (LLD)"
aliases: [Design Instagram LLD, Instagram Stories LLD]
tags: [system-design, lld, case-study, go, interface-segregation]
status: reference-quality
---

# Design Instagram (LLD)

> [!abstract] What you'll be able to do after this chapter
> Use small, capability-specific interfaces (Interface Segregation) so the type system itself prevents commenting on a Story, instead of a runtime `if content.Type == "story"` check — and reuse BookMyShow's lazy-expiry discipline for a genuinely different resource (content visibility, not a seat hold).

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design the core object model for Instagram — users create posts and stories, comment on and like content, and bookmark posts for later."

## Step 2 — Requirements

**Functional:** create a post (image + caption), create a story (auto-expires after 24 hours), comment on a post, like content, bookmark/unbookmark a post.

**Non-functional:** stories must stop being visible after expiry **without a constantly-running background sweep** for correctness. Different content types support different actions — a story can be liked but **not commented on**; a post can be both.

> [!example]+ 🪜 How to build this live, step by step (interview execution order, with code)
> **Checkpoint 1 (~6 min) — `Post` alone, no `Story`, no interfaces yet.**
> ```go
> type Post struct {
>     ID       string
>     AuthorID string
>     Caption  string
>     likes    map[string]bool
> }
>
> func (p *Post) Like(userID string) { p.likes[userID] = true }
> func (p *Post) LikeCount() int      { return len(p.likes) }
> ```
> **Pattern used: none.** Get one content type fully working before introducing a second one.
>
> **Checkpoint 2 (~6 min) — bad draft, kept brief.** A single `Content` struct with a `Type string` field ("post"/"story"), and `AddComment` starting with `if content.Type == "story" { return errors.New("cannot comment on a story") }`. Narrate the pain quickly: *"every new content-specific rule is another type-check inside a method that's supposed to be generic."*
>
> **Checkpoint 3 (~10-12 min) — refactor into small, capability-specific interfaces.**
> ```go
> // Likeable and Commentable — Interface Segregation. Post implements
> // BOTH; Story implements ONLY Likeable. The type system itself
> // prevents calling AddComment on a Story — no runtime type-check.
> type Likeable interface {
>     Like(userID string)
>     LikeCount() int
> }
>
> type Commentable interface {
>     AddComment(userID, text string)
>     Comments() []Comment
> }
>
> type Comment struct {
>     UserID string
>     Text   string
> }
>
> func (p *Post) AddComment(userID, text string) {
>     p.comments = append(p.comments, Comment{UserID: userID, Text: text})
> }
> func (p *Post) Comments() []Comment { return p.comments }
> ```
> **Pattern used: Interface Segregation.** `Post` satisfies both interfaces; a `Story` type (Checkpoint 4) will only satisfy `Likeable` — calling `story.AddComment(...)` won't compile at all, a stronger guarantee than a runtime check that might be forgotten somewhere.
>
> **Checkpoint 4 (~8 min) — `Story`, with lazy expiry.**
> ```go
> type Story struct {
>     ID        string
>     AuthorID  string
>     ExpiresAt time.Time
>     likes     map[string]bool
> }
>
> func (s *Story) Like(userID string) { s.likes[userID] = true }
> func (s *Story) LikeCount() int      { return len(s.likes) }
>
> // IsExpired is checked lazily at READ time — same discipline as
> // BookMyShow's seat-hold expiry: no background sweep required for
> // correctness, just check-on-access.
> func (s *Story) IsExpired() bool {
>     return time.Now().After(s.ExpiresAt)
> }
> ```
> **Pattern used: none new** — `Story` deliberately does NOT implement `Commentable`. Say this explicitly: *"I'm not adding an empty `AddComment` that returns an error — I'm just not implementing the interface at all, so it's a compile error, not a runtime one."*
>
> **If you're short on time:** stop after Checkpoint 3. `Post` implementing both interfaces, with `Story` even just sketched as "implements Likeable only," demonstrates the actual design decision — bookmarking and lazy expiry are straightforward additions you can describe rather than fully type.

## Step 3 — The bad first draft

```go
type Content struct {
    ID       string
    AuthorID string
    Type     string // "post" or "story"
    comments []Comment
}

func (c *Content) AddComment(userID, text string) error {
    if c.Type == "story" {
        return errors.New("cannot comment on a story")
    }
    c.comments = append(c.comments, Comment{UserID: userID, Text: text})
    return nil
}
```

## Step 4 — Why it breaks

> [!bug] Every content-specific rule becomes another type-check
> A single `Content` struct with a `Type` field forces every method that has type-specific behavior to branch on that field — `AddComment` checks for "story," and a future content type (Reels? Live video, which can't be liked after it ends?) means auditing and editing every one of these methods again. This is the same Open/Closed shape as every earlier chapter, but the fix here is different: not Strategy, but **splitting into smaller interfaces so the illegal operation simply doesn't exist for that type**.

## Step 5 — Refactor, precisely

> [!tip] Runtime check vs. compile-time impossibility — a real, meaningful difference
> The bad draft's fix would normally be "add a type check." The *better* fix here is structural: `Post` implements both `Likeable` and `Commentable`; `Story` implements only `Likeable`. Code that tries to call `story.AddComment(...)` fails to **compile**, not just fails at runtime with an error someone has to remember to check. This is Interface Segregation earning its keep — small, capability-specific interfaces instead of one interface (or one struct) trying to describe every content type's full behavior.

---

## Step 6 — Complete, compilable Go implementation

```go
// FILE: content.go
package instagram

type Comment struct {
    UserID string
    Text   string
}

type Likeable interface {
    Like(userID string)
    LikeCount() int
}

type Commentable interface {
    AddComment(userID, text string)
    Comments() []Comment
}
```

```go
// FILE: post.go
package instagram

// Post implements BOTH Likeable and Commentable.
type Post struct {
    ID       string
    AuthorID string
    Caption  string
    likes    map[string]bool
    comments []Comment
}

func NewPost(id, authorID, caption string) *Post {
    return &Post{ID: id, AuthorID: authorID, Caption: caption, likes: make(map[string]bool)}
}

func (p *Post) Like(userID string) { p.likes[userID] = true }
func (p *Post) LikeCount() int      { return len(p.likes) }

func (p *Post) AddComment(userID, text string) {
    p.comments = append(p.comments, Comment{UserID: userID, Text: text})
}
func (p *Post) Comments() []Comment { return p.comments }
```

```go
// FILE: story.go
package instagram

import "time"

// Story implements ONLY Likeable — deliberately does not satisfy
// Commentable. Calling AddComment on a Story is a compile error, not
// a runtime check that could be forgotten somewhere.
type Story struct {
    ID        string
    AuthorID  string
    ExpiresAt time.Time
    likes     map[string]bool
}

func NewStory(id, authorID string, ttl time.Duration) *Story {
    return &Story{ID: id, AuthorID: authorID, ExpiresAt: time.Now().Add(ttl), likes: make(map[string]bool)}
}

func (s *Story) Like(userID string) { s.likes[userID] = true }
func (s *Story) LikeCount() int      { return len(s.likes) }

// IsExpired — checked lazily on access, same discipline as
// BookMyShow's seat-hold expiry (LLD/06), applied here to content
// visibility instead of a resource lock.
func (s *Story) IsExpired() bool {
    return time.Now().After(s.ExpiresAt)
}
```

```go
// FILE: story_feed.go
package instagram

type StoryFeed struct {
    stories []*Story
}

func (f *StoryFeed) Add(s *Story) {
    f.stories = append(f.stories, s)
}

// ActiveStories filters lazily at read time — no background sweep
// needed to "clean up" expired stories for correctness.
func (f *StoryFeed) ActiveStories() []*Story {
    var active []*Story
    for _, s := range f.stories {
        if !s.IsExpired() {
            active = append(active, s)
        }
    }
    return active
}
```

```go
// FILE: bookmarks.go
package instagram

import "sync"

type BookmarkService struct {
    mu        sync.Mutex
    bookmarks map[string]map[string]bool // bookmarks[userID][postID]
}

func NewBookmarkService() *BookmarkService {
    return &BookmarkService{bookmarks: make(map[string]map[string]bool)}
}

func (b *BookmarkService) Add(userID, postID string) {
    b.mu.Lock()
    defer b.mu.Unlock()
    if b.bookmarks[userID] == nil {
        b.bookmarks[userID] = make(map[string]bool)
    }
    b.bookmarks[userID][postID] = true
}

func (b *BookmarkService) Remove(userID, postID string) {
    b.mu.Lock()
    defer b.mu.Unlock()
    delete(b.bookmarks[userID], postID)
}

func (b *BookmarkService) List(userID string) []string {
    b.mu.Lock()
    defer b.mu.Unlock()
    var result []string
    for postID := range b.bookmarks[userID] {
        result = append(result, postID)
    }
    return result
}
```

```go
// FILE: main.go  (adjust import path to your module name)
package main

import (
    "fmt"
    "time"

    instagram "example.com/instagram"
)

func main() {
    post := instagram.NewPost("p1", "alice", "sunset photo")
    post.Like("bob")
    post.AddComment("bob", "beautiful!")
    fmt.Println("Post likes:", post.LikeCount(), "comments:", len(post.Comments()))

    story := instagram.NewStory("s1", "alice", 24*time.Hour)
    story.Like("carol")
    // story.AddComment("carol", "nice") // would NOT compile — Story has no AddComment method

    feed := &instagram.StoryFeed{}
    feed.Add(story)
    fmt.Println("Active stories right now:", len(feed.ActiveStories())) // 1

    bookmarks := instagram.NewBookmarkService()
    bookmarks.Add("dave", post.ID)
    fmt.Println("Dave's bookmarks:", bookmarks.List("dave"))
}
```

---

## Interview Q&A

> [!question]- Why not just give `Story` an `AddComment` method that always returns an error?
> That's the runtime-check approach — it compiles, but every caller has to remember to check the error, and nothing stops a future refactor from silently dropping that check. Not implementing `Commentable` at all makes the illegal call a **compile error**, catchable before the code ever runs — a strictly stronger guarantee for the same intent.

> [!question]- How would you add Reels (short videos, likeable and commentable, but with a view count Posts don't have)?
> A new `Reel` struct implementing both `Likeable` and `Commentable` (same as `Post`), plus its own `RecordView`/`ViewCount` methods that no interface forces on it — Reels simply has one more capability than Posts, expressed as one more method, not a shared interface bloated with a field only Reels needs.

> [!question]- Why lazy expiry for stories instead of a background job that deletes expired ones?
> A background sweep adds operational complexity (a job that must run reliably) for a property that's just as correctly enforced by checking `IsExpired()` at read time — the story data can even be cleaned up later, in bulk, for storage hygiene, without that cleanup being load-bearing for *correctness* the way it would be if reads trusted an un-swept list.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[LLD/06 - Design BookMyShow - Seat Booking/Design BookMyShow - Seat Booking|Design BookMyShow]] (lazy expiry) · [[CS Fundamentals/10 - Design Principles/01 - SOLID Principles|SOLID Principles]]*
