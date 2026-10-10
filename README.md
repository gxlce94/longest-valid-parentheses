# longest valid parentheses: three O(n) ways to solve it, with the stack sentinel, the DP merge case, and the O(1)-space trick

Pull up LeetCode 32 and the statement looks harmless: given a string of `(` and `)`, return the length of the longest valid substring. It's tagged **Hard**, and the reaction from most people is the same — how hard can counting brackets be?

Then the first submission fails on `")()())"`, the second fails on `"(()"`, and an hour has gone by.

The difficulty in this problem isn't the algorithm. It's the bookkeeping. Unmatched brackets act as walls that split a string into independent runs, and the length you want is the widest gap between two walls. Every correct solution is a different way of tracking where those walls are.

Here are the three approaches that get accepted, how each one handles the walls, and how to decide which one to write when an interviewer asks.

## What the problem asks, precisely

Given a string containing only `(` and `)`, return the length of the longest well-formed parentheses substring. Substring, not subsequence — the characters have to be contiguous.

The standard examples:

| Input | Output | Why |
| --- | --- | --- |
| `"(()"` | 2 | `"()"` is valid, the trailing `(` is not |
| `")()())"` | 4 | `"()()"` |
| `""` | 0 | empty string |

Constraints: `0 <= s.length <= 3 * 10^4`, characters limited to those two options. Time limit is generous enough for O(n) and O(n log n), not for O(n³).

### Two things to settle before writing code

**A valid substring always ends with `)`.** Nothing that ends in `(` can be balanced, which kills half the search space immediately.

**An unmatched `)` is not a nuisance, it's a boundary.** In `"())((()))"`, the unmatched `)` at index 2 separates the left run of length 2 from the right run of length 6. The answer is 6, not 8, because a substring can't span the wall. Getting this right is 90% of solving the problem.

## Why the naive stack approach falls apart

Everyone's first version looks like this: push `(` onto a stack, pop when you see `)`, count the pairs. It works for `"()()"` and dies on `"(()"`.

The problem is that the stack tells you *whether* the string is balanced, not *where* the balanced runs start and end. Once you've counted three pairs, you still don't know if they were three separate `"()"` runs separated by garbage or one contiguous `"((()))"`. The moment a stray `)` shows up in the middle, everything you counted before it becomes irrelevant — and a plain counter has no memory of where the previous boundary was.

The fix is the same idea in all three approaches: instead of counting brackets, track positions.

## Approach 1: index stack with a `-1` sentinel

Store indices, not characters. The stack always has an unmatched boundary sitting at its bottom, and the gap between that boundary and the current index is the length of the valid run you're standing inside.

Initialize the stack with `-1`. That's a fake boundary just before the string starts, so a run that begins at index 0 still produces a correct length.

For each character:

- `(` → push its index.
- `)` → pop the top. If the stack is now empty, this `)` had nothing to match, so push its index as the new boundary. If it's not empty, the current run length is `i - stack.top()`.

Trace on `")()())"`:

| i | char | Action | Stack after | Run length |
| --- | --- | --- | --- | --- |
| 0 | `)` | pop → empty → push 0 | `[0]` | — |
| 1 | `(` | push 1 | `[0,1]` | — |
| 2 | `)` | pop → top 0 | `[0]` | 2 − 0 = 2 |
| 3 | `(` | push 3 | `[0,3]` | — |
| 4 | `)` | pop → top 0 | `[0]` | 4 − 0 = 4 |
| 5 | `)` | pop → empty → push 5 | `[5]` | — |

Max is 4. The `)` at index 5 killed the run, exactly as it should.

python
def longestValidParentheses(s: str) -> int:
    stack = [-1]
    best = 0
    for i, ch in enumerate(s):
        if ch == "(":
            stack.append(i)
        else:
            stack.pop()
            if not stack:
                stack.append(i)   # this ')' becomes the new wall
            else:
                best = max(best, i - stack[-1])
    return best


O(n) time, O(n) space. The sentinel does real work: drop it and you'll write a branch like `i + 1 if not stack else i - stack[-1]`, which is the same logic wearing a hat.

## Approach 2: dynamic programming

`dp[i]` = length of the longest valid substring that *ends* at index `i`. Any index ending in `(` gets `dp[i] = 0` for free, so the array starts as all zeros and you only compute values where `s[i] == ')'`.

Two cases:

**Case A: `s[i-1] == '('`.** You just closed a fresh pair, so `dp[i] = dp[i-2] + 2`. The `dp[i-2]` term is what glues this pair onto whatever ended right before it — that's how `"()"` + `"()"` becomes 4 instead of 2.

