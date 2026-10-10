# number of islands: the pattern behind LeetCode 200, plus the follow-ups that actually decide the interview

Most people who type "number of islands" into a search bar aren't looking for a definition. They're 20 minutes into a problem, their code returns `1` when the expected answer is `3`, and they want to know why.

So let's start there. The answer to that specific bug is almost always a type mismatch, and you'll find it in the gotchas section below. Everything else — the two standard solutions, the complexity answer you should say out loud, and the variations interviewers stack on top of the same 20 lines of code — comes after.

## What the problem actually gives you

LeetCode 200 hands you an `m x n` grid of characters: `'1'` is land, `'0'` is water. Constraints are `1 <= m, n <= 300`, so the largest input is 90,000 cells.

Two rules do most of the work:

- Land connects **horizontally or vertically**, never diagonally.
- The grid edges are surrounded by water, so you don't need to handle wrap-around.

Count the groups. Example one:


1 1 1 1 0
1 1 0 1 0
1 1 0 0 0
0 0 0 0 0


Answer: `1`. The top-left block is one connected mass even though it has a notch in it.


1 1 0 0 0
1 1 0 0 0
0 0 1 0 0
0 0 0 1 1


Answer: `3`.

The reason this problem shows up so often in interviews isn't the grid. It's that it's the cleanest possible framing of a **connected components** problem. Every land cell is a node. Edges go to the four neighbors that are also land. An island is a connected component, and counting islands is counting components. Once you see that, the solution writes itself — and so do about six other problems that are the same problem wearing a different hat.

## The one idea you need

Scan the grid cell by cell. Every time you find a `'1'` that nothing has claimed yet, that's a new island: increment your counter, then flood fill from that cell and mark every connected land cell as visited. The number of flood fills you started equals the number of islands.

You don't need a separate `visited` array to do this. Flip each visited `'1'` to `'0'` as you go. Same information, less memory, and the outer scan naturally skips cells you've already handled.

Recursive DFS:

python
def numIslands(grid):
    if not grid or not grid[0]:
        return 0
    rows, cols = len(grid), len(grid[0])
    count = 0

    def sink(r, c):
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != '1':
            return
        grid[r][c] = '0'          # mark visited before recursing
        sink(r + 1, c)
        sink(r - 1, c)
        sink(r, c + 1)
        sink(r, c - 1)

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                count += 1
                sink(r, c)
    return count


BFS, same skeleton with an explicit queue:

python
from collections import deque

def numIslands(grid):
    if not grid or not grid[0]:
        return 0
    rows, cols = len(grid), len(grid[0])
    count = 0

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] != '1':
                continue
            count += 1
            grid[r][c] = '0'               # mark on enqueue, not on pop
            q = deque([(r, c)])
            while q:
                cr, cc = q.popleft()
                for nr, nc in ((cr+1, cc), (cr-1, cc), (cr, cc+1), (cr, cc-1)):
                    if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == '1':
                        grid[nr][nc] = '0'
                        q.append((nr, nc))
    return count


Both are `O(m * n)` time. Every cell gets inspected a constant number of times: once by the outer scan, once by whichever flood fill reaches it. Space is `O(m * n)` in the worst case — either the recursion stack or the queue, when the whole grid is a single island.

Notice the comment in the BFS version. That line is not cosmetic, and it's the second most common reason this solution fails a hidden test.

## DFS, BFS, or union-find

All three are correct. They differ in what breaks first.

| Approach | What it costs you | When it's the right call |
| --- | --- | --- |
| Recursive DFS | Shortest code, but recursion depth equals the size of the largest island — up to 90,000 frames | Default choice in Java, C++, or Go; fine in Python for small grids |
| Iterative BFS | ~10 extra lines and a queue, no stack limit | Python with large open grids, or when the interviewer asks about stack depth |
| Union-find | Most code, needs parent and rank arrays | Grids that change over time (islands being added), or when you're told the grid streams in |

The union-find version is worth knowing because it's what the follow-up problem is built on. The version I've seen recommended most often unions each land cell with its left and up neighbors, then returns the number of disjoint sets — roughly `O(V log I)` with union by rank, where `I` is the size of the largest island.

