---
title: "211. Design Add and Search Words Data Structure"
aliases: [Design Add and Search Words Data Structure, Add and Search Word, LeetCode 211]
tags: [dsa, leetcode, trie, dfs, backtracking, wildcard, design, medium]
difficulty: Medium
leetcode: 211
---

# 211. Design Add and Search Words Data Structure
`Medium` · **Pattern:** Trie + DFS to branch on the `.` wildcard

> [!question] Problem
> Design a data structure that supports adding new words and finding if a string matches any previously added string.
>
> Implement the `WordDictionary` class:
> - `WordDictionary()` initializes the object.
> - `void addWord(word)` adds `word` to the data structure.
> - `bool search(word)` returns `true` if there is any string in the data structure that matches `word`. `word` may contain dots `'.'` where a dot **can match any letter**.
>
> **Example:**
> ```
> addWord("bad"); addWord("dad"); addWord("mad");
> search("pad");   // false
> search("bad");   // true
> search(".ad");   // true
> search("b..");   // true
> ```
>
> **Constraints:**
> - `1 <= word.length <= 25`
> - `word` in `addWord` consists of lowercase letters.
> - `word` in `search` may contain up to `2` dots; other chars are lowercase.
> - At most `10^4` calls to `addWord` and `search`.

---

## 🧩 Pattern this follows

> [!tip] Same trie as #208 — the twist is the `.` forcing a branch
> `addWord` is plain trie insertion. `search` differs: at a **normal letter**, follow that one child (fail if missing). At a **`.`**, it could be *any* letter, so you must try **every non-null child** and succeed if *any* subtree matches the rest of the word — that's a **DFS / backtracking** fan-out. When the word index reaches the end, success = the current node is `isTerminal`.

### 🖼️ Visualizing it

`search(".ad")` — the dot forks into every first-letter child; each explores `ad` under it.

```mermaid
graph TD
    S["dfs(root, \".ad\", 0)"] -->|"'.' → try all children"| B["child 'b' → dfs(\"ad\",1)"]
    S -->|try all| D["child 'd' → dfs(\"ad\",1)"]
    S -->|try all| M["child 'm' → dfs(\"ad\",1)"]
    B --> B2["'a'→'d'→ isTerminal? ✔ return true"]
    D --> D2["'a'→'d'→ isTerminal? ✔"]
    M --> M2["'a'→'d'→ isTerminal? ✔"]
```

## 💻 My Solution (C++)

```cpp
class TrieNode{
    public:
    char data;
    vector<TrieNode*> children;
    bool isTerminal;

    TrieNode(char ch){
        data=ch;
        children.resize(26,nullptr);
        isTerminal=false;
    }

};

class WordDictionary {
public:

    TrieNode* head;
    WordDictionary() {
        head=new TrieNode('\0');
    }
    
    void addWord(string word) {
        
        TrieNode* curr=head;


        for(char c:word){
            int index=c-'a';
            if(curr->children[index]==nullptr){
                TrieNode* newNode=new TrieNode(c);
                curr->children[index]=newNode;
            }
            curr=curr->children[index];
        }

        curr->isTerminal=true;

    }

    bool dfs(TrieNode* node, string& word, int index){
        if(index==word.size()){
            return node->isTerminal;
        }

        if(word[index]!='.'){
            int childIndex=word[index]-'a';

            if(node->children[childIndex]==nullptr){
                return false;
            }
            return dfs(node->children[childIndex],word,index+1);

        }

        for(int i=0;i<26;i++){
            if(node->children[i]!=nullptr){
                if(dfs(node->children[i],word,index+1)) {
                    return true;
                }
            }
        }

        return false;
    }
    
    bool search(string word) {
        return dfs(head,word,0);
    }
};

/**
 * Your WordDictionary object will be instantiated and called as such:
 * WordDictionary* obj = new WordDictionary();
 * obj->addWord(word);
 * bool param_2 = obj->search(word);
 */
```

## 🔍 Walkthrough

1. **`addWord`** — identical to [[Implement Trie (LeetCode #208)]] insert: walk/create nodes letter by letter, mark the final node `isTerminal`.
2. **`search`** delegates to `dfs(head, word, 0)`.
3. **`dfs(node, word, index)`**:
   - **Base case:** `index == word.size()` → the whole pattern consumed → return `node->isTerminal` (a real word must end exactly here).
   - **Normal letter** (`word[index] != '.'`): one deterministic path. If that child is `nullptr` → `false`; else recurse into it at `index+1`.
   - **Wildcard `.`**: loop all 26 slots; for each existing child, recurse. Return `true` on the **first** subtree that matches; if none match, `false`.
4. Backtracking is implicit — the recursion tries a child, and on failure the loop simply moves to the next child.

## ⏱️ Complexity

| | Complexity | Why |
|---|---|---|
| **`addWord`** | O(L) | One step per character |
| **`search`** | O(L) with no dots; up to O(26^d · L) worst | Each `.` forks up to 26 ways; `d` = number of dots (capped at 2 here, so cheap in practice) |
| **Space** | O(total chars added) + O(L) recursion | Trie storage + DFS stack depth `L` |

## 🚀 Tricks & Similar Problems

> [!success] Only the `.` needs DFS — plain letters stay a straight walk
> Don't fan out on every character; branch **only** on `.`. Returning `true` on the first matching child short-circuits the rest of the loop, so best case stays fast. Because dots are capped at 2, the `26^d` blow-up is bounded and harmless here.
> **Similar pattern:** [[Implement Trie (LeetCode #208)]] (the base structure), [[Word Search II (LeetCode #212)]] (trie-guided DFS, but over a grid), [[Trie — Fundamentals & Full Implementation]].