**Case B: `s[i-1] == ')'`.** The character before is also a closing bracket, so `s[i]` has to match something further left. Look at `i - dp[i-1] - 1`: that's the position immediately before the whole valid block ending at `i-1`. If that character is `(`, you've bridged the gap — `dp[i] = dp[i-1] + 2 + dp[i - dp[i-1] - 2]`. If it's not a `(`, `dp[i]` stays 0.

That third term is the one people miss. Without it, `"()(())"` returns 4: the inner `"(())"` gets counted, the leading `"()"` doesn't. With it, `dp[5] = 2 + 2 + 2 = 6`.

python
def longestValidParentheses(s: str) -> int:
    n = len(s)
    dp = [0] * n
    best = 0
    for i in range(1, n):
        if s[i] != ")":
            continue
        if s[i - 1] == "(":
            dp[i] = (dp[i - 2] if i >= 2 else 0) + 2
        else:
            j = i - dp[i - 1] - 1
            if j >= 0 and s[j] == "(":
                dp[i] = dp[i - 1] + 2 + (dp[j - 1] if j >= 1 else 0)
        best = max(best, dp[i])
    return best


O(n) time, O(n) space. You cannot compress this to O(1) space — each `dp[i]` can reach back an arbitrary distance, so there's nothing to discard.

The failure mode here is index arithmetic, not logic. Two bounds checks matter: `i >= 2` in case A and `j >= 0` in case B. Miss either and you get a runtime error on inputs like `"()"` or `"))))"`.

## Approach 3: two counter passes, O(1) space

No stack, no array — just two integers. Scan left to right, incrementing `left` for `(` and `right` for `)`. Whenever `left == right`, you've found a balanced block and its length is `2 * left`. Whenever `right > left`, the string has become unsalvageable from this starting point, so reset both counters to zero.

That handles every case where the invalid bracket comes *after* the valid part. It completely fails on `"(()"`, where the extra `(` sits at the end and the counters never equalize.

So run the same scan backwards with the comparison flipped: reset when `left > right`, record when they're equal. The reverse pass sees the trailing `(` as a leading one and catches the run.

python
def longestValidParentheses(s: str) -> int:
    best = left = right = 0

    for ch in s:                      # catches extra ')'
        left += ch == "("
        right += ch == ")"
        if left == right:
            best = max(best, left + right)
        elif right > left:
            left = right = 0

    left = right = 0
    for ch in reversed(s):            # catches extra '('
        left += ch == "("
        right += ch == ")"
        if left == right:
            best = max(best, left + right)
        elif left > right:
            left = right = 0

    return best


O(n) time, O(1) space. Two passes, not one — a single pass only sees half the symmetry, which is precisely why this version trips people in interviews.

## Which one do you actually write?

| Approach | Time | Space | Write it when |
| --- | --- | --- | --- |
| Index stack with sentinel | O(n) | O(n) | Default choice. Easiest to reason about out loud, hardest to get wrong |
| DP | O(n) | O(n) | Interviewer wants optimal substructure, or you're being graded on a DP-heavy round |
| Two-pass counters | O(n) | O(1) | The follow-up: "can you do it without extra space?" |

The stack version is the one to reach for first. It's short, the sentinel is a single line, and you can narrate it while typing: *pop, check if empty, otherwise take the gap.* Interviewers can follow it in real time.

The two-pass version is the answer to the follow-up question, not the opening move. And with `n` capped at 30,000 characters, memory is not a real constraint at these sizes — O(n) space is nothing. The reason to learn the O(1) version is that it demonstrates you understand *why* the stack worked, not because you're optimizing for a limit that doesn't exist.

The DP version is worth knowing because the problem carries the Dynamic Programming tag and shows up in DP-focused rounds. If you're only memorizing one, memorize the stack.

## The pattern underneath: boundaries and reset points

Step back from the three solutions and the same idea appears in all of them. Brackets come in pairs, so the natural data structure is a stack. But you want a *length*, not a match, and length means positions, which means every approach is really answering one question: where does the current valid run begin?

- The stack answers it by storing the index of the last unmatched bracket at the bottom.
- The DP answers it by storing run lengths and reaching backwards.
- The counters answer it by throwing everything away the moment the run becomes unsalvageable.

