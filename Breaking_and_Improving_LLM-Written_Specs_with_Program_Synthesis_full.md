# Breaking and Improving LLM-Written Specs with Program Synthesis

**ECE1724H F3 — Program Synthesis and Verification, Fall 2026 — Project Proposal**

**Team:** Yicheng Yao (`yicheng.yao@mail.utoronto.ca`), Jarvis Wang (`jarvis.wang@mail.utoronto.ca`)

## 1. Problem and Motivation

Formal verification can prove that a program matches its specification, but it cannot tell us whether the specification says what we actually meant. If the spec is wrong, a "verified" program can still be wrong. Specs usually go wrong in one of four ways:

- **Too weak.** The spec accepts wrong programs. If `min(a, b)` is specified only as `r <= a && r <= b`, then `return a - 1` passes verification, even though it is clearly not `min`.
- **Too strict on inputs.** The precondition rules out inputs that should be allowed. If the spec adds `a != b`, the case `a == b` is never checked.
- **Empty (vacuous).** The spec says nothing at all, either because no input satisfies the precondition or because the postcondition holds for every output. Every program passes.
- **Too strong.** The spec rejects the correct program, so verification fails even though the code is fine.

None of this is new to hardware verification engineers. They call these problems *vacuous passes* and *over-constrained assumptions*, and they rely on cover properties and vacuity checks to catch them. Software verification has no standard tool that plays the same role.

The problem is becoming more pressing because LLMs are now being asked to write specs, either by turning plain-English requirements into formal postconditions or by writing specs alongside code. An LLM-written spec can look reasonable and pass every syntax check, yet still be too weak. Today these specs are mostly checked against a handful of test cases and a few slightly changed ("mutated") versions of the correct program. That approach misses any weakness the tests happen not to cover, and a failed test gives the LLM very little to go on when it tries to fix the spec.

**Our idea is to attack the spec.** We turn program synthesis around: instead of synthesizing a correct program, we synthesize a *wrong* program that still passes the spec. We ask an LLM to write such a program and use the Z3 solver to confirm that it really does pass the spec and really is wrong. If we can find one, the spec is definitely too weak, and the program itself is a readable example of the problem, such as "your spec accepts `return a - 1`". We then hand that example back to the spec-writing LLM so it can fix the spec.

## 2. Approach

### 2.1 Tasks and the spec format

Each task in our benchmark has three parts, all written by hand:
- a plain-English **description**, such as "Return x limited to the range [lo, hi]. You may assume lo <= hi";
- the **allowed inputs**: the input types (integers, or fixed-length integer arrays) plus any assumptions stated in the description, here `lo <= hi`;
- a **golden spec**: a trusted postcondition that says which outputs are correct.

We write the allowed inputs and the golden spec ourselves and never take them from the LLM, since they are the ground truth that the LLM's spec is measured against.

How we write the golden spec depends on the task. For tasks with **one correct answer**, such as min or clamp, we write a **golden program**: a short expression Z3 can reason about, such as `If(x < lo, lo, If(x > hi, hi, x))`. The intended output for each input is whatever the golden program returns, and the golden spec is just "the result equals the golden program's output". For tasks with **many correct answers**, such as "return any number larger than both inputs", there is no single right output and therefore no golden program, so we write the golden spec directly (`r > a and r > b`). Any output it allows counts as an intended output.

Before using a golden program or golden spec, we test it on a few hand-picked inputs (or, for golden specs, outputs that should be accepted or rejected), and the other team member reviews it. The golden spec and golden program are needed for this project so that we can tell whether each LLM-written spec is correct and evaluate our approach. In real work they are usually not available; we discuss this case in Section 4.2.

**The LLM's spec.** The LLM writes a precondition and a postcondition as small Python functions that build Z3 formulas. For min, a good answer looks like this:

```python
def pre(a, b):     return True
def post(a, b, r): return And(r <= a, r <= b, Or(r == a, r == b))
```

