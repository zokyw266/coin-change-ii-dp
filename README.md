# Coin Change II: Why Loop Order Decides Combinations vs Permutations, the Space-Optimized DP, and What Interviewers Actually Ask

You wrote the nested loops, ran them on `amount = 5, coins = [1,2,5]`, and got 9. The expected answer is 4. Your recurrence is fine — the arithmetic is correct. The problem is the order of the two loops, and that single detail is what makes Coin Change II the problem people get wrong after confidently explaining the recurrence out loud.

Coin Change II (LeetCode 518) is a counting problem wearing a knapsack costume. Get the loop order right and it collapses into four lines. Get it wrong and you quietly count permutations while the judge counts combinations. Below: the recurrence, the space optimization, the overflow case newer test data actually triggers, and how this differs from the three problems it gets confused with.

## What the question is actually asking

You're given an integer array `coins` and an integer `amount`. Return the number of combinations that make up that amount. Each denomination is available infinitely many times, and order doesn't count.

`coins = [1,2,5], amount = 5` gives 4:

- `5 = 5`
- `5 = 2 + 2 + 1`
- `5 = 2 + 1 + 1 + 1`
- `5 = 1 + 1 + 1 + 1 + 1`

`2+2+1`, `2+1+2` and `1+2+2` are the same combination — three coins of the same denominations — and get counted once.

Constraints worth noting: up to 300 coin types, each coin up to 5000, `amount` up to 5000, all coin values unique. The final answer is guaranteed to fit in a signed 32-bit integer. That last sentence matters more than it looks, and we'll come back to it.

A quick sanity check on the edge cases before any code:

| Input | Output | Why |
| --- | --- | --- |
| `amount = 3, coins = [2]` | `0` | No combination of 2s reaches 3 |
| `amount = 10, coins = [10]` | `1` | Exact match, one coin |
| `amount = 0, coins = [3,7,11]` | `1` | Choosing nothing is a combination |

That last row catches people. An amount of zero is reachable in exactly one way — the empty multiset — and that's where `dp[0] = 1` comes from.

## The 2D recurrence, without the hand-waving

Define `dp[i][j]` as the number of combinations that make amount `j` using the first `i` coin types. Two choices per state:

- Don't use coin `i` at all: inherit `dp[i-1][j]`
- Use coin `i` at least once: take `dp[i][j - coins[i-1]]`

So:


dp[i][j] = dp[i-1][j] + dp[i][j - coins[i-1]]


The part that trips people up is the second term staying on row `i` instead of dropping to `i-1`. That's the unbounded part. If coins could be used once, you'd write `dp[i-1][j - coins[i-1]]`; because supply is infinite, you stay put and let the same coin be spent again. Recursion-to-DP converters often flatten that to `i-1` and end up with the 0/1 knapsack answer instead.

Base case: `dp[0][0] = 1`, everything else in row 0 is 0. One way to make nothing, no ways to make anything with no coins.

Tracing `coins = [1,2,5], amount = 5` with rows as coin types:

| Amount | After `1` | After `1,2` | After `1,2,5` |
| --- | --- | --- | --- |
| 0 | 1 | 1 | 1 |
| 1 | 1 | 1 | 1 |
| 2 | 1 | 2 | 2 |
| 3 | 1 | 2 | 2 |
| 4 | 1 | 3 | 3 |
| 5 | 1 | 3 | 4 |

Row by row it's unremarkable. The interesting column is amount 2: adding the coin `2` takes it from 1 way (`1+1`) to 2 (`1+1`, `2`). Amount 5 only becomes 4 once the coin `5` arrives. Nothing is ever counted twice, because coins are processed in a fixed order and each row only inherits forward.

## The 1D version, and the rule that actually matters

You don't need two dimensions. Since row `i` only reads from row `i` and row `i-1` at lower indices, one array works:


dp = [0] * (amount + 1)
dp[0] = 1

for coin in coins:                 # coin in the OUTER loop
    for j in range(coin, amount + 1):
        dp[j] += dp[j - coin]

return dp[amount]


Two things are doing work here. `dp[0] = 1` seeds the empty combination. The inner loop starting at `coin` silently handles the case where a coin is larger than the amount — those states just get skipped.

