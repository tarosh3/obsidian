---
title: "Design Twitter (LLD)"
aliases: [Design Twitter LLD, Twitter Timeline Merge]
tags: [system-design, lld, case-study, go, algorithms, heap]
status: reference-quality
---

# Design Twitter (LLD)

> [!abstract] What you'll be able to do after this chapter
> Recognize timeline generation as a K-way merge problem (the exact shape of "Merge K Sorted Lists"), implement it correctly with a heap, and know precisely why that's an algorithmic fix, not a design-pattern one.

> [!info] Distinct from the HLD version
> [[HLD/06 - Design Twitter - News Feed/Design Twitter - News Feed|The HLD chapter]] covers fan-out-on-write vs. fan-out-on-read at distributed scale, and the celebrity-follower hybrid fix. This chapter is the **in-process object model** underneath a fan-out-on-read implementation — the actual algorithm that computes a merged, chronological timeline on demand.

---

## Step 1 — The interview question

> [!question] As an interviewer would ask it
> "Design the core object model for Twitter — users post tweets, follow other users, like/retweet, and view a chronological home timeline of tweets from everyone they follow."

## Step 2 — Requirements

**Functional:** post a tweet, follow/unfollow a user, like a tweet, retweet a tweet, `GetTimeline(userID)` — a chronological feed merging tweets from everyone the user follows.

**Non-functional:** `GetTimeline` is called far more often than tweets are posted — it needs to stay fast as a user follows more people, without recomputing everything from scratch on every call. Like/retweet must be **idempotent** — a user liking the same tweet twice shouldn't double-count.

> [!example]+ 🪜 How to build this live, step by step (interview execution order, with code)
> **Checkpoint 1 (~8 min) — post + follow, no timeline yet.**
> ```go
> type Tweet struct {
>     ID        string
>     AuthorID  string
>     Text      string
>     CreatedAt time.Time
> }
>
> type TwitterService struct {
>     tweetsByUser map[string][]*Tweet // newest-first, append-only per user
>     follows      map[string][]string // follows[userID] = who they follow
> }
>
> func (s *TwitterService) PostTweet(userID, text string) *Tweet {
>     t := &Tweet{ID: generateID(), AuthorID: userID, Text: text, CreatedAt: time.Now()}
>     s.tweetsByUser[userID] = append([]*Tweet{t}, s.tweetsByUser[userID]...) // prepend — newest first
>     return t
> }
>
> func (s *TwitterService) Follow(userID, targetID string) {
>     s.follows[userID] = append(s.follows[userID], targetID)
> }
> ```
> **Pattern used: none.** Get posting and following working, and confirm each user's own tweet list stays newest-first — that ordering is what the next checkpoints depend on.
>
> **Checkpoint 2 (~8 min) — naive timeline, deliberately.**
> ```go
> func (s *TwitterService) GetTimeline(userID string) []*Tweet {
>     var all []*Tweet
>     for _, followeeID := range s.follows[userID] {
>         all = append(all, s.tweetsByUser[followeeID]...)
>     }
>     sort.Slice(all, func(i, j int) bool {
>         return all[i].CreatedAt.After(all[j].CreatedAt)
>     })
>     return all
> }
> ```
> **Pattern used: none — write it this way on purpose.** Narrate: *"this collects EVERY tweet from EVERY followee and sorts the whole thing on every single call — it's not using the fact each followee's list is already sorted."*
>
> **Checkpoint 3 (~12-15 min) — refactor into a heap-based K-way merge. This is the actual centerpiece.**
> ```go
> // feedHeapItem tracks one followee's "current position" in their
> // own (already-sorted) tweet list.
> type feedHeapItem struct {
>     tweet  *Tweet
>     userID string
>     idx    int
> }
>
> type feedHeap []*feedHeapItem
> func (h feedHeap) Len() int           { return len(h) }
> func (h feedHeap) Less(i, j int) bool { return h[i].tweet.CreatedAt.After(h[j].tweet.CreatedAt) }
> func (h feedHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
> func (h *feedHeap) Push(x interface{}) { *h = append(*h, x.(*feedHeapItem)) }
> func (h *feedHeap) Pop() interface{} {
>     old := *h
>     item := old[len(old)-1]
>     *h = old[:len(old)-1]
>     return item
> }
>
> func (s *TwitterService) GetTimeline(userID string, limit int) []*Tweet {
>     h := &feedHeap{}
>     heap.Init(h)
>     for _, followeeID := range s.follows[userID] {
>         if tweets := s.tweetsByUser[followeeID]; len(tweets) > 0 {
>             heap.Push(h, &feedHeapItem{tweet: tweets[0], userID: followeeID, idx: 0})
>         }
>     }
>
>     var result []*Tweet
>     for h.Len() > 0 && len(result) < limit {
>         item := heap.Pop(h).(*feedHeapItem)
>         result = append(result, item.tweet)
>
>         followeeTweets := s.tweetsByUser[item.userID]
>         if nextIdx := item.idx + 1; nextIdx < len(followeeTweets) {
>             heap.Push(h, &feedHeapItem{tweet: followeeTweets[nextIdx], userID: item.userID, idx: nextIdx})
>         }
>     }
>     return result
> }
> ```
> **Pattern used: none — this is the "Merge K Sorted Lists" algorithm, not a GoF design pattern.** Say this explicitly if asked. Each followee's tweet list is already sorted; a max-heap of "next candidate from each list" pops the globally-newest tweet in `O(log F)` per pop (F = number of followees) instead of sorting everything. Critically, it also **stops at `limit`** — Checkpoint 2's version processes every tweet from every followee even when only 20 are needed.
>
> **Checkpoint 4 (remaining time, or if asked) — idempotent like.**
> ```go
> type LikeService struct {
>     mu    sync.Mutex
>     likes map[string]map[string]bool // likes[tweetID][userID]
> }
>
> func (s *LikeService) Like(tweetID, userID string) bool {
>     s.mu.Lock()
>     defer s.mu.Unlock()
>     if s.likes[tweetID] == nil {
>         s.likes[tweetID] = make(map[string]bool)
>     }
>     if s.likes[tweetID][userID] {
>         return false // already liked — idempotent no-op
>     }
>     s.likes[tweetID][userID] = true
>     return true
> }
> ```
> No new pattern — the same "atomic claim, keyed by (resource, actor)" shape used for likes/retweets/bookmarks throughout this book. `Retweet` is the identical shape again.
>
> **If you're short on time:** stop after Checkpoint 3. The heap-based merge, correctly implemented and capped at `limit`, is the actual substance of this question — like/retweet are straightforward map operations you can describe rather than fully type.

