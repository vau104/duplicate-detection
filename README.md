# Find the Duplicate Number: Binary Search vs Floyd's Cycle Detection, With Python and C++ Code and the Proof Interviewers Ask For

Almost everyone solves LeetCode 287 in about ninety seconds. You write a set, you return the first value you've seen before, you hit submit. Then you scroll back up and read the constraints: the array is read-only, and extra space must be constant. The solution you just wrote does neither.

That gap between "obvious answer" and "allowed answer" is what makes this problem worth studying. It's a Medium that behaves like a Hard the moment an interviewer asks you to implement it under the stated rules, and it's one of the cleanest places to learn the fast-and-slow-pointer pattern, because the array hides a linked list inside it.

## What the constraints actually rule out

Before writing a line of code, it helps to know which doors the problem has already closed.

The array `nums` holds n + 1 integers, every value is in the range **[1, n]**, and exactly one value repeats (it may appear three or five times, not just twice). You must not modify `nums`, you may use only O(1) extra space, and your runtime should be better than O(n²).

That combination kills three otherwise reasonable solutions:

- A hash set or frequency array works in O(n) time but costs O(n) space.
- Sorting works, but reorders the input, and most library sorts also use extra memory internally.
- "Negative marking" — flipping `nums[abs(v)]` to a negative sign to flag values you've seen — runs in O(n) time and O(1) space, but it mutates the array. Some variants let you restore it afterward; here the read-only rule means it's out.

What's left is two approaches that satisfy everything: binary search on the *value range*, and Floyd's tortoise-and-hare cycle detection. They're worth understanding separately, because interviewers tend to push on whichever one you didn't pick.

## The problem, with the examples you'll be walked through

| Input | Output | Notes |
| --- | --- | --- |
| `[1,3,4,2,2]` | `2` | 2 appears twice |
| `[3,1,3,4,2]` | `3` | the duplicate is not adjacent to itself |
| `[3,3,3,3,3]` | `3` | one value, five times |

Constraints to keep in mind while testing: `n` goes up to 10⁵, so an O(n²) double loop is out, and `nums.length == n + 1` always. The answer is a **value**, never an index — which sounds trivial until you watch someone return `slow` after thinking of it as a position in the array.

## Approach 1: hash set, the answer that fails the rules

python
class Solution:
    def findDuplicate(self, nums: list[int]) -> int:
        seen = set()
        for num in nums:
            if num in seen:
                return num
            seen.add(num)
        return -1


Time: O(n). Space: O(n). It's the right first thing to say out loud in an interview — "I can do this in linear time with a set, but that violates the constant-space requirement, so let me think about what the values being in [1, n] buys me." Interviewers tend to reward that framing more than an immediate clever answer.

## Approach 2: binary search on the value range

Here's the observation that makes this work. If the array held n distinct numbers from 1 to n, then for any `x`, exactly `x` of them would be ≤ `x`. Count 1, 2, 3 to 3 and you get three numbers. Now introduce a duplicate: at some point the count of values ≤ `x` exceeds `x`, and it stays ahead for every `x` at or above the duplicate.

That gives you a monotone predicate — false, false, ..., true, true — which is exactly the shape binary search is built for. You binary search the *value range* [1, n], not the indices, and count on each step.

python
class Solution:
    def findDuplicate(self, nums: list[int]) -> int:
        left, right = 1, len(nums) - 1
        while left < right:
            mid = (left + right) // 2
            count = sum(1 for x in nums if x <= mid)
            if count > mid:   # excess in [1, mid] → duplicate is ≤ mid
                right = mid
            else:
                left = mid + 1
        return left


Time: O(n log n) — log n iterations, each scanning the array. Space: O(1). It satisfies every constraint, and it's the version AlgoMonster's own write-up of this problem builds around, using their "find the first true" template where the feasible function is `count(≤ x) > x`.

The tradeoff is speed: for n = 10⁵ you're doing roughly 17 full scans. Fine in practice, and it beats Floyd's asymptotically nowhere. Which brings us to the answer most interviewers are hoping you'll reach.

## Approach 3: Floyd's tortoise and hare, applied to an array

The trick is to stop thinking of `nums` as an array and start treating it as a function. From index `i`, the next node is `nums[i]`. Every value is in [1, n], so every value is a legal index. Follow that mapping from index 0 and you've built a linked list you never had to allocate.

Two facts make this work:

**A cycle must exist.** There are n + 1 starting positions but only n possible destinations, so by the pigeonhole principle two different indices map to the same value. Once two paths converge, they never separate again — from that node on, the walk is identical. That node is the start of the cycle, and the value at it is your duplicate.

**Index 0 sits outside the cycle.** No value is smaller than 1, so nothing ever points back to index 0. That's convenient: a walk starting at index 0 always travels some distance and then enters the loop, rather than starting somewhere inside it.

Phase 1 sends a slow pointer one step and a fast pointer two steps until they collide somewhere inside the cycle. Phase 2 resets one pointer to the start and moves both one step at a time; wherever they meet is the cycle entry, which is the duplicate.

python
class Solution:
    def findDuplicate(self, nums: list[int]) -> int:
        slow = fast = nums[0]

        # Phase 1: find a meeting point inside the cycle
        while True:
            slow = nums[slow]
            fast = nums[nums[fast]]
            if slow == fast:
                break

        # Phase 2: find the entrance to the cycle
        slow = nums[0]
        while slow != fast:
            slow = nums[slow]
            fast = nums[fast]

        return slow