The outer loop is the whole problem. Swap the loops and you count permutations:


for j in range(1, amount + 1):     # amount in the OUTER loop
    for coin in coins:
        if j >= coin:
            dp[j] += dp[j - coin]


For `amount = 4, coins = [1,2]`, the coin-outer version gives 3 — `[1,1,1,1]`, `[1,1,2]`, `[2,2]`. The amount-outer version gives 5, because `[1,1,2]`, `[1,2,1]` and `[2,1,1]` get counted separately. Same numbers, different question. The fix is one line of reordering, and it's a genuine difference in what you're solving, not a micro-optimization.

## Python, both versions

Two-dimensional, closest to the derivation:

python
class Solution:
    def change(self, amount: int, coins: list[int]) -> int:
        dp = [[0] * (amount + 1) for _ in range(len(coins) + 1)]
        for i in range(len(coins) + 1):
            dp[i][0] = 1

        for i in range(1, len(coins) + 1):
            c = coins[i - 1]
            for j in range(amount + 1):
                dp[i][j] = dp[i - 1][j]
                if j >= c:
                    dp[i][j] += dp[i][j - c]

        return dp[len(coins)][amount]


One-dimensional, which is what you'd write under time pressure once you trust the loop order:

python
class Solution:
    def change(self, amount: int, coins: list[int]) -> int:
        dp = [0] * (amount + 1)
        dp[0] = 1
        for c in coins:
            for j in range(c, amount + 1):
                dp[j] += dp[j - c]
        return dp[amount]


Recursion with memoization works too and is often easier to reason about first, since the "pick the coin and stay at the same index" decision maps directly onto the state `(i, remaining)`. It's the same `O(n × amount)` work with extra stack overhead. Teams reading your code in an interview will not care which you pick, but they will care whether you can explain the loop order, because that's the part that reveals whether you understand the problem or recognized a shape.

## Complexity, and the overflow nobody mentions

Time is `O(n × amount)` where `n` is the number of coin types — every `(coin, amount)` pair is touched once. Space drops from `O(n × amount)` to `O(amount)` with the rolling array.

The guarantee that "the answer fits in a signed 32-bit integer" is about the returned value, not about the intermediate states your DP walks through on the way there. There's a test case floating around with `amount = 4681` and coins `[2,4,6,...,4680]`. Every coin is even and the target is odd, so the final answer is 0. But the table still computes the number of ways to reach 4680 on the way, and that intermediate value is large enough to blow past `INT_MAX` — and in some implementations past `long long` too. In C++ that's a signed overflow, which is undefined behaviour, not a wraparound you can shrug off.

The fix is to accumulate in a wider type and clamp:

cpp
long long ways = (long long)dp[i - 1][j] + dp[i][j - coins[i - 1]];
dp[i][j] = (int)std::min(ways, (long long)INT_MAX);


Python and Java are partly insulated — Python has arbitrary-precision integers, Java's `int` silently wraps rather than invoking UB — but if you're writing the C++ version in an interview of a company that grades on edge cases, capping the intermediate is the difference between passing and a wrong-answer on a case where your final answer was correct all along.

## Coin Change II vs the three problems it gets mixed up with

Most of the confusion here is really confusion between four different questions that share a table shape. Sorting them out early saves you from pattern-matching the wrong solution in an interview.

| Problem | What it asks | Can a coin repeat? | Order matters? | Recurrence |
| --- | --- | --- | --- | --- |
| Coin Change (322) | Minimum coins to reach the amount | Yes | No | `dp[j] = min(dp[j], dp[j-c] + 1)` |
| **Coin Change II (518)** | **Number of combinations** | Yes | No | `dp[j] += dp[j-c]` |
| Combination Sum IV (377) | Number of ordered sequences | Yes | Yes | amount in the outer loop |
| Target Sum (494) | Ways to assign `+`/`-` to hit a target | Each number once | No | 0/1 knapsack counting |

The useful takeaway: 322 and 518 differ only in whether you minimize or sum. 518 and 377 differ only in loop order. 494 differs because each item is used once, which flips the inner loop to run backwards. If you can state those three distinctions from memory, the code writes itself.

## Getting it to stick before the interview

