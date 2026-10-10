# majority element: the O(n) Boyer-Moore trick, why it survives, and the n/3 follow-up interviewers ask next

Two kinds of people end up searching for "majority element." The first hit LeetCode 169 in a study plan and want to know why sorting the array and returning the middle element works. The second got the question in a real interview, produced a hash map answer, and then got hit with: "Can you do it in constant space?" followed by "What if I remove the guarantee that a majority exists?"

Both groups need the same thing — not a working solution, but an explanation of *why* the answer survives elimination. Here's the whole picture: the four approaches worth knowing, a step-by-step trace, the edge cases that break sloppy code, and the follow-up questions that come right after.

## What the problem actually asks

Given an array `nums` of size `n`, the majority element appears **more than ⌊n/2⌋ times**. LeetCode 169 guarantees one exists, so you never handle the "no answer" case.

- `nums = [3,2,3]` → `3`
- `nums = [2,2,1,1,1,2,2]` → `2`

Constraints: `1 <= n <= 50,000`, values from `-10^9` to `10^9`.

Two details carry more weight than they look like they do:

**The comparison is strict.** Something appearing exactly `⌊n/2⌋` times is not a majority. In `[1,2,2,3]`, the value `2` shows up twice, `⌊4/2⌋ = 2`, and `2` is not a majority. Half the array is not enough — you need more than half.

**At most one element can qualify.** Two elements would each need more than `n/2` occurrences, which adds up to more than `n` slots in an array of size `n`. That's why 169 returns a single integer, and it's also why the harder version (appearing more than `n/3` times) can return at most two values.

## Four approaches, and what each one actually costs

| Approach | Time | Space | When it's the right answer |
| --- | --- | --- | --- |
| Brute force, count each element against the rest | O(n²) | O(1) | Only as a starting point you immediately improve on |
| Hash map of counts | O(n) | O(n) | The default first answer; exit early once a count passes `n//2` |
| Sort and return `nums[n//2]` | O(n log n) | O(1) | Shortest working code, and it surprises people that it's correct |
| Boyer-Moore voting | O(n) | O(1) | The one the interviewer is fishing for |
| Random sampling | O(n) expected | O(1) | Worth mentioning; any random pick is the majority with >1/2 probability |

### Hash map: honest and boring

python
def majorityElement(nums):
    counts = {}
    for num in nums:
        counts[num] = counts.get(num, 0) + 1
        if counts[num] > len(nums) // 2:
            return num


Linear time, linear space, exits early. Nobody will be impressed, and it's still the right thing to write first so the interviewer sees you have a baseline before you optimize.

### Sorting: correct for a reason worth stating