cpp
class Solution {
public:
    int findDuplicate(vector<int>& nums) {
        int slow = nums[0], fast = nums[0];
        do {
            slow = nums[slow];
            fast = nums[nums[fast]];
        } while (slow != fast);

        slow = nums[0];
        while (slow != fast) {
            slow = nums[slow];
            fast = nums[fast];
        }
        return slow;
    }
};


Time: O(n). Space: O(1). Array untouched.

### Why phase 2 lands on the duplicate, not just anywhere

This is the part people memorize and then can't defend. Set `F` = number of steps from index 0 to the cycle entry, and `C` = the cycle length. Suppose the pointers meet `x` steps into the cycle. Slow has walked `F + x`; fast has walked twice that, and because it's been lapping the cycle, it's also at position `F + x + mC` for some positive integer `m`. So:


2(F + x) = F + x + mC   →   F + x = mC   →   F = mC − x


Now restart a pointer at index 0. After `F = mC − x` steps it stands exactly on the cycle entry. The pointer still sitting `x` steps into the cycle, walking the same `F` steps, lands at position `x + (mC − x) = mC` — the entry again. They arrive at the same node by construction, and that node is the duplicate value. No magic, just modular arithmetic.

## Side by side

| Approach | Time | Extra space | Meets all constraints? |
| --- | --- | --- | --- |
| Hash set / counting | O(n) | O(n) | No — space |
| Sort and scan neighbors | O(n log n) | O(log n) or O(n) | No — mutates input |
| Negative marking | O(n) | O(1) | No — mutates input |
| Binary search on value range | O(n log n) | O(1) | Yes |
| Floyd's cycle detection | O(n) | O(1) | Yes |

## Mistakes that cost you the round

**Returning an index instead of a value.** `slow` is a value here because array values *are* indices, but the variable is easy to misread. Trace `[3,1,3,4,2]` by hand once and the distinction stops feeling academic.

**Starting the pointers at 0 and expecting the same code path to work.** Initializing `slow = fast = nums[0]` and initializing both at index 0 are both correct — they just differ by one iteration. Mixing the two patterns, however, means your phase-1 loop exits at the wrong moment, or you reset the wrong pointer.

**Reaching for a set and forgetting why it's disallowed.** Saying "I'll just use a hash set" without acknowledging the space constraint is the single most common way candidates lose points on this problem.

**Confusing it with "find all duplicates."** LeetCode 442 lets you mutate the array and asks for every repeated value. Here you return one number and never touch the input.

## The follow-ups interviewers actually ask

*Prove a duplicate must exist.* n + 1 values drawn from [1, n] into n buckets — pigeonhole, said in one sentence.

*Can you do it in linear time?* Yes, Floyd's. This is the follow-up the problem statement itself hints at, and it's your cue to explain the cycle-entry math above.

*What if the array streams and can't be revisited?* Floyd's algorithm only ever follows `nums[nums[i]]`, so it's a natural fit — but you now need random access, which streaming doesn't give you. Worth saying out loud as a limitation rather than guessing.

*What if there are multiple duplicates?* Floyd's breaks down, because more than one node can have two incoming edges. You're back to binary search with a modified predicate, or external memory.

## Practicing the pattern instead of the problem

287 is one of a family. Middle of the Linked List, Linked List Cycle, Linked List Cycle II, Happy Number, and this problem all reduce to "two pointers moving at different speeds over a structure that loops." Recognizing that the array *is* a linked list is the actual skill being tested — the code is fifteen lines once you see it.

That's the structural argument behind pattern-based prep, and it's where AlgoMonster's focus lies: 48 named patterns with decision flowcharts, template code in multiple languages, AI-assisted explanations, Speedrun mode for pre-interview review, and company-tagged question sets. Its 287 walkthrough frames the problem as a first-true binary search with `count(≤ x) > x` as the feasible condition, which is a good test of whether a resource matches how you think — if you were expecting a cycle-detection writeup, you'll notice immediately. Several of the Lite Problems editorials, 287 included, are readable without an account, so that's the cheapest way to check whether the teaching style fits you before paying.

Also worth knowing before you buy: the content is text and diagrams, with no video; system design coverage is introductory rather than senior-level; and there's no monthly billing option, only annual or lifetime.

| Plan | What you get | Price | Billing |
| --- | --- | --- | --- |
| Free | Selected Lite Problems editorials and sample lessons, limited access | $0 | No card required |
| Pro — Yearly | All patterns, lessons, and problems; unlimited AI assistant; unlimited Speedrun; real questions from top companies updated monthly; 12 months of access | $99/year ($8.25/month) | Annual, with a 3-day free trial [ start the yearly plan](https://algo.monster/subscribe?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates and an internal referral for qualified candidates | $189 | One-time payment [ get lifetime access](https://algo.monster/subscribe?fpr=li69) |

The math is simple enough to do yourself. If you're prepping for one interview cycle in the next two months, the annual plan covers it. If you expect to interview again in two years — promotions, a company change, a team that pivots — the lifetime plan costs $90 more than a single year and doesn't expire. What neither plan fixes is the thing 287 actually tests: spending 15 of your 45 minutes cycling through approaches that don't apply. That's recognition speed, and it comes from reps.

## A short answer for the next person who asks

If you only remember three sentences: a set solves it but breaks the space rule, binary search over the value range satisfies every constraint in O(n log n), and Floyd's two-pointer walk hits O(n) by treating values as pointers into a hidden cycle. The only real trap is that the answer is a value, not a position.

If you want the full walkthrough with the tested code and a pattern map for the rest of the family, [👉 browse AlgoMonster's Fast & Slow Pointers material](https://algo.monster/liteproblems/287?fpr=li69) — or just solve it once more by hand on `[3,1,3,4,2]` and you'll own it.