**Checking the LLM's output before we use it.** Since the spec arrives as Python code, we have to run it to use it, and running LLM-written code blindly is risky for three reasons. First, it could harm the machine: the code might import modules, delete files or make network calls, usually by accident. Second, it might not be a spec at all: the LLM could write loops, call `min()` or `sorted()`, or print things, and Z3 cannot reason about any of that, so our checks would crash or quietly check the wrong thing. Third, it might use operations Z3 handles poorly, such as multiplying two variables, which can cause timeouts or "unknown" answers and add noise to the results.

To avoid this, we parse the code with Python's `ast` module *without running it* and accept it only if it sticks to a small set of logical, arithmetic and array operations. That way every spec we analyze is safe to run and is a formula Z3 can check. The programs written by the attacker (Section 2.3) go through the same check.

Specs that fail this step are counted as a separate kind of failure ("did not follow the format"). This keeps formatting mistakes apart from real spec-quality problems, and also tells us how often the model fails to follow instructions at all.

### 2.2 Checking the spec

Every spec goes through up to five checks. The first four are single questions to Z3. For three of them, a **sat** answer (Z3 found values) means there is a problem, and the values Z3 found are our example. The empty-precondition check works the other way around. In the table, *golden* refers to the golden spec, which for one-answer tasks simply means "equals the golden program's output".

| Check | What we give Z3 | Z3 returns **sat** (found) | Z3 returns **unsat** (none exist) |
|---|---|---|---|
| Empty precondition | allowed inputs **and** `pre` | ✅ fine: some allowed input satisfies `pre` | ❌ **empty**: no allowed input satisfies `pre` |
| Too strict on inputs | allowed inputs **and not** `pre` | ❌ **too strict**: an allowed input that `pre` rules out | ✅ fine |
| Too strong | allowed inputs **and** `pre` **and** golden accepts `r` **and not** `post(r)` | ❌ **too strong**: an input where an intended output is rejected | ✅ fine |
| Too weak (single output) | allowed inputs **and** `pre` **and** `post(r)` **and** golden rejects `r` | ❌ **too weak**: an input plus a wrong output the spec accepts | ✅ fine |
| Too weak (whole program) | run the attacker (Section 2.3), only if the single-output check found the spec too weak | — | — |

A postcondition that holds for every output (the second kind of empty spec from Section 1) does not need its own check, because it accepts every wrong output and is caught as too weak.

**Example: the single-output check on clamp.** Take the task "Return x limited to the range [lo, hi]. You may assume lo <= hi", and suppose the LLM writes the following spec:

| | |
|---|---|
| Allowed inputs (from the task) | `lo <= hi` |
| LLM's precondition `pre` | `lo <= hi` |
| LLM's postcondition `post` | `And(lo <= r, r <= hi)` |
| Golden output (from the golden program) | `If(x < lo, lo, If(x > hi, hi, x))` |

We ask Z3 whether some allowed input has an output `r` that the LLM's spec accepts but that differs from the golden output:

```python
x, lo, hi, r = Ints('x lo hi r')
s = Solver()
s.add(lo <= hi)                                  # allowed inputs
s.add(lo <= hi)                                  # pre
s.add(And(lo <= r, r <= hi))                     # post accepts r
s.add(r != If(x < lo, lo, If(x > hi, hi, x)))    # r is not the golden output
s.check()   # sat: x = 5, lo = 0, hi = 10, r = 0
```

Z3 answers **sat** with x = 5, lo = 0, hi = 10 and r = 0: the spec accepts 0 even though the correct answer is 5. The spec is too weak because it never says the result should equal x when x is already inside the range.

### 2.3 The attacker

The attacker looks for a program `p` that does two things at once:

```text
for every allowed input x:  if pre(x) holds, then post(x, p(x)) holds    -- p passes the LLM's spec
for some allowed input x:   p(x) is not an intended output                -- but p is wrong
```

Here "not an intended output" means that `p(x)` differs from the golden program's output or, for tasks with many correct answers, breaks the golden spec. Only the second condition needs the golden program; the first compares `p` against the LLM's spec alone. In practice, where there is no golden program, the second condition is judged against partial ground truth such as existing tests, or by a person reviewing the program (Section 4.2).

