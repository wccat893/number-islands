# number of islands python: DFS, BFS, and union-find code that survives a 300×300 grid

Most people searching for this already know what an island is. They know it's land connected horizontally and vertically, they know you count the components, and they can sketch the algorithm on a napkin. What they need is Python that runs on the real constraints — because the recursive solution almost everyone writes first is the same one that throws a `RecursionError` on the hidden test case and leaves you staring at a green "passed 2 of 3 examples" screen.

So let's go through the whole thing: what LeetCode 200 actually asks, three working solutions, the bugs that show up in discussion threads over and over, and how this one problem sits inside a much smaller family of patterns than it looks.

## What LeetCode 200 actually asks

You get an `m x n` grid of `'1'` (land) and `'0'` (water). An island is land connected 4-directionally — up, down, left, right, no diagonals. All four edges of the grid are surrounded by water. Return the number of islands.

The constraints matter more than the prose: `1 <= m, n <= 300`. That single line is why half the solutions you'll find online are subtly broken.

Two examples, both worth memorising because interviewers often hand you the first one and expect the second:

text
Input:  [["1","1","1","1","0"],
         ["1","1","0","1","0"],
         ["1","1","0","0","0"],
         ["0","0","0","0","0"]]
Output: 1

Input:  [["1","1","0","0","0"],
         ["1","1","0","0","0"],
         ["0","0","1","0","0"],
         ["0","0","0","1","1"]]
Output: 3


One thing to watch: "number of islands" covers two different problems. LC 200 is a static grid. LC 305, *Number of Islands II*, starts with all water and adds land one position at a time, returning the count after each addition. Same name, different tool — if you're halfway through a solution and the problem talks about `positions`, you've landed on the wrong one.

## The pattern underneath

A grid is a graph in disguise. Each cell is a node, each 4-neighbour pair is an edge, and an island is a connected component. Once you frame it that way, the solution menu collapses to two moves: explore each component when you find it (DFS or BFS), or merge components as you go (union-find). Everything else is bookkeeping — bounds checks, a visited marker, and a counter.

## Solution 1: recursive DFS

The version that reads best and breaks most often.

python
from typing import List

class Solution:
    def numIslands(self, grid: List[List[str]]) -> int:
        if not grid or not grid[0]:
            return 0

        rows, cols = len(grid), len(grid[0])
        count = 0

        def sink(r: int, c: int) -> None:
            if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != "1":
                return
            grid[r][c] = "0"          # mark visited
            sink(r + 1, c)
            sink(r - 1, c)
            sink(r, c + 1)
            sink(r, c - 1)

        for r in range(rows):
            for c in range(cols):
                if grid[r][c] == "1":
                    count += 1
                    sink(r, c)
        return count


Four lines of actual work. The catch is depth. On a 300×300 grid that's all land, the recursive chain can run roughly 90,000 frames deep. Python's default recursion limit is around 1,000, so that case raises `RecursionError` while every small test passes.

You can call `sys.setrecursionlimit(10**6)`, and plenty of people do. It works, but it removes the guardrail rather than the problem: deep enough recursion in CPython can exhaust the C stack and crash the interpreter instead of raising a clean exception. Raising the limit a few thousand is a reasonable compromise. Needing 90,000 frames is a signal to switch approaches.

## Solution 2: iterative DFS with an explicit stack

python
class Solution:
    def numIslands(self, grid: List[List[str]]) -> int:
        rows, cols = len(grid), len(grid[0])
        count = 0

        for r in range(rows):
            for c in range(cols):
                if grid[r][c] != "1":
                    continue
                count += 1
                stack = [(r, c)]
                grid[r][c] = "0"
                while stack:
                    cr, cc = stack.pop()
                    for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
                        nr, nc = cr + dr, cc + dc
                        if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == "1":
                            grid[nr][nc] = "0"   # mark on push
                            stack.append((nr, nc))
        return count


The detail that matters is where you mark a cell visited. Marking when you push keeps a cell from being queued twice; marking when you pop means the same cell can sit in the stack several times and get processed repeatedly. On a dense grid that's wasted memory and wasted iterations.