## Four ways this goes wrong under pressure

**1. Comparing a character to an integer.** This is your bug. `grid[r][c] == 0` is false for every cell in a `char` or `string` grid, so your flood fill either never fires or never stops correctly, and the count collapses. On one AlgoMonster discussion thread about this exact problem, a user posted "expected output: 3, actual output: 1" and the fix was literally changing `0` to `"0"` [1]. If your answer is suspiciously small, check that first.

**2. Marking cells visited when you dequeue them instead of when you enqueue them.** With the pop-side check, the same cell can be pushed multiple times before it's ever processed. On a fully open 300x300 grid that's the difference between passing and a time-limit-exceeded verdict, because the queue balloons with duplicates. Mark on enqueue and the problem disappears.

**3. Assuming recursion is free.** Python's default recursion limit is around 1,000. A 300x300 grid that's all land is one island with 90,000 cells, and a naive recursive DFS will hit `RecursionError` long before it finishes. Either use BFS, use an explicit stack, or raise the limit and accept the deeper stack. Interviewers do ask about this — "what's your worst case?" is a real question, not a formality.

**4. Mutating the caller's data without saying so.** Flipping `'1'` to `'0'` is the standard trick and LeetCode doesn't care, but in a live interview you should mention it: "I'm destroying the input to save space; if you need it preserved, I'll use a visited set instead." That one sentence signals more than the code does.

The diagonal assumption deserves a mention too. It's stated in the problem, and people still misread it. Ask before you assume, especially in the non-LeetCode phrasing of this question where "adjacent" is sometimes left undefined.

## The follow-ups are the same code with one twist

This is the part most write-ups skip, and it's where the actual interview difficulty lives. Interviewers use Number of Islands as a launchpad because you can change one constraint and the problem becomes a different one.

- **Max Area of Island (695).** Return the size of the largest island instead of the count. Flood fill returns the number of cells it sank; track the maximum.
- **Number of Distinct Islands (694).** Two islands count as the same if one can be translated onto the other. The trick is to serialize each island's traversal — record the direction you took into each cell and the direction you took on the way out, then drop those strings into a set. AlgoMonster's version of this problem uses exactly that path-signature approach [2].
- **Island Perimeter (463).** Count edges instead of components. Every land cell contributes 4, minus 2 for each shared side.
- **Making A Large Island (827).** Label each island with an ID and its area, then try flipping every water cell and sum the distinct neighbor IDs.
- **Number of Islands II (305).** Land is added one coordinate at a time and you report the count after each addition. Union-find is no longer an alternative — it's the answer.
- **"What if the grid is too big to fit in memory?"** Rows arrive as a stream. Union-find wins again: you only need the previous row's component IDs to union against the current one.
- **"What if diagonals count?"** Eight directions instead of four. If you wrote your neighbor loop with a direction array, this is a one-line change, which is the real reason to use a direction array.

A good answer to "any follow-ups?" is to name two or three of these and say what changes. It shows you understand the pattern rather than one problem.

## What to actually say out loud

The interview isn't scored on whether you can type a flood fill. It's scored on whether you get to the right approach fast. A reasonable 45-minute script:

1. Restate the problem and confirm two things: four-directional adjacency only, and the input can be modified.
2. Name the shape: "this is connected components on a grid graph."
3. State the approach before writing code: scan, and flood fill from each unvisited land cell.
4. Give complexity up front: `O(m * n)` time, `O(m * n)` space worst case.
5. Code it, then trace it on a 3x3 example.
6. Offer the trade-off you didn't take: "BFS would avoid the recursion depth issue if the grid were all land."

The candidates who struggle here usually aren't failing on implementation. They're spending the first 15 minutes trying approaches that don't fit the problem's structure — which is a pattern-recognition problem, not a coding problem.

## Where this problem lives in a pattern-based curriculum

Number of Islands is the canonical example of one specific pattern: treating a matrix as a graph. Same traversal templates you'd use on an adjacency list, but neighbors come from index arithmetic instead of a stored edge list, and bounds checks do the work of a "is this neighbor reachable" check.

