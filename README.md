# min stack: the O(1) getMin trick explained, with working two-stack code in Python and Java

A stack gives you `push`, `pop` and `top` in constant time. Add one more method — `getMin()` — and a structure you've used a thousand times suddenly has a problem. Finding the smallest value by scanning costs O(n) per call. Interviewers want O(1), every call, including after pops that remove the current minimum.

That single constraint is the whole question. Everything else is bookkeeping you have to do correctly, which is why min stack shows up in interview sets for companies including Amazon, Adobe and LinkedIn and why so many otherwise-correct solutions fail on one line.

## The problem, stated plainly

Design a stack supporting four operations, all in O(1) time:

- `push(val)` — add `val` to the top
- `pop()` — remove the top element
- `top()` — return the top element without removing it
- `getMin()` — return the smallest element currently in the stack

The LeetCode version (problem 155) uses `val` in the range −2³¹ to 2³¹−1, guarantees `pop`, `top` and `getMin` are only called on non-empty stacks, and caps the total number of calls at 3 × 10⁴. Note what the constraints do *not* say: nothing about extra memory. That's the door everything walks through.

The naive fix — keep one `min` variable and update it on push — breaks on pop. Push `-2, 0, -3`, and `min` is `-3`. Pop once and the real minimum is `-2`, but your variable still says `-3`. Recomputing it means scanning, which puts you back at O(n). A single variable has no memory of what the minimum *used to be*, and that's the actual difficulty.

## Tracing the canonical example

The standard test case is `push(-2)`, `push(0)`, `push(-3)`, `getMin()`, `pop()`, `top()`, `getMin()` → `-3, 0, -2`.

With a second stack that records "the minimum at this depth," the state evolves like this:

| Call | Main stack | Min stack | Output |
| --- | --- | --- | --- |
| `push(-2)` | `[-2]` | `[-2]` | — |
| `push(0)` | `[-2, 0]` | `[-2]` | — |
| `push(-3)` | `[-2, 0, -3]` | `[-2, -3]` | — |
| `getMin()` | `[-2, 0, -3]` | `[-2, -3]` | `-3` |
| `pop()` | `[-2, 0]` | `[-2]` | — |
| `top()` | `[-2, 0]` | `[-2]` | `0` |
| `getMin()` | `[-2, 0]` | `[-2]` | `-2` |

The pop at depth 3 removes `-3`, and the min stack pops with it — so `-2` reappears as the answer automatically. Nothing is recomputed. That's the trick: each depth carries a snapshot of the minimum as it existed at that moment.

## Approach 1: two stacks, with the pop sync done right

python
class MinStack:
    def __init__(self):
        self.stack = []
        self.min_stack = []

    def push(self, val: int) -> None:
        self.stack.append(val)
        if not self.min_stack or val <= self.min_stack[-1]:
            self.min_stack.append(val)

    def pop(self) -> None:
        if self.stack[-1] == self.min_stack[-1]:
            self.min_stack.pop()
        self.stack.pop()

    def top(self) -> int:
        return self.stack[-1]

    def getMin(self) -> int:
        return self.min_stack[-1]


Two details carry the whole implementation. First, `pop` only touches `min_stack` when the value leaving the main stack equals the current minimum — otherwise the min stack stays put. Second, `push` uses `<=`, not `<`.

> If you push the same minimum value twice, `<=` records it twice. With `<`, popping one copy removes the minimum from the min stack even though another copy is still sitting in the main stack, and `getMin()` starts returning the wrong number on the next call.

An equally valid variant keeps both stacks the same height. Instead of pushing only when records are broken, every push puts `min(val, min_stack.top())` onto the min stack, and every pop removes from both. It uses more memory and simpler logic — no equality check in `pop` at all. Java's `ArrayDeque` version usually adds a sentinel to avoid empty-stack checks:

java
class MinStack {
    private Deque<Integer> stack = new ArrayDeque<>();
    private Deque<Integer> minStack = new ArrayDeque<>();

    public MinStack() {
        minStack.push(Integer.MAX_VALUE); // removes null checks in push()
    }

    public void push(int val) {
        stack.push(val);
        minStack.push(Math.min(val, minStack.peek()));
    }

    public void pop() {
        stack.pop();
        minStack.pop();
    }

    public int top() { return stack.peek(); }

    public int getMin() { return minStack.peek(); }
}


