# binary tree right side view: BFS and DFS solutions explained, with the edge cases that trip people up in interviews

The problem sounds almost too easy: stand on the right side of a binary tree, look left, list the nodes you can see from top to bottom. That's LeetCode 199, rated Medium, and the difficulty has nothing to do with writing a queue. It's that the intuitive mental model — "walk down the right spine and print whatever is there" — is wrong in a way the test cases are designed to catch.

The rule that actually holds: at every level, you want the last node in a left-to-right ordering of that level. Not the right child of the previous visible node. Not the deepest node on the right branch. Just the rightmost node at each depth.

## Two examples that break the obvious approach

Take `root = [1,2,3,null,5,null,4]`. The tree looks like this:


        1          <- [1]
      /   \
     2     3       <- [2, 3]
      \     \
       5     4     <- [5, 4]


Read it level by level: `[1]`, `[2,3]`, `[5,4]`. Take the last element of each row and you get `[1,3,4]`. Node 5 doesn't appear even though it's the only thing below node 2 — it sits at the same depth as 4, and 4 is to its right.

Now flip it: `root = [1,2,3,4,null,null,null,5]`.


        1          <- [1]
      /   \
     2     3       <- [2, 3]
    /
   4               <- [4]
  /
 5                 <- [5]


Output: `[1,3,4,5]`. Both 4 and 5 are left children, and both are visible, because there is nothing on the right side of the tree to occlude them. Anyone who committed to "follow the right children" stops after `[1,3]` and never recovers.

The constraints are forgiving: between 0 and 100 nodes, values in the range -100 to 100. Small enough that stack depth is never a practical issue. That's a hint about what the interviewer is actually testing — not memory management, but whether you can restate the problem correctly before coding.

## Solution 1: BFS with level-order traversal

This is the version most people write, and it maps almost directly onto the definition of the answer. Process the tree one level at a time, and record the last node of each level.

python
from collections import deque
from typing import List, Optional

class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

class Solution:
    def rightSideView(self, root: Optional[TreeNode]) -> List[int]:
        if not root:
            return []

        res = []
        queue = deque([root])

        while queue:
            level_size = len(queue)          # snapshot BEFORE the inner loop
            for i in range(level_size):
                node = queue.popleft()
                if i == level_size - 1:      # last node of this level
                    res.append(node.val)
                if node.left:
                    queue.append(node.left)
                if node.right:
                    queue.append(node.right)

        return res


The mechanics are worth saying out loud, because this is where most bugs live. `level_size` freezes the boundary of the current level before you start pushing children. Skip that snapshot and children leak into the row you're still processing, so "last node of the level" stops meaning anything.

There's a second variant that avoids the index entirely: enqueue right children before left children, then the front of the queue is always the rightmost node of the current level.

python
while queue:
    res.append(queue[0].val)                 # rightmost node of this level
    for _ in range(len(queue)):
        node = queue.popleft()
        if node.right:
            queue.append(node.right)
        if node.left:
            queue.append(node.left)


Both are correct. The index version is easier to reason about when you're nervous; the second is shorter and reads nicely once you trust it. Both run in **O(n) time**, since every node enters and leaves the queue exactly once. Space is **O(w)**, where w is the widest level of the tree — up to about n/2 for a perfect tree.

## Solution 2: DFS, right child first, with a depth counter

The DFS solution is where the problem gets interesting, because it inverts the reasoning. Instead of visiting level by level, you do a preorder traversal that always goes right before left. The first node you touch at any new depth is, by construction, the rightmost node at that depth.

python
class Solution:
    def rightSideView(self, root: Optional[TreeNode]) -> List[int]:
        res = []

        def dfs(node: Optional[TreeNode], depth: int) -> None:
            if not node:
                return
            if depth == len(res):        # first time we reach this depth
                res.append(node.val)
            dfs(node.right, depth + 1)
            dfs(node.left, depth + 1)

        dfs(root, 0)
        return res


The `depth == len(res)` check is the clever part. The result list grows by exactly one entry per level, so its length is always the depth you have not recorded yet. The first node to arrive at that depth claims the slot; every later node at the same depth finds the condition false and is ignored.

Why the equality rather than `>=`? Because the traversal never skips a depth — a node at depth d can only have children at d+1 — so `len(res)` can never overshoot. If you find yourself writing `>=` "just to be safe," check whether you started the root at depth 0. Mixing a 1-based depth with a 0-based list length silently drops the root from the answer, which is the single most common bug in this solution.

Complexity: **O(n) time**, **O(h) space** for the recursion stack, where h is the tree height. On a balanced tree that's O(log n), which is where DFS beats the queue version on memory. On a completely skewed tree it degrades to O(n), which is where BFS wins instead.

One line worth remembering: swap the recursion order to `dfs(node.left, ...)` before `dfs(node.right, ...)` and you get the left side view. Same code, mirrored output.

## Which solution should you actually write?

|  | BFS (level-order) | DFS (right-first preorder) |
| --- | --- | --- |
| Lines of code | ~15 | ~10 |
| Time | O(n) | O(n) |
| Extra space | O(w), widest level | O(h), tree height |
| Memory edge case | Heavy on very wide trees | Recursion depth on skewed trees |
| Follow-up flexibility | High — same skeleton for zigzag, level averages, left view | Lower — needs restructuring |
| Typical bug | Forgetting to snapshot level size | Off-by-one on depth |