AlgoMonster files it under Graph / Matrix as Graph, alongside the other grid traversal problems, with a BFS/DFS template you adapt rather than rewrite [1]. The site's whole premise is that this pattern-first framing beats grinding problems in random order: roughly 48 patterns, with decision flowcharts that walk you from problem description to likely algorithm — sorted input, contiguous subarray, tree or graph — so you stop guessing. The format is text and interactive diagrams rather than video, which is either the reason you'll like it or the reason you won't.

You can read the Number of Islands walkthrough itself for free: 👉 [read AlgoMonster's Number of Islands breakdown](https://algo.monster/liteproblems/200?fpr=li69). The distinct-islands variant sits at 👉 [Number of Distinct Islands on AlgoMonster](https://algo.monster/liteproblems/694?fpr=li69).

## AlgoMonster's plans and current pricing

There's no monthly option and no team plan. Three tiers show on the site, and the paid ones rotate through promotions often enough that the number you see at checkout can differ from the number in a review written three months ago.

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample lessons and problems — enough to judge the teaching style, not enough to prep with | $0 | — | [Browse AlgoMonster free content](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems unlocked; unlimited AI assistant; unlimited Speedrun; company-specific question lists; 3,000+ editorials; bonus Foundation and Build-an-LLM courses; 12 months of access | Listed at $119/year ($9.92/mo); promos have run it at $99/year | Annual, cancel or upgrade anytime | [See the current Pro Yearly price](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in Yearly plus lifetime updates, one 1:1 coaching session with the founder, and an internal referral for qualified candidates | Listed at $459; promos have run it as low as $189 | One-time payment | [Check the Pro Lifetime price](https://algo.monster/subscribe?fpr=li69) |

Two things worth flagging before you buy. First, some specialized courses built with external partners — the Quant Interview Prep track among them — are sold separately from these plans. Second, AlgoMonster has historically run 50%-style sales, which is why you'll see lifetime access quoted at $189 in some places and $459 in others. Both numbers have appeared on the official subscribe page at different times. The checkout page is the only price that matters; treat any figure in a blog post, including this one, as a snapshot.

Outside the plans: no videos anywhere in the product, around 231 lessons and problems in the core curriculum, and basic system design and behavioral content that's a starting point rather than enough for a senior loop. AlgoMonster also states a job guarantee — at least one offer after completing the curriculum — and publishes learner testimonials from engineers at Meta and Amazon on its own site.

## Is it worth paying when the solution is a free search away?

No, if Number of Islands is the only thing standing between you and your interview. The solution is above, LeetCode's editorial is free, and this pattern takes an afternoon to internalize.

The paid case is different, and it's narrower than the marketing suggests: you have 4 to 8 weeks, you can already write correct code in one language, and your actual bottleneck is deciding which approach fits a problem you've never seen. That's the failure mode the pattern-and-flowchart approach targets. If your bottleneck is instead practice volume or live pressure with someone watching you code, a text-based curriculum won't fix either one, and you'll still need to solve a few hundred problems somewhere else.

For someone planning to interview repeatedly over several years, the one-time lifetime price has a defensible logic. For a single interview loop, a year of access is the cheaper bet — and if you just want to see whether the teaching style suits you, the free tier costs nothing to evaluate. 👉 [Start on the free tier or pick a plan](https://algo.monster/subscribe?fpr=li69)

## Quick answers

**Is Number of Islands easy or medium?** Medium on LeetCode, and one of the most frequently asked grid questions. The algorithm is easy; the follow-ups aren't.

**What's the time complexity?** `O(m * n)`. Each cell is visited at most twice — once by the outer scan, once by the flood fill that claims it.

**Does the grid start empty?** Constraints say `m, n >= 1`, but defensive code checks `if not grid or not grid[0]` anyway. Interviewers sometimes hand you an empty grid to see if you crash.

**Do diagonals count as connected?** No, unless you're told otherwise. Ask.

**Can I solve it with union-find?** Yes, and you should be able to sketch it, because "what if land keeps getting added?" is a standard follow-up.

**Does AlgoMonster have a free trial or monthly plan?** There's a free tier with limited content. No monthly billing — annual or lifetime only.