That's why this problem is filed under the "stack" family alongside things like valid parentheses checking, minimum additions to make parentheses valid, and removing invalid brackets — same structure, different twist. Once you've seen it as "track the last unmatched position," the sentinel `-1` stops looking like a trick and starts looking like the obvious move.

This is the framing AlgoMonster builds its whole catalog around. The platform is written by engineers who worked at Google and by competitive programmers, and it's structured as roughly 48 reusable patterns rather than a problem list — you learn the pattern first, then see which questions it applies to. Each pattern comes with an explanation, a code template, and a set of practice problems, plus decision flowcharts for the "which pattern does this need?" moment that costs people the first five minutes of an interview. It's text and diagrams throughout, no videos, which is either a selling point or a dealbreaker depending on how you learn.

The site runs an AI assistant for hints, a Speedrun mode for timed practice, and company-tagged questions that get refreshed monthly. Subscribers also get access to a CS fundamentals course and a course on building an LLM from scratch; those are bundled into the same subscription rather than sold separately.

## AlgoMonster pricing: what the plans page lists

Current plan cards on the official subscribe page break down like this:

| Plan | What you get | Price | Billing |
| --- | --- | --- | --- |
| Free | Limited access to a sample of patterns and problems | $0 | — |
| Pro – Monthly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company-tagged questions updated monthly; full access for one month | $18.99/month | Monthly, cancel anytime |
| Pro – Yearly | Everything in Pro plus 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials, and a member discount on live bootcamps | $5.84/month, billed as $69.99/year | Annual (12 months) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates | $89 | One-time |

Purchase links: 👉 [Compare all AlgoMonster plans and current prices](https://algo.monster/subscribe?fpr=li69)

A caveat worth stating plainly: AlgoMonster runs promotions constantly, and third-party review sites quote a wide spread of numbers — some list an annual around $99 and a lifetime around $189, one lists a lifetime of $459 against a $918 list price, and a few still repeat figures from older pricing. Those pages go stale faster than they get updated. The numbers on the official plans page above are the ones captured from the subscription page itself; what you actually see at checkout is the number that matters.

Also worth knowing before you pay: there's no team or enterprise tier. This is a single-seat product for people preparing on their own.

## Is it worth paying for something you can solve for free?

If you're here for this one problem, no. Solutions, editorials, and full walkthroughs of LeetCode 32 are free in about forty places, including the ones you just read through. Paying for a subscription to solve a single bracket problem doesn't make sense, and nothing on AlgoMonster's plans page is priced for that.

The subscription makes sense in a narrower situation: you're working through a full interview cycle, you keep getting stuck on *identifying* which technique a problem needs rather than implementing it, and a structured pattern catalog would save you the flailing. That's the specific gap the platform is built for.

Two things to weigh honestly. First, format: it's text and illustrations, no video walkthroughs at any price. If you learn by watching someone talk through a solution, this will feel like a wall. Second, coverage: the pattern catalog and company-tagged questions are the core, but it's not an all-in-one interview platform — system design and behavioral content are introductory level, so if your loop includes a senior system design round, you're budgeting for something else on top.

The free tier is the honest test. Work through a couple of patterns there, see whether the text-first explanation style matches how you read, and only then decide whether the yearly makes sense. The yearly sits at $69.99 versus $89 for lifetime, which is an unusual gap — at that spread, the one-time payment costs only about 27% more than a single year, and the lifetime option includes future updates. If you're anywhere close to the fence, and you expect to interview more than once over the next few years, the lifetime is the easier call. 👉 [See the current plan options](https://algo.monster?fpr=li69)

## Bug checklist before you submit

Run through these five before declaring victory on LeetCode 32. Every one of them is a real failure mode, not a hypothetical.

- **Missing sentinel.** Initializing an empty stack instead of `[-1]` breaks any input that starts with `(`, which is most of them.
- **Not resetting the boundary.** After popping, you must check whether the stack is empty and push the current index if it is. Skip that and `")()())"` returns 6 instead of 4.
- **DP bounds.** `dp[i-2]` when `i == 1` and `dp[j-1]` when `j == 0` are both out-of-range accesses waiting to happen.
- **Forgetting the merge term.** Dropping `dp[i - dp[i-1] - 2]` from case B silently undercounts `"()(())"` by 2. Silent wrong answers are worse than crashes.
- **One counter pass instead of two.** Forward only, `"(()"` returns 0. Backward only, `"())"` returns 0. You need both directions to cover both asymmetries.

Fix all five and the problem is genuinely solved — not pattern-matched into passing, actually solved, which is the version you can reproduce on a whiteboard when it counts.