## Solution 3: BFS with `deque`

Same shape, different container.

python
from collections import deque
from typing import List

class Solution:
    def numIslands(self, grid: List[List[str]]) -> int:
        if not grid or not grid[0]:
            return 0

        rows, cols = len(grid), len(grid[0])
        count = 0

        for r in range(rows):
            for c in range(cols):
                if grid[r][c] != "1":
                    continue
                count += 1
                q = deque([(r, c)])
                grid[r][c] = "0"
                while q:
                    cr, cc = q.popleft()      # popleft, not pop
                    for dr, dc in ((1, 0), (-1, 0), (0, 1), (0, -1)):
                        nr, nc = cr + dr, cc + dc
                        if 0 <= nr < rows and 0 <= nc < cols and grid[nr][nc] == "1":
                            grid[nr][nc] = "0"
                            q.append((nr, nc))
        return count


`deque.popleft()` is O(1); calling `pop()` on a list-based queue is also fine here, but `list.pop(0)` is O(n) and turns your linear solution quadratic. That mistake shows up constantly in posted solutions.

DFS and BFS have identical complexity here, so the choice isn't about speed. Pick BFS when the follow-up question involves distance or layers — "fewest steps to reach land", "rotten oranges spreading per minute", "nearest exit". Pick DFS when you're only counting components and want the shortest code on the board.

## Solution 4: union-find, and when it's the right tool

For a static grid, union-find is more code for the same asymptotic result and usually slower in practice — you pay the `find` calls and lose cache friendliness. It's not the version to write on LC 200.

It becomes the right answer on the incremental variant. If land arrives one position at a time and you need the count after each arrival, re-running a flood fill every time costs you `O(k * m * n)` for `k` additions. Union-find with path compression and union by size does it in roughly `O(k * α(m·n))`: adding a piece of land starts a new component, then each land neighbour merges, and the counter either stays put or drops by one per merge.

If you want to see that template written out properly, AlgoMonster's *Matrix as Graph* material covers the static problem and a union-find walkthrough of the incremental one side by side, so the "why not union-find here" question answers itself. [👉 Try the free lessons and compare the two write-ups yourself](https://algo.monster?fpr=li69)

## The bugs that break correct-looking code

**Comparing against `0` instead of `"0"`.** The grid holds strings. `grid[r][c] == 0` is never true, so no cell is ever treated as visited, and every single land cell gets counted as its own island. This exact slip is all over AlgoMonster's discussion thread for the problem — someone posts a solution that returns `1` for a grid whose expected output is `3`, and the answer turns out to be a missing pair of quotes. If your output looks like the total count of `'1'` cells, check this first.

**Mutating the caller's grid.** The LeetCode signature lets you overwrite cells, and sinking land as you go is the standard trick — it saves a separate visited matrix. But you've destroyed the input. If the same `grid` gets reused by another test case or another function, your count comes out wrong and the bug is invisible at the call site. Either keep a `visited` set, or copy the grid at the top of the function and be explicit that you're doing it.

**Marking at pop instead of push.** Covered above, but it's the most common source of "why is this BFS using 400 MB".

## Complexity, and how to say it out loud

Time is `O(m·n)`. Every cell is pushed onto the stack or queue at most once, because the moment it's marked it can never qualify again — the outer loop and the flood fill each touch a cell once.

Space is `O(m·n)` in the worst case. A grid that's entirely land can put every cell on the stack at some point. On a mostly-water grid it's dramatically less. If you're not allowed to mutate the input, add `O(m·n)` for the visited structure.

Saying it as "O(m·n) time, O(m·n) worst-case space, and I'm mutating the grid so I don't need a visited matrix" is what interviewers want to hear — the reasoning, not the formula.

## Answering it in an interview

The code is the easy half. What separates a pass from a "we'll let you know":

