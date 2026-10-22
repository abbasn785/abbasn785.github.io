# AI is grading AI now. Here's how to keep it honest.

*What I learned building an evaluation framework for AI agents.*

Back when we wrote ordinary code, testing was the easy part. You called a function, you compared its output against a known value, and the assertion was either true or false. `assertEqual(add(2, 2), 4)`. Done.

Agents broke that. An agent's output is non-deterministic and open-ended: a paragraph of natural language, a recommendation, a root-cause analysis. Run it twice on the same input and you get two differently worded answers that may both be correct, or both wrong, or one of each. You can't `assertEqual` your way out of that. There's no verbatim comparison to make.

So we do the next best thing: we use an LLM's natural-language understanding to read the agent's output, read the *correct* output, and decide whether they carry the same meaning and value. This technique is called **LLM-as-a-judge**, and it's exactly what it sounds like: AI grading AI.

That should make you a little nervous. You're now stacking one non-deterministic system on top of another and asking the top one to be a fair, consistent judge. It can be, but only if you build guardrails around it. After building one of these systems and watching it fail in instructive ways, I've landed on three.

## First, what "evaluating an agent" even means

You can evaluate an agent on a lot of axes: did it pick the right tools, was its reasoning trajectory sound, did it stay on task. This post is about the one that ultimately matters most to a user: **answer correctness.** Was the final answer right?

Grading answer correctness needs exactly three ingredients:

1. A set of inputs that your agent responds to.
2. The validated correct output for each input to serve as the ground truth.
3. Your agent's output for each input.

The obvious way to get #3 is to do what unit tests do: freeze a set of inputs, run your agent against them at test time, grade the results. That works fine for a self-contained agent. But if your agent depends on live telemetry (metrics, logs, the current state of a system), a frozen input is a lie. The world the agent investigates has moved on since you captured the test case.

The better approach is to dark-launch the agent: run it silently against real, live triggers in real time, alongside whatever human or existing process currently does that job. Then you collect both the agent's output and the real validated output, and grade them against each other. The agent is being tested on exactly the conditions it'll face in production, because they are production.

That collection step, quietly gathering both sides, is where the guardrails live.

## Guardrail 1: Keep as much of the pipeline deterministic as possible

Here's the mistake I made first, and it's an easy one to make when LLMs feel magic.

I built the whole eval system as an agentic workflow: a single AI agent, armed with MCP tools, told to go collect all the outputs from where they lived, normalize them, and grade them. No real code. The instructions were detailed and it felt elegant.

It turned out to be a mess. The collector would disregard explicit instructions about how to gather outputs. Most damagingly, it would *summarize* the outputs it collected instead of preserving them verbatim, quietly dropping detail that the grade depended on. And over a long session it made the ordinary mistakes long-context LLMs make, such as instruction-following decaying and earlier rules falling out of focus. I ended up babysitting every run. I was hand-checking each score, confirming it had actually followed the rubric, and kicking it back to regrade again and again. I couldn't trust a score, because I couldn't trust that the thing being scored had even been captured faithfully.

The fix was to stop asking the LLM to do things a script does better. I codified the entire collection and normalization pipeline in plain, deterministic code: fetch each output from its source, preserve it byte-for-byte, structure it into a known schema. The LLM's job shrank to the *one* step that genuinely requires language understanding: the grade itself.

The principle generalizes: **an LLM should touch the smallest possible surface of your eval.** Every step you can express as code, express as code. Reserve the model for the irreducibly fuzzy judgment, and hand it clean, complete, structured inputs when you do. Determinism everywhere you can afford it is what makes the one non-deterministic step trustworthy.

## Guardrail 2: Limit context rot

An LLM's context window is a double-edged sword. It's what lets a model remember a mistake and not repeat it. But it's also how bias creeps in: the model weighs something it *shouldn't* be seeing, and it quietly tilts a later judgment. The gradual decline in output quality driven by *what's in the window* (not by hitting the length limit) has a name: context rot ([see article by Jake Minns](https://towardsdatascience.com/governed-context-managing-context-rot-in-claude-code/)).

It's tempting to treat everything you hand the model as passive reference material: there if it's relevant, ignored if it's not. It isn't. Context is active input: a model tends to incorporate what's in its window even when that content is irrelevant to the task. So whatever you put in front of the judge shapes its answer, whether you meant it to or not. The design question isn't just "what does the judge need to see?" It's "what could it see that would bias the exact judgment I'm asking for?"

An example in my system came from grading several agents at once. I was evaluating four agents on the same input, and the efficient-looking move was to hand the judge everything together (the ground truth plus all four agents' answers) and ask for a score per agent. It scored, but the scores were subtly wrong, and it took me a while to see why.

With four RCAs sharing one context window, the judge blurred them. While grading agent A, it would consider a detail that agent B had actually raised, mistake a detail from agent C's output for ground truth, or let one strong answer in the batch affect the score of a weak one sitting next to it. Each agent was being graded partly against the *other agents* instead of purely against the ground truth. Put four answers in the window and the model treats all four as fair game. Again: context is active input, not a set of labeled boxes it keeps separate on your behalf.

The fix is isolation. One grading call per agent, and that call sees exactly two things: that agent's output and the ground truth. Nothing about the other agents, not even that they exist. A score for agent A is now a function of A and the truth alone. It comes out the same whether A is graded alongside three brilliant answers or three terrible ones, which is the whole point of a fair comparison.

More broadly: at every step, ask what the model is allowed to see, and cut anything that could bias the specific judgment you're asking for. Blind the judge to information it doesn't strictly need. Context rot is subtle precisely because the model never tells you it happened: it won't flag that a stray input swayed it, it just quietly folds that input into the answer, and the score drifts.

## Guardrail 3: Make sure the "correct" output is actually correct and complete

Here's the assumption that will burn you: that your ground truth is perfect.

If you're building an agent, its job is almost certainly being done by a human today, and the "correct" outputs you're grading against are whatever those humans produced. Human outputs are **incomplete.** People fill in the fields they care about, skip the ones they don't, leave placeholders, and move on. Grade a perfectly good agent answer against a half-empty human record and you'll score the agent as wrong when it was the record that was lacking.

So put a **completeness gate** in front of your ground truth. Before an input counts, require the metadata and fields that mark a *finished* correct answer: a status flag, specific required fields, non-placeholder content. If the ground truth isn't complete, the case doesn't enter the eval. It's better to grade fewer cases well than many cases against fiction.

Some gaps a field check can't catch. A record can have every required field filled and still be too thin to grade against: a vague sentence where a real mechanism should be. For that I run a separate LLM check whose only job is to decide whether the ground truth establishes a real, gradable answer at all. And this is where Guardrail 2 comes back around: that check sees *only* the ground truth, never any agent's output. If I let it see a confident, fluent agent answer, that answer could talk it into treating a stub as gradable, the model reading "this all sounds plausible" and waving through ground truth that says nothing. Blind to the answers, a placeholder reads as a placeholder every time.

## The through line

None of these guardrails are about making the LLM smarter. They're about **boxing it in**: shrinking its job, controlling what it sees, and refusing to grade against truth you haven't verified is true and whole.

AI grading AI is genuinely workable, but the trust doesn't come from the model. It comes from the deterministic scaffolding you build around it: code for everything code can do, blindness where bias would creep in, and a hard gate on the quality of your ground truth. Get those right and the one fuzzy step in the middle, the judgment only a language model can make, becomes something you can actually rely on.