If the interviewer is likely to extend the question — level averages, zigzag ordering, the left side view — write the BFS version. The skeleton survives all of those with small edits. If they ask for something without a queue, or you want the shortest correct answer on the board, DFS is hard to beat.

## The five bugs that show up most often

1. **Following only right children.** Fails on `[1,2,3,4,null,null,null,5]`, where two visible nodes are left descendants. The rule is per-level, not per-branch.
2. **Reading `len(queue)` inside the inner loop.** Levels blend together and "rightmost" becomes meaningless. Snapshot it first.
3. **Off-by-one on the DFS depth.** `depth == len(res)` requires the root to start at depth 0.
4. **Skipping the empty tree.** `root = []` must return `[]`, not crash or return `[None]`.
5. **Taking the wrong end of the queue.** In the right-child-first variant, the visible node is at the front of the queue, not the back.

text
root = [1,2,3,null,5,null,4]        -> [1,3,4]
root = [1,2,3,4,null,null,null,5]   -> [1,3,4,5]
root = [1,null,3]                   -> [1,3]
root = []                           -> []


Everything above is knowledge you can reconstruct from the definition. The part that actually decides your interview outcome is speed: recognizing within the first two minutes that this is a level-order problem wearing a disguise, and that the "rightmost visible node" is just an index into each level.

## This problem is a pattern, not a problem

Right side view sits in the middle of a cluster of questions that share one skeleton:

- 102 — Binary Tree Level Order Traversal
- 103 — Binary Tree Zigzag Level Order Traversal
- 199 — Binary Tree Right Side View
- 515 — Find Largest Value in Each Tree Row
- 637 — Average of Levels in Binary Tree
- 314 — Binary Tree Vertical Order Traversal

If you can write 199 from memory, you can write 637 and 515 with a two-line change — swap `node.val` for a running sum or a running max. That's the argument for studying patterns instead of grinding volume: interviewers rarely repeat the exact question you practiced, they ask a cousin of it.

AlgoMonster is built entirely around that premise. The curriculum is organized into roughly 48 patterns rather than a flat list of problems, with BFS-on-tree and DFS-on-tree as their own modules, and pattern-selection flowcharts that walk you from "what does this question look like" to "which template applies." The problem itself has its own writeup, and there's an active community thread under it where learners argue the two queue variants — record the front after pushing right children first, or keep the natural left-right order and check the index. That kind of comparison is genuinely useful when the point is building a reusable mental template rather than memorizing one file of code.

👉 [Work through the level-order pattern end to end](https://algo.monster/subscribe?fpr=li69)

## What AlgoMonster actually includes, and what it costs

AlgoMonster is a structured, text-and-diagram platform created by former Google engineers. The official plan page lists access to 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials, plus an AI assistant trained on the content, a Speedrun mode for timed review, real questions from major tech companies updated monthly, company-specific question banks, and a member discount on live bootcamps. Bonus courses are included at no extra cost — currently a CS fundamentals track and a Build an LLM course.

Two things it is not. There is no video content: every explanation is text with interactive diagrams, which is fast for readers and a poor fit if you learn by watching someone else's screen. And system design coverage is introductory — engineers targeting senior loops typically add a dedicated system design resource.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems | $0 | — | [Start with the free tier](https://algo.monster/subscribe?fpr=li69) |
| Pro Yearly | All patterns, lessons and problems, unlimited AI assistant, unlimited Speedrun, company question banks, bonus courses, 12 months of access | $99/year ($8.25/month billed annually) | Annual, cancel anytime | [Get Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro Lifetime | Everything in the annual plan plus lifetime updates and future courses | $189 | One-time payment | [Get Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

A few details that affect the math. There is no monthly option — it's annual or lifetime, and no team or enterprise tier. AlgoMonster runs sales frequently, so the number you see at checkout is the number that counts; treat the figures above as the current listed pricing rather than a permanent one.

The practical split: if you have an interview in 4–8 weeks and no plan to keep practicing afterward, the annual plan is the lower-risk buy. If you expect to change jobs every couple of years and go through this again, lifetime pays for itself against roughly two annual renewals, and you stop caring about sale timing.

👉 [Compare the annual and lifetime plans side by side](https://algo.monster/subscribe?fpr=li69)

## Quick answers

**Does the right side view always come from the right subtree?**
No. It's the rightmost node at each depth, which can be a left descendant whenever the right side of the tree is shallower.

**Can I solve it without a queue?**
Yes — the right-first DFS above uses only the recursion stack, O(h) extra space. On balanced trees that's less memory than BFS.

**Why not just record the deepest right branch?**
Because example 2 returns four values from a tree whose right branch ends at the second level. Depth and rightness are two separate conditions.

**Is this worth learning beyond the interview?**
It's the same traversal that powers level averages, zigzag ordering, vertical order, and a pile of graph problems. The queue-snapshot idiom in particular shows up constantly.

The right side view is a small problem with an outsized lesson: state the invariant before you write code. "Last node of each level" answers every edge case on its own, and once you can write that invariant down, both solutions take about five minutes.