We give an LLM the task description and the spec and ask it for the **simplest plausible implementation that passes the spec but is wrong**, written in the same small expression language as the golden program so that Z3 can reason about it. The attacker runs as a guess-and-check loop, in the style of counterexample-guided synthesis (CEGIS), with the LLM doing the guessing:

1. **Hint.** We pass along the example from the single-output check, e.g., "the spec accepts r = 0 when x = 5, lo = 0, hi = 10, but the intended output is 5". This tells the LLM where the spec is weak. (This step uses the golden program.)
2. **Guess.** The LLM proposes a program, for example `return lo`.
3. **Check.** Z3 checks whether the program passes the LLM's spec on every allowed input, and whether it is wrong on at least one allowed input. (The second question uses the golden program.)
4. **Retry.** If either check fails, we tell the LLM why, e.g., "your program fails the spec when x = 11" or "your program is actually correct", and let it try again, up to 3 attempts per spec.

Since Z3 confirms every program, we never report a false alarm, even when the LLM gets things wrong. One edge case: if a spec is too strong at some input in a way that rejects *every* possible output there, no program can pass it at all, so the attacker cannot succeed. We record these cases separately.

#### 2.3.1 Why an LLM attacker rather than a search-based synthesizer

Our original plan was a classical search-based synthesizer. We decided to build the LLM attacker first, for four reasons:

- **LLMs are now strong program writers.** We expect a modern LLM to find wrong-but-passing programs at least as often as a classical search, and to produce bugs that look like real developer mistakes rather than odd programs only a machine would write.
- **Search is expensive.** The number of possible programs grows exponentially with size: a small integer grammar over two variables has about 2,000 programs up to size 6, but about 17 million up to size 11. A wrong program for a spec as simple as clamp can already need size 11, and we have around a hundred specs to check.
- **Search can only find what its grammar can express.** Any wrong program that needs an operation outside the grammar, or is slightly over the size limit, is out of reach. Each new kind of task, such as arrays, state or timing, would also need its own grammar.
- **Scope.** With two people, the LLM attacker lets us put our time into the spec checks, the repair loop and the evaluation.

This choice has costs. When the LLM attacker comes back empty-handed, that does not prove the spec is strong. However, *detecting* a too-weak spec never depends on the attacker, since the single-output check already does that; the attacker's job is to turn the detection into a readable wrong program. The LLM also will not always find the shortest such program, so we ask for the simplest one and record program size. Finally, LLM results vary between runs, so we fix the model version and settings, repeat runs and cache responses (Section 3.1).

A useful side effect is that we can measure the attacker's success rate automatically. The single-output check tells us exactly which specs are too weak, so we simply count how many of them the attacker turns into a confirmed wrong program.

#### 2.3.2 Future work: a search-based attacker

If time allows, we will also build a search-based attacker and compare it with the LLM one. The plan is deliberately simple: list programs from smallest to largest in a small grammar (variables, constants, `+`, `-`, comparisons and if-then-else), check each one with Z3 using the same two conditions as above, and stop at the first program that passes the LLM's spec but is wrong. Because smaller programs come first, the result is the shortest wrong program in the grammar. If nothing turns up within the size limit, we report "no wrong program found up to size N".

### 2.4 Fixing the spec

In our experiments, repair starts from a first spec that is too weak, because that is the case where all three feedback types below make sense. Specs that are already correct are left alone.

```text
spec = a saved first spec that is too weak
repeat up to 3 times:
    problems = run all checks on spec (and the attacker, for feedback type 3)
    if there are no problems: stop
    spec = LLM rewrites the spec, given the description, the old spec and the problems
```

We compare three kinds of feedback:
1. **Pass/fail only.** The LLM is told the spec is wrong but not why. This is our baseline.
2. **A single example** from whichever check failed, e.g., "for x = 5, lo = 0, hi = 10 your spec accepts 0, but the intended output is 5".
3. **A whole wrong program.** Same as (2), except that a too-weak spec is reported with the attacker's program, e.g., "your spec accepts `return lo`".