python
def majorityElement(nums):
    nums.sort()
    return nums[len(nums) // 2]


Sorting groups all copies of the majority into one contiguous block. A block occupying more than half the array is forced to cover the midpoint index `n//2` wherever it starts. That's the entire proof, and saying it out loud is better than saying "it just works."

Cost is O(n log n) time. Extra space is O(1) if you sort in place with a non-recursive sort, O(log n) if your language's sort recurses.

### Boyer-Moore: two variables, one pass

python
def majorityElement(nums):
    candidate, count = None, 0
    for num in nums:
        if count == 0:
            candidate = num
        count += 1 if num == candidate else -1
    return candidate


java
public int majorityElement(int[] nums) {
    int candidate = 0, count = 0;
    for (int num : nums) {
        if (count == 0) candidate = num;
        count += (num == candidate) ? 1 : -1;
    }
    return candidate;
}


One pass, two scalars, no auxiliary structure. That's the answer to "constant space."

## Tracing it on [2,2,1,1,1,2,2]

The array has seven elements, so a majority needs more than 3 occurrences. Element `2` appears 4 times, at indices 0, 1, 5, 6.

| Step | Element | count before | Action | candidate | count after |
| --- | --- | --- | --- | --- | --- |
| 1 | 2 | 0 | count is 0, adopt candidate | 2 | 1 |
| 2 | 2 | 1 | matches candidate, increment | 2 | 2 |
| 3 | 1 | 2 | differs, decrement | 2 | 1 |
| 4 | 1 | 1 | differs, decrement | 2 | 0 |
| 5 | 1 | 0 | count is 0, adopt candidate | 1 | 1 |
| 6 | 2 | 1 | differs, decrement | 1 | 0 |
| 7 | 2 | 0 | count is 0, adopt candidate | 2 | 1 |

Final candidate: `2`.

Note what step 5 does. The correct value temporarily *loses* the candidacy to `1`. That's the part that makes the algorithm feel like a magic trick instead of a counting scheme — and it's usually the part an interviewer pokes at. The answer is that the counter isn't tracking a lead; it's tracking unmatched copies.

A look at AlgoMonster's [free write-up of LeetCode 169](https://algo.monster/liteproblems/169?fpr=li69) shows the same trace laid out element by element, which is handy if you want to watch the counter move on a different input.

👉 [See the step-by-step trace and the rest of the pattern](https://algo.monster/liteproblems/169?fpr=li69)

## Why the candidate survives

Think of it as repeated cancellation. Take any two distinct elements in the array and delete both. Each deletion removes at most one copy of the true majority `m` — because the other element of the pair isn't `m` — while removing one non-`m`. Since `m` holds more than half the slots, you can never cancel every copy. Pair off elements until no two distinct ones remain, and whatever is left is `m`.

Boyer-Moore is that cancellation performed in a single streaming pass. `count` is the number of unmatched copies of the current candidate. When `count` hits zero, all previous copies have been paired off, so the next element starts a fresh group. At the end, the surviving candidate is the majority.

The streaming framing isn't a metaphor. The algorithm came out of Boyer and Moore's 1981 paper "MJRTY — A Fast Majority Vote Algorithm," which was about finding a majority in a sequence with constant working memory.

## Where the algorithm lies to you

Boyer-Moore guarantees a correct answer *only when a majority exists*. Given `[1,1,2,2,3]`, it walks through the whole array and hands you `3` — the last candidate standing, and not the majority of anything.

So if the interviewer removes the guarantee ("return the element or `-1` if there is none"), you add a verifying pass:

python
def majorityElementOrNone(nums):
    candidate, count = None, 0
    for num in nums:
        if count == 0:
            candidate = num
        count += 1 if num == candidate else -1
    return candidate if nums.count(candidate) > len(nums) // 2 else None


Still O(n) time and O(1) space, two passes. Hashing a stream twice is not an option; scanning it twice is. This is a good answer to give before the interviewer finishes describing the variation.

## Five ways people break the implementation

1. **Using `>=` in the verification.** `[1,2,2,3]` has no majority, but `count >= n/2` accepts `2`. Strictly greater, always.
2. **Floating-point division.** In Python 3, `n / 2` returns a float while `n // 2` returns an int; in JavaScript you need `Math.floor(n / 2)`. Java's integer division is already floor for positive values. The bug shows up in the `n % 2 == 1` cases, which is exactly when you're testing manually with small arrays.
3. **Mixing the two update styles.** `if count == 0: candidate = num; count = 1` and `count += 1 if num == candidate else -1` are both correct, but you have to pick one. Blending them resets the counter twice or lets it drift negative, and the candidate ends up wrong on inputs like `[1,2,3]`.
4. **Assuming it returns the most frequent element.** It returns a strict majority if one exists. `[1,1,2,2,3]` has no majority; the mode is a tie; the algorithm still returns something.
5. **Reusing index-based counting.** The counting-sort trick works only if you know the value range is small and non-negative. Here values run to ±10⁹, so that shortcut is off the table.

## The n/3 follow-up: Majority Element II

The natural escalation is LeetCode 229: return every element appearing more than ⌊n/3⌋ times. At most two elements can clear that bar (three would exceed `n`), so you keep two candidates and two counters.

python
def majorityElementII(nums):
    cand1 = cand2 = None
    c1 = c2 = 0
    for num in nums:
        if num == cand1:
            c1 += 1
        elif num == cand2:
            c2 += 1
        elif c1 == 0:
            cand1, c1 = num, 1
        elif c2 == 0:
            cand2, c2 = num, 1
        else:
            c1 -= 1
            c2 -= 1
    res = []
    for c in (cand1, cand2):
        if c is not None and nums.count(c) > len(nums) // 3:
            res.append(c)
    return res


Two things trip people up. The order of the `if` checks matters — you must test the current element against both existing candidates *before* filling an empty slot, otherwise a third distinct value overwrites a live candidate. And the verification pass is mandatory, not optional: unlike 169, 229 gives no guarantee that anything qualifies, so both survivors have to be counted to confirm.

The counting itself is cheap — two `count` calls are O(n) each, so the whole thing stays linear.

## What interviewers ask after the code works

**"What if the input is a stream?"** Then sorting is gone and a hash map is off the table for memory reasons. Boyer-Moore still works, in one pass, with constant space — which is the point of the algorithm. Note the tradeoff: a single pass gives you a candidate, not a verified answer. You need a second pass to confirm, or you accept the candidate as the answer because the problem guarantees one exists.

**"Generalize it to elements appearing more than n/k times."** That's the Misra-Gries summary: keep `k-1` candidate slots with counters, and when an incoming element matches none of them and no slot is empty, decrement every counter instead. Boyer-Moore is the `k = 2` case wearing a different name. This shows up in rate limiting, log analysis, and anything described as "heavy hitters."

**"Solve it with divide and conquer."** The problem carries that tag on LeetCode for a reason. Recursively find the majority of each half; if the two agree, return that value; if they disagree, count both within the current range and return whichever exceeds half of the range's length. O(n log n) time — worse than Boyer-Moore, and still a fair thing to be asked to derive.

**"Why does the sorted middle element always work?"** Answer with the block argument above, not with "because it's the median."

## Where this fits a study plan

A single problem like this teaches a pattern: cancellation-based greedy selection. The same idea reappears in Majority Element II, in Misra-Gries, and in problems where you're told to find something that "survives" pairwise elimination. Learning it as a named pattern rather than as one solution you memorized is what makes the n/3 variant tractable the first time you see it under time pressure.

AlgoMonster organizes its material that way — patterns, flowcharts for choosing between them, and template code — and it's built by ex-Google engineers and competitive programmers. Here's the current pricing straight from its subscription page:

| Plan | What you get | Price | Billing period | Get access |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems | $0 | — | [Explore the free content first](https://algo.monster?fpr=li69) |
| Pro – Yearly | All patterns, lessons and problems, unlimited AI assistant, unlimited Speedrun, company questions refreshed monthly, bonus courses | $99/year (shown as $8.25/mo billed annually, about $0.28/day) | 12 months, cancel or upgrade anytime | [Start the yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates and an internal referral to a top tech company for qualified candidates | $189 | Charged once, next to nothing amortized over a career | [Get lifetime access](https://algo.monster/subscribe?fpr=li69) |

Two practical notes on that table. The lifetime tier is the same content as the annual one — the difference is whether updates in future years cost you anything, which matters if you expect to interview again in two or three years rather than once. And the subscription includes bonus courses (CS fundamentals, a build-an-LLM course) at no extra cost, while some partner-developed material, like the Quant Interview Prep course, is sold separately.

The honest limits, based on what's on the platform and what third-party reviewers keep flagging: it's text and diagrams, with no video walkthroughs, and the practice bank is in the low hundreds of problems rather than the thousands LeetCode carries. System design coverage is introductory. If your gap is pattern recognition and you have four to eight weeks, that format suits the job. If your gap is raw volume, or you need deep system design prep for a senior loop, budget for a second resource.

👉 [Compare both plans on the subscription page](https://algo.monster/subscribe?fpr=li69)

## The short version

Write the hash map first to show you have a baseline. Move to sorting if you want the shortest correct code. Land on Boyer-Moore when the interviewer asks about space, and explain it through cancellation rather than through the counter mechanics alone. Then say the sentence that separates a memorized answer from an understood one: *the candidate is correct because a majority cannot be fully cancelled, and if the problem stopped guaranteeing a majority, I'd add a second pass to verify.*

That last sentence is also what prepares you for the follow-up at n/3, where the guarantee is gone and the verification pass is where most submissions actually fail.