- **Ask about connectivity first.** Diagonals in or out? LeetCode says 4-directional, but plenty of variants count 8. This is a thirty-second question that changes your direction list.
- **Ask what the cell type is.** LC 200 uses `'1'`/`'0'` strings. LintCode's version of the same problem gives you a boolean matrix of `1`/`0` integers with a tighter 50×50 bound. Same algorithm, different comparisons.
- **Ask whether you're allowed to modify the grid.** It's a real trade-off and naming it shows you know what mutating costs.
- **Say the pattern before you type.** "A grid is a graph, an island is a connected component, so I'll count components." Interviewers grade the recognition as much as the implementation.
- **Lead with recursion, name its limit.** "This is the clean version, but at the 300×300 constraint the recursion depth can hit 90,000, so if you want it production-safe I'd use an explicit stack." That sentence is worth more than the code.

Likely follow-ups: maximum island area, the incremental variant with positions, the diagonal-connected variant, and "what if the grid doesn't fit in memory" — which is answered by scanning row by row and union-finding each row against the previous one.

## Where this problem actually lives in a prep plan

Number of Islands isn't a one-off. It's the visible edge of a family: connected components, flood fill, surrounded regions, max area of island, rotting oranges, walls and gates, Pacific Atlantic water flow. Same move every time — pick a neighbourhood, flood the component, count or record something — with a different stopping condition and a different grid shape.

That's the argument for pattern-based prep over problem-count prep. Grinding these one at a time gets you good at the ones you've seen and flat on the ones you haven't, which is exactly the wrong distribution for an interview where the problem is new by definition. AlgoMonster is built around that idea: 48 patterns, each written as a template plus the variants that break the template, with a decision flowchart that asks questions about the input ("is it a grid? is the input sorted? are you looking for a contiguous run?") instead of handing you problem number 1,347.

## AlgoMonster plans and pricing

Worth knowing before you click anything: AlgoMonster's checkout price moves with sales, and review blogs quote figures from whatever sale they snapshot — $99/year and $189 lifetime show up in several 2026 write-ups, and one lists $459 lifetime. Treat the subscribe page as the source of truth. Here's what it shows as of writing:

| Plan | What you get | Price | Billing | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to a slice of the lessons and problems — enough to judge whether text-and-diagram teaching works for you | $0 | — | [ Start with the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; unlimited AI assistant; unlimited Speedrun; real questions from top tech companies updated monthly; member discount on live bootcamps | $5.84/mo, billed as $69.99/year | 12 months, cancel or upgrade anytime | [ Get Pro yearly](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the yearly plan plus lifetime updates, plus the bonus courses (CS fundamentals, building an LLM from scratch) | $89 | One-time payment | [ Get Pro lifetime](https://algo.monster?fpr=li69) |

There's no monthly plan — it's annual or lifetime, nothing in between.

Two limits worth stating plainly, because they matter for the "is this worth it" call. There are no video walkthroughs at all; AlgoMonster is entirely text and diagrams. And the problem bank is around 231 problems, far smaller than LeetCode's 3,000-plus. You're buying the framework and the templates, not volume. Most people end up pairing it with a free problem source for repetition.

For someone with an interview in four to eight weeks, the yearly plan matches the timeline and costs less than half a month of a bootcamp. Lifetime is the better math if you expect to do this again in two or three years, which for most working engineers is less a prediction than a description of the last decade.

## Quick answers to the questions people actually type

**DFS or BFS for number of islands?** Either. Identical complexity. Use BFS if a follow-up involves distance, DFS if you just want the shortest code.

**Why does my solution return 1 instead of 3?** Almost always `== 0` instead of `== "0"`, or a visited marker that never gets set. Print the grid after the first flood fill and the answer is usually obvious.

**Does recursion work at the maximum constraint?** Not reliably. A 300×300 all-land grid can need around 90,000 frames against a default limit of about 1,000. Use the iterative stack version, or accept the risk with a raised limit.

**Is union-find faster?** Not on the static grid. It's the right call on the incremental variant, where re-running a flood fill after every land addition gets expensive fast.

## The short version

Write the recursive DFS to show the idea, keep the iterative stack version in your back pocket for the constraint the interviewer is going to ask about, and know that union-find exists but isn't the answer here. The bug that trips most people isn't algorithmic at all — it's a quote mark.