This is the shape you'll find in AlgoMonster's editorial for the problem — both the Python and Java versions use the two-stack approach, with the sentinel handling the empty case. If you want to compare your line-by-line against a written walkthrough, 👉 [read AlgoMonster's Min Stack editorial and full solution](https://algo.monster?fpr=li69).

## Approach 2: one stack of pairs

Instead of two parallel stacks, store a pair at each position: the value, and the minimum from the bottom up to that position.

python
def push(self, val):
    current_min = min(val, self.stack[-1][1]) if self.stack else val
    self.stack.append((val, current_min))


`getMin()` returns `stack[-1][1]`, `top()` returns `stack[-1][0]`, and `pop()` is a plain pop. There's no synchronization to get wrong, which is why this version is easier to explain out loud in an interview. The cost is memory: every element now stores two integers instead of one, even when the minimum never changes. The two-stack version above only pushes to the min stack when a new minimum or a tie appears, so on input like `1, 5, 8, 3, 7` it stores far less.

Neither approach is "the right one." Pair-per-element is the safer answer under pressure; two-stack-with-conditional-push is the one that shows you understand the space tradeoff.

## Approach 3: one stack, difference encoding

The space-optimal variant stores no second stack at all. Each entry holds the difference between the pushed value and the minimum at the time of the push, and a single variable tracks the current minimum.

python
class MinStack:
    def __init__(self):
        self.stack = []
        self.min = float("inf")

    def push(self, val):
        if not self.stack:
            self.stack.append(0)
            self.min = val
        else:
            self.stack.append(val - self.min)
            if val < self.min:
                self.min = val

    def pop(self):
        diff = self.stack.pop()
        if diff < 0:           # the element being removed *was* the minimum
            self.min = self.min - diff   # restore the previous minimum

    def top(self):
        diff = self.stack[-1]
        return self.min if diff < 0 else diff + self.min

    def getMin(self):
        return self.min


A negative encoded value means "this entry was the minimum when it was pushed." That's how `pop` can recover the previous minimum without any extra storage. Extra space here is O(1) beyond the values themselves, and it's genuinely clever.

It's also the version most likely to bite you in a fixed-width language. In Java, `val - min` can overflow an `int` when the values sit near `Integer.MIN_VALUE` and `Integer.MAX_VALUE` — the LeetCode constraints explicitly allow that range. Use `long` for the encoded values if you go this route. Python's arbitrary-precision integers make the overflow question disappear, which is part of why this variant circulates mostly in Python discussions.

## Complexity, side by side

| Approach | push | pop | top | getMin | Extra space |
| --- | --- | --- | --- | --- | --- |
| Scan on `getMin` | O(1) | O(1) | O(1) | O(n) | none |
| Two stacks (conditional push) | O(1) | O(1) | O(1) | O(1) | up to O(n) |
| Two stacks (equal height) | O(1) | O(1) | O(1) | O(1) | up to O(n) |
| Pair per element | O(1) | O(1) | O(1) | O(1) | O(n), 2 ints per entry |
| Difference encoding | O(1) | O(1) | O(1) | O(1) | O(1) extra |

"O(1) extra space" in the last row doesn't mean the structure is free — the values still occupy O(n). It means no second array, and the classic follow-up interviewers ask here ("can you do it without another stack?") has a real answer.

## The mistakes that cost people the offer

Four failure modes account for most wrong answers:

**Calling `min()` inside `getMin()`.** Push all elements into one list and return `min(self.stack)` and you've written an O(n) operation behind an O(1) label. It passes the sample case and fails the point of the question.

**Dropping the `<=` on push.** Covered above, but worth repeating because it only shows up when the same minimum is pushed twice — exactly the test case a careful interviewer adds.

**Unsynchronized pops.** If `pop` removes from the main stack but forgets the min stack, the two drift apart. Every subsequent `getMin()` is wrong, usually returning a value that has already left the stack.

**No empty-stack plan.** In Java, an `ArrayDeque` peek on an empty deque throws. The LeetCode signature guarantees `pop`, `top` and `getMin` are called on non-empty stacks, but a sentinel like `Integer.MAX_VALUE` in the min stack makes the code robust without scattering null checks, and costs one integer.

Interviewers also ask for the reasoning out loud, not just the code. Being able to say "the minimum at depth *k* never changes once it's recorded, so I can store it at push time and read it back at constant cost" is the answer they're listening for.

## Where this question fits in the wider pattern set

Min stack is a *design* question wearing a data-structure costume. The reusable idea — augment a standard structure with the derived value you'll need later — reappears constantly:

- **Max stack** — identical problem, `getMax()` instead, plus a `popMax()` variant that needs more thought
- **Queue with max** — the same instinct, but the natural tool is a monotonic deque
- **LRU cache** — a hash map plus a doubly linked list, two structures cooperating
- **Min stack with O(1) time and O(1) extra space** — the difference-encoding follow-up
- **Monotonic stack problems** (next greater element, largest rectangle in histogram) — related vocabulary, different technique; the stack here holds a sorted run of candidates rather than tracking a global minimum

That last distinction trips people up. A "monotonic stack" and a "min stack" aren't the same tool, and interviewers notice when you reach for the wrong one. If what's actually missing is the ability to pick the right structure from the problem statement rather than the code itself, that's a pattern-recognition gap, which is the specific thing platforms like AlgoMonster are built around — 👉 [see how AlgoMonster organizes stack and design problems by pattern](https://algo.monster?fpr=li69).

## Practicing min stack specifically

The problem itself needs maybe 20 minutes of study. The useful question is what you do with the 30 variations that follow it.

AlgoMonster's free Lite Problems section includes the Min Stack editorial with the two-stack implementations in Python and Java, so you can read the reference solution without paying. Beyond that, the Pro portion adds the pattern lessons these problems hang off, Speedrun for timed review, an inline editor for each problem, company-specific question banks, and an AI assistant that explains selected platform text or debugs a submitted solution. Solutions are available in Python, Java, JavaScript, C++, Go and Haskell.

What it doesn't have matters too. Reviews are consistent about three gaps: there's no video content at all, system design coverage is introductory and not enough for senior loops, and the problem count is deliberately small — third-party writeups describe the core curriculum as roughly 48 patterns and around 231 lessons and problems, against LeetCode's 3,000+. AlgoMonster's own subscription page advertises larger figures (500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials), which counts the bonus course content alongside the interview patterns. Both numbers can be true of different things; the honest reading is that the pattern curriculum is compact and the surrounding course library is broader. Several review-writers also mention a job-guarantee claim attached to completing the curriculum — the terms are on the site, so read them rather than assuming.

## AlgoMonster pricing: what's actually on the checkout page

There's no monthly plan and no team plan. The free tier gives limited access to sample patterns and lessons. Paid access is annual or lifetime:

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited sample of patterns and lessons, enough to read editorials like Min Stack | $0 | — | [Browse the free problems](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company question banks; 500+ lessons across 6 courses; bonus Foundation and Build-an-LLM courses; member discount on live bootcamps | $99/year (shown as $8.25/mo billed annually) | Annual, cancel or upgrade anytime | [Start the yearly plan](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan, plus lifetime updates, billed once | $189 | One-time payment | [Get lifetime access](https://algo.monster?fpr=li69) |

Two things worth knowing before you buy. The page displays those prices against struck-through list prices of $198/year and $378, so what you see is already a promotional rate — and promotions rotate. Different snapshots of the same page have shown the yearly plan as low as $69.99, so the number in your own checkout is the only current one. Second, the Quant Interview Prep course developed with external partners is sold separately and is not part of either plan.

## Is it worth paying for if you searched "min stack"?

Handle this honestly, because the answer depends on what you actually need.

If you want to understand this one problem, you don't need a subscription. The two-stack solution above works, the free editorial is readable, and LeetCode will grade your submission. Save the money.

Pay for the yearly plan if you're within a few weeks of interviews and your bottleneck is selection speed — sitting in front of a problem and not knowing whether it wants a stack, a heap or a two-pointer sweep. The pattern framework is aimed exactly at that gap, and $99 sits below a year of LeetCode Premium at $159.

Pay for lifetime if you expect to interview more than once in the next few years. At $189 against $99/year, it pays for itself before the end of a second annual cycle, which matters for engineers who job-hop every two to three years and re-prep each time.

Skip it entirely if you learn best from video, if you're targeting senior roles where system design is a full interview round, or if you already recognize patterns and just need volume. In those cases the money is better spent elsewhere — AlgoMonster's own reviewers say as much, which is a point in their favor.

## Quick answers to the follow-ups

**Does min stack need O(n) extra space?** For the two-stack and pair-based versions, yes — up to n extra integers, though the conditional-push variant stores fewer when the minimum rarely changes. The difference-encoding version uses O(1) extra space but carries an overflow risk in fixed-width integer languages.

**Can I just sort the stack?** No. Sorting a stack costs more than O(1) per operation, and it destroys the LIFO order that `pop()` and `top()` depend on.

**How hard is it in an interview loop?** It's rated Medium on LeetCode. The concept is easy; the edge cases are where people lose it.

**Is a min stack the same as a monotonic stack?** No. A min stack tracks the global minimum at each depth. A monotonic stack maintains a deliberately sorted sequence of candidates and is used for things like next-greater-element problems. Different tools, similar names.

**Why does `pop` need to check equality with the min stack top?** Because that's the only signal that the departing element was the minimum. Without the check, the min stack either drifts out of sync or loses minimums that are still present at other depths.

Get the sync right, use `<=` on push, and know why the snapshot works — that's the entire problem, and it's the same reasoning you'll reuse the next time an interviewer asks you to make a familiar structure answer an unfamiliar question in constant time.