A revised spec can develop a different problem along the way, for instance becoming too strong. In that case we report it with the example from the check that failed.

## 3. Benchmarks and Evaluation

### 3.1 Benchmarks, model and experiments

**Benchmarks.** We will start with 30 tasks, 10 in each of three groups, and add more if time allows:
1. small integer functions such as min, max, abs, clamp, median and sign;
2. integer and array problems adapted from MBPP/HumanEval-style datasets, using fixed-length arrays (for example, position of the maximum, counting occurrences, checking whether an array is sorted, and the insertion position in a sorted list);
3. tasks with many correct answers, to make sure our checks do not flag a spec that is correct but loose. For example, "return any number larger than both a and b" has 8, 9 and 100 all correct for a = 3, b = 7, and "return the index of a largest element" has both 1 and 3 correct for [5, 9, 2, 9].

For each task we write the allowed inputs and either a golden program or a golden spec, as described in Section 2.1. The golden specs also double as a test of our checks: every golden spec must pass all of them.

**Choice of LLM.** We plan to use **Claude Sonnet 5.5** for all LLM roles: writing specs, attacking them and repairing them. It is strong at writing code and following formatting instructions, which keeps format failures low and lets us focus on spec quality. It is also a mid-tier model, much cheaper and faster than flagship models, which matters across a few hundred calls. Comparing several LLMs is left for future work.

One concern is that Sonnet 5.5 may be *too* good. If it writes correct specs for almost every task, we will have too few weak specs to attack and repair, and RQ2 and RQ3 would rest on very little data. We will check this early on a small pilot set. If weak specs turn out to be rare, we will switch the spec writer (writing and repairing specs) to a lower-tier model, which makes more mistakes, and keep Sonnet 5.5 as the attacker.

**Experiments.** We fix the model version and settings and cache every LLM response. We then run two experiments:
- **Experiment 1: checking first specs (RQ1, RQ2, RQ4).** The LLM writes a first spec for every task three times, so we can see how much the results vary. Every first spec goes through the checks, and every first spec found too weak is attacked (up to 3 attempts).
- **Experiment 2: repair (RQ3).** We take the too-weak first specs from one of the three repeats and run the repair loop once with each feedback type (Section 2.4), for up to 3 rounds each, at low temperature.

With 30 tasks, this comes to at most about 400 LLM calls.

### 3.2 How we decide whether a spec is correct

We count an LLM-written spec as **correct** if Z3 proves it is logically equivalent to the golden spec, meaning that for every allowed input the two accept exactly the same outputs. This is a separate Z3 query, which gives us an objective ground truth.

The four Z3 checks line up with this definition. A spec that is not equivalent to the golden spec must fail at least one of them: either its precondition rules out an allowed input (too strict), it rejects an intended output (too strong), or it accepts a wrong output (too weak). So whenever a spec is incorrect, the checks tell us how, and the repair loop in Section 2.4 stops before its round limit only when the spec is correct.

### 3.3 What data we collect

In Experiment 1, a **sample** is one task in one repeat. In Experiment 2, a **repair run** is one too-weak first spec with one feedback type. We log the following for every sample and repair run, so that all results can be recomputed from the logs:

| Level | Data points |
|---|---|
| Each spec (first and revised) | whether it passed the format check; the result and example from each of the four Z3 checks; whether it is equivalent to the golden spec; its labels (correct, too weak, too strong, too strict on inputs, empty, or did not follow the format; a spec can have more than one) |
| Each attack | number of attempts; whether a confirmed wrong program was found; the program and its size; why failed attempts were rejected (did not pass the spec, or was actually correct) |
| Each repair run | feedback type; number of rounds used; whether it ended with a correct spec, and after how many rounds |
| Cost | Z3 time per check and per attack; number of LLM calls, tokens and dollars per sample and per repair run; number of Z3 "unknown" answers or timeouts |

### 3.4 Research questions and how we answer them