---

## Step 3 — Why the naive version breaks, precisely

> [!bug] Two separate problems, not one
> **Cost:** `GetTimeline` in Checkpoint 2 is `O(N log N)` per call, where `N` is the *total* tweet count across every followee — recomputed from scratch on every single read, even though each followee's own list never needed re-sorting (it was already sorted). **Waste:** it builds the *entire* combined list before sorting, even when the caller only wants the first 20 tweets.

## Step 4 — The fix, precisely

> [!tip] This is an algorithm problem wearing a system-design costume
> Each followee's tweets are already chronologically sorted (append-only, newest-first). Merging several already-sorted lists into one combined sorted output is exactly the classic **Merge K Sorted Lists** problem — solved with a heap holding one "current candidate" per list, popping the global best and advancing that one list's pointer. The complexity drops to `O((F + limit) log F)`, and the `limit` cap means the algorithm does no more work than the caller actually needs.

## Step 5 — Where this connects back to the HLD chapter

> [!info] This is the mechanism a fan-out-on-read design needs
> [[HLD/06 - Design Twitter - News Feed/Design Twitter - News Feed|The HLD chapter's]] fan-out-on-read approach (compute a feed on demand instead of precomputing it on every post) needs *exactly* this merge algorithm to actually be efficient — without it, "compute on demand" would mean the expensive full-sort from Checkpoint 2, at real scale. The heap-based merge is what makes fan-out-on-read viable at all, not just a theoretical option.

## Step 6 — Complete, compilable Go implementation

```go
// FILE: tweet.go
package twitter

import "time"

type Tweet struct {
    ID        string
    AuthorID  string
    Text      string
    CreatedAt time.Time
}
```

```go
// FILE: service.go
package twitter

import (
    "container/heap"
    "fmt"
    "sort"
    "sync"
    "time"
)

type TwitterService struct {
    mu           sync.Mutex
    tweetsByUser map[string][]*Tweet // newest-first, append-only per user
    follows      map[string]map[string]bool
    nextID       int
}

func NewTwitterService() *TwitterService {
    return &TwitterService{
        tweetsByUser: make(map[string][]*Tweet),
        follows:      make(map[string]map[string]bool),
    }
}

func (s *TwitterService) PostTweet(userID, text string) *Tweet {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.nextID++
    t := &Tweet{ID: fmt.Sprintf("t-%d", s.nextID), AuthorID: userID, Text: text, CreatedAt: time.Now()}
    s.tweetsByUser[userID] = append([]*Tweet{t}, s.tweetsByUser[userID]...)
    return t
}

func (s *TwitterService) Follow(userID, targetID string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    if s.follows[userID] == nil {
        s.follows[userID] = make(map[string]bool)
    }
    s.follows[userID][targetID] = true
}

type feedHeapItem struct {
    tweet  *Tweet
    userID string
    idx    int
}

type feedHeap []*feedHeapItem

func (h feedHeap) Len() int            { return len(h) }
func (h feedHeap) Less(i, j int) bool  { return h[i].tweet.CreatedAt.After(h[j].tweet.CreatedAt) }
func (h feedHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *feedHeap) Push(x interface{}) { *h = append(*h, x.(*feedHeapItem)) }
func (h *feedHeap) Pop() interface{} {
    old := *h
    n := len(old)
    item := old[n-1]
    *h = old[:n-1]
    return item
}

// GetTimeline merges every followee's already-sorted tweet list via
// a heap — see Step 3/4 for why this beats sorting everything fresh
// on every call.
func (s *TwitterService) GetTimeline(userID string, limit int) []*Tweet {
    s.mu.Lock()
    defer s.mu.Unlock()

    h := &feedHeap{}
    heap.Init(h)
    for followeeID := range s.follows[userID] {
        if tweets := s.tweetsByUser[followeeID]; len(tweets) > 0 {
            heap.Push(h, &feedHeapItem{tweet: tweets[0], userID: followeeID, idx: 0})
        }
    }

    var result []*Tweet
    for h.Len() > 0 && len(result) < limit {
        item := heap.Pop(h).(*feedHeapItem)
        result = append(result, item.tweet)

        followeeTweets := s.tweetsByUser[item.userID]
        if nextIdx := item.idx + 1; nextIdx < len(followeeTweets) {
            heap.Push(h, &feedHeapItem{tweet: followeeTweets[nextIdx], userID: item.userID, idx: nextIdx})
        }
    }
    return result
}

// GetTimelineNaive kept for the Step 3 trace/comparison — never used
// in production once GetTimeline exists.
func (s *TwitterService) GetTimelineNaive(userID string) []*Tweet {
    s.mu.Lock()
    defer s.mu.Unlock()
    var all []*Tweet
    for followeeID := range s.follows[userID] {
        all = append(all, s.tweetsByUser[followeeID]...)
    }
    sort.Slice(all, func(i, j int) bool { return all[i].CreatedAt.After(all[j].CreatedAt) })
    return all
}
```

```go
// FILE: like_service.go
package twitter

import "sync"

// LikeService and a RetweetService share the identical shape — an
// atomic, idempotent claim keyed by (resource, actor), the same
// pattern already proven in the BookMyShow chapter.
type LikeService struct {
    mu    sync.Mutex
    likes map[string]map[string]bool // likes[tweetID][userID]
}

func NewLikeService() *LikeService {
    return &LikeService{likes: make(map[string]map[string]bool)}
}

func (s *LikeService) Like(tweetID, userID string) bool {
    s.mu.Lock()
    defer s.mu.Unlock()
    if s.likes[tweetID] == nil {
        s.likes[tweetID] = make(map[string]bool)
    }
    if s.likes[tweetID][userID] {
        return false
    }
    s.likes[tweetID][userID] = true
    return true
}

func (s *LikeService) LikeCount(tweetID string) int {
    s.mu.Lock()
    defer s.mu.Unlock()
    return len(s.likes[tweetID])
}
```

```go
// FILE: main.go  (adjust import path to your module name)
package main

import (
    "fmt"

    twitter "example.com/twitter"
)

func main() {
    service := twitter.NewTwitterService()
    likes := twitter.NewLikeService()

    service.PostTweet("alice", "hello world")
    service.PostTweet("bob", "go is fun")
    aliceTweet2 := service.PostTweet("alice", "second tweet")

    service.Follow("carol", "alice")
    service.Follow("carol", "bob")

    timeline := service.GetTimeline("carol", 10)
    fmt.Println("Carol's timeline (newest first):")
    for _, t := range timeline {
        fmt.Printf("  [%s] %s: %s\n", t.CreatedAt.Format("15:04:05"), t.AuthorID, t.Text)
    }

    fmt.Println("First like:", likes.Like(aliceTweet2.ID, "carol"))  // true
    fmt.Println("Duplicate like:", likes.Like(aliceTweet2.ID, "carol")) // false — idempotent
    fmt.Println("Like count:", likes.LikeCount(aliceTweet2.ID))
}
```

---

## Interview Q&A

> [!question]- Why not just keep one single globally-sorted list of all tweets and filter by followee at read time?
> That flips the cost the wrong way — filtering a global list by "is this author someone I follow" is `O(total tweets in the system)` per read, far worse than merging just this user's followees' lists. Partitioning by author (each user's own append-only list) is what makes the heap-merge approach only pay for the followees that actually matter to *this* reader.

> [!question]- What happens if a user follows 10,000 people?
> The heap holds at most `F` items at any time (one per followee with remaining tweets) — `F = 10,000` means `O(log 10,000) ≈ 13` per heap operation, still fast. The real cost driver becomes fetching each followee's latest tweet initially, which is a separate, genuine scaling concern the HLD chapter's caching/precomputation strategies exist to address — worth naming as the natural escalation path rather than claiming this LLD version scales unboundedly.

> [!question]- How would you support retweets appearing in the timeline correctly, ordered by retweet time, not original post time?
> Model a retweet as its own timeline entry (referencing the original tweet's content but carrying its own `CreatedAt` = retweet time), stored in the retweeting user's own `tweetsByUser` list — the heap-merge algorithm above needs zero changes, since it just merges "each user's chronological entries," agnostic to whether an entry is an original post or a retweet.

---
*Related: [[00 - Start Here/How This Handbook Works|Book Map]] · [[HLD/06 - Design Twitter - News Feed/Design Twitter - News Feed|HLD version]] · [[LLD/06 - Design BookMyShow - Seat Booking/Design BookMyShow - Seat Booking|Design BookMyShow]] (idempotent claim pattern)*