Reading one editorial teaches you this problem. It doesn't teach you to recognize an unbounded-knapsack counting problem 40 minutes into a coding screen, when the interviewer has framed it as "how many ways can we make change with these denominations" and you have 15 minutes left. That recognition step is a separate skill, and it's the one the format of most free resources doesn't build — a solution page gives you the loop order already fixed, with no signal about how you were supposed to arrive at it.

This is the gap AlgoMonster is built for. It's a pattern-based course put together by ex-Google engineers, and the Dynamic Programming track covers unbounded knapsack and the counting problems that hang off it. Instead of a page per problem, you get the pattern's template across several languages, the reasoning for why the loops sit in that order, and a set of nearby problems that are deliberately just different enough that you have to decide which variant you're looking at. Their decision flowcharts are the part worth trying: a chain of questions about the input (sorted? contiguous? can elements repeat? counting or optimizing?) that narrows you to a pattern before you start writing. There's a free write-up of this exact problem on their site, plus a discussion thread where users work through the bottom-up conversion and separately note that reading an editorial is not the same as producing one. 👉 [Start with the free lessons and see whether the pattern format fits how you learn](https://algo.monster?fpr=li69)

Worth being straight about the tradeoffs. AlgoMonster is text and diagrams — there is no video anywhere on the platform, which is great if you learn by reading and running code, and tedious if you don't. The problem bank is in the low hundreds, well short of LeetCode's catalogue, so volume practice still happens elsewhere; most people who use it pair it with LeetCode rather than replacing it. System design coverage is introductory and won't carry a senior loop on its own. It also has no free-form trial — there's a free tier with a subset of content, and that's your preview.

## What AlgoMonster costs right now

The pricing is unusually simple: three tiers, no monthly billing, no add-on upsells on the core curriculum.

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of lessons and problems; enough to judge the teaching style | $0 | — | [Open a free account](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems unlocked, unlimited AI assistant, unlimited Speedrun, company question banks refreshed monthly, member discount on live bootcamps, plus the Foundation and Build an LLM courses | $8.25/mo | Billed annually at $99/year | [Compare the yearly plan](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the yearly plan, permanently, plus lifetime content updates and an internal referral to a top tech company for qualified candidates | $189 | One-time | [See the lifetime plan](https://algo.monster?fpr=li69) |

Two things to check yourself before buying, because pricing on this page is promotional and shifts. The listed prices above are what the plan page currently shows; the same page labels it a limited-time offer, and third-party discount trackers have historically listed a higher "original" price, so the number you see may differ from the number here. And the Quant Interview Prep course is developed with an external partner and sold separately — it isn't bundled.

On the arithmetic: if you're prepping for one interview cycle, the yearly plan is the straightforward pick. Lifetime pays for itself in year two, which is roughly the point at which a typical engineer re-enters interview prep anyway. The plans do not include live coaching or mock interviews with real engineers, and no plan tier adds it.

## Short answers to the questions that bring people here

**Why is `dp[0] = 1` and not 0?** Because there's exactly one combination that reaches amount 0 — using no coins. Zero would say the same sum is unreachable, and every subsequent count would collapse to nothing.

**Does `amount = 0` return 1?** Yes. Same reason.

**Do I need to sort the coins?** No. The loop order handles combinations; sortedness changes nothing about the count.

**Is the coin-outer version always correct?** Yes, as long as the question asks for combinations. If it asks for sequences — Combination Sum IV — you want the amount in the outer loop, and the answer will be larger.

**My memoized top-down solution is correct but my tabulation isn't. What went wrong?** You translated `f(i, remaining)` into a 2D table and then optimized the loop order without redefining what `dp[j]` means. With coins outside, `dp[j]` stops meaning "using the first `i` coins" and starts meaning "using coins processed so far, in fixed order." Same array, different definition. That's the trap the AlgoMonster discussion thread on this problem keeps circling back to.

**Which language should I write it in?** Whatever you'd write under pressure. The overflow caveat is C++-specific; everything else is identical across languages.

The whole problem comes down to one decision and one line. Coins in the outer loop means combinations. Amount in the outer loop means permutations. Everything else — the base case, the space optimization, the clamping on C++ — follows from that.