**RQ1: How common are bad specs?** We want to know how often LLM-written specs are empty, too strict on inputs, too strong or too weak, and what they tend to leave out, such as "the result is one of the inputs", index bounds or "the answer is unique". We answer this with the labels of every first spec from Experiment 1, broken down by task group and averaged over the three repeats, with the spread shown. We also group the too-weak specs by what is missing, based on the examples the checks return. The results will be a table of defect rates and a short list of the most common missing conditions, with examples.

**RQ2: How good is the attacker?** Among the first specs that the single-output check found too weak in Experiment 1, we measure how often the attacker produces a confirmed wrong program, how many attempts it needs, and how simple its programs are, broken down by task group. If we build the search-based attacker (Section 2.3.2), we also compare the two on success rate, program size and time for the integer tasks. The results will be a table of success rates, attempts and program sizes.

**RQ3: What feedback helps the LLM fix its spec?** We want to know which feedback type produces the most correct specs in the fewest rounds, and in particular whether a whole wrong program helps more than a single example. Using Experiment 2, all three feedback types start from the same too-weak first spec with the same round limit, so each spec serves as its own control. We compare the share of repair runs that end with a correct spec and the average number of rounds. The key comparison is (3) against (2), while (1) tells us whether detailed feedback helps at all. We also count, spec by spec, how often one feedback type fixed a spec that another did not. The results will be a table of success rates and rounds per feedback type, and a chart of the fraction of specs fixed after each round.

**RQ4: Is it practical?** Using the cost logs from both experiments, we measure the time and money spent per spec on the Z3 checks, the attacker and the LLM calls, broken down by task group. We also check whether Z3 copes with the array tasks, whose specs need `ForAll`/`Exists`, by comparing the rate of "unknown" answers and timeouts on array tasks and integer tasks. The results will be a cost table per task group.

## 4. Discussion

### 4.1 Why synthesize a whole wrong program?

- Z3 on its own can only give us one bad point: a single input and a wrong output that the spec accepts. It cannot output a whole wrong program, because it finds values for variables, not programs.
- A whole program is more useful than one bad point because it shows the kind of mistake the spec allows, not just one place where it fails, which points directly at the missing condition.
- A wrong program that passes the spec is exactly what could slip through verification, so it shows the real risk.
- A program also stays readable on array or stateful tasks, where a single wrong output can be hard to interpret.
- A whole program can potentially give the LLM more valuable information for rewriting the spec than one bad point. This is the question our repair experiment answers: Section 2.4 compares the two kinds of feedback, and RQ3 measures which one leads to more correct specs.

### 4.2 What if there is no golden spec?

- In real work there is usually no golden program or golden spec, since the spec is often written before any trusted code exists. Ours exist only to measure spec quality in our experiments.
- However, industry projects usually already have **partial ground truth**: test cases and probes that give known-correct and known-wrong outputs for some inputs, plus properties the engineer cares about.
- This partial ground truth can replace the golden spec, and most of the flow still works. The empty-precondition and too-strict checks only need the allowed inputs and are unchanged. The too-strong and too-weak checks compare the spec against the known-correct and known-wrong outputs instead of the golden spec. The attacker still uses Z3 to confirm that its program passes the spec on every input, and shows it is wrong where it disagrees with a known output or breaks a property.
- We keep the benefit of program synthesis: the attacker still produces a readable wrong program that passes the spec. An engineer can review it quickly ("your spec also allows `return lo`; is that OK?"), or it can be fed back to an LLM to repair the spec.
- The guarantee is weaker, though. Every problem found is still real, but only problems at the inputs the tests cover (or that break a property) can be found. Passing every check no longer proves the spec is correct, and repair may make the spec fit the tests while staying weak elsewhere. This is why our experiments use golden specs: we need them to measure true correctness, defect rates and repair success.

### 4.3 Are the tasks realistic enough?

- Keeping the tasks small is deliberate: it lets us write trusted golden specs and measure errors exactly.
- The tasks still cover common sources of weak specs: missing cases, a missing "result is one of the inputs", index bounds, quantifiers and tasks with many correct answers.
- If time allows, we will audit specs from a public Dafny-based benchmark and add tasks with state and timing. The LLM attacker makes these easy to add, since it needs no grammar.
