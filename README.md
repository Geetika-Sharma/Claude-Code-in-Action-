# Claude-Code-in-Action
Notes from https://anthropic.skilljar.com/claude-code-in-action

## Steering Long Sessions
### Scope the work first with plan mode
Before Claude writes a single line, get it to lay out a plan. In plan mode, Claude does its research in read-only mode. It reads the code, figures out what needs to change, and hands you a plan to review.

When you get that plan, actually read it. Don't skim it. The more thorough the plan, the fewer surprises you'll hit once Claude starts executing. If something's off or missing, just ask Claude to add it where you want. Iterating on a plan is much faster than letting Claude run and hoping for the best, then cleaning up the mess.
Steer while Claude works

Once Claude is running, you have a few ways to keep it pointed in the right direction. The first is compaction.

#### Compact
Compact summarizes your conversation, uses that summary as the new context, and deletes the old messages. This frees up your context window so Claude can keep going. The risk is that something important gets dropped in the summary, and Claude drifts off course.

So don't just run `/compact` on its own. Add instructions after the command to tell Claude how to summarize. For example, if you finished debugging a while back and now you only care about some API changes, say so:

```
/compact Focus on the --version flag implementation
```
Anything you write after the command shapes what the summary keeps. That's your steering wheel for context.

#### Rewind
When Claude heads down the wrong path, you don't have to prompt your way back out. Rewind takes you to your last checkpoint. Every user prompt creates a checkpoint you can revert to. To open the menu, double tap escape on an empty prompt.

From the rewind menu you get a few options:
- **Restore code and conversation** - roll back both together.
- **Restore conversation** - roll back just the chat.
- **Restore code** - roll back just the files.
- **Summarize from here** - summarizes everything after the checkpoint. Great if you had a side conversation and just want to free up some space.
- **Summarize up to here** - summarizes everything before the checkpoint. Great when you had a long setup phase you want to compress, but you want to keep the implementation parts intact.

Let Claude run more autonomously
Everything so far assumes you're hands-on, watching and correcting. If you want something more autonomous, there's goal and loop.

#### Goal
Goal sets a completion condition. You describe what "done" looks like, and Claude keeps working across turns until a fast evaluator confirms those conditions are met. It won't just stop the first time it thinks it's finished.

For example:
```
/goal all tests in src/billing pass, and the type checker reports zero errors
```

To cancel it, run `/goal clear`. One important constraint: the evaluator only reads the transcript. So your condition has to be checkable from the output Claude actually produces, like the results of a test run.

#### Loop
Loop runs a prompt on an interval between turns, either fixed or self-paced. Use it to pull something external, like a CI run or a deploy, and act when the state changes.
To stop a loop, just press escape.

#### Run parallel work with worktrees

The steering metaphor so far assumes one steering wheel in one car. But when you're running multiple agents on the same codebase, you don't want two steering wheels in one car. That's unsafe. Two Claude sessions fighting over the same files leads to conflicts.

That's where `worktrees` come in. Instead of sessions stepping on each other, each one gets its own independent file tree.

Because each agent has its own tree, they can't clobber each other's changes. When a session exits, a clean worktree is automatically removed.

There's one helpful file to know about. A `.worktreeinclude` file at the repo root lists git-ignored files to copy into each worktree. This is useful for things like an environment variable file or a local config that you need in every worktree but don't want to commit to version control.

### Putting it together

Handling long Claude Code sessions comes down to a handful of habits:

    Scope your work first, then steer.
    Direct your compaction so the summary keeps what matters.
    Use the rewind menu to course correct when Claude drifts.
    Set a goal when you can describe "done" better than you can describe the steps.
    Run parallel work in worktrees.

Do that, and you can trust a long run without babysitting every step of it.

## CLAUDE.md 

Here's a trap that catches almost everyone: your CLAUDE.md file keeps growing. You hit a problem, you add a rule. You hit another, you add another rule. Before long you've got one giant file, and Claude starts ignoring parts of it. That's not a bug in Claude. It's how the file works.

The key thing to understand is that CLAUDE.md is not enforced configuration. It's guidance. Every line competes with every other line for Claude's attention. The longer the file gets, the more it competes with itself, and the less reliably Claude follows any single rule. So the goal isn't to write down everything. The goal is to keep the file tight. The leaner the file, the more of it Claude actually follows.

### First, ask if CLAUDE.md is even the right tool

Before you write a rule, ask whether it belongs in CLAUDE.md at all. Some rules are guidance, and some rules are hard lines that must never be crossed. Those are two different jobs.

Take a rule like "never push to main." If you put that in CLAUDE.md, you're hoping Claude reads it and respects it. Most of the time it will. But "most of the time" isn't good enough for something that dangerous. A hard rule like that belongs in a pre-tool-use hook instead.

The difference matters. A hook is code that runs before Claude takes an action, and it can actually block the action. So even if Claude does try to push to main, the hook stops it. That's real enforcement, not a polite request. Move your hard rules to hooks and let CLAUDE.md handle the softer conventions.

### The four locations
<img width="700" height="400" alt="image" src="https://github.com/user-attachments/assets/ec3c2253-85d3-47e0-9757-340e9a220630" />

CLAUDE.md isn't just one file sitting in your project. There are four places it can live, and Claude loads all of them together at launch. Nothing gets dropped, and they stack.

Here's what each one is for:

    Managed policy - the org-level file your platform team controls. You can't exclude it, so org policy is always in play.
    User - your personal preferences that follow you across every project on your machine.
    Project - the file shared with your team, checked into the repo.
    Local - ignored by git. Your personal notes for this one repository only.

That last one, local, is easy to overlook but really handy. Say you're refactoring off in your own branch and you want Claude to hold some architectural decisions in mind while you work. That doesn't belong in the shared project file where it'd affect your whole team. It goes in local, where it's just yours for this repo.

### Split up a big file with imports

When your project file starts getting long, you can break it into pieces using the path-to-file import syntax. Instead of one wall of text, you point to other files:

    @.claude/conventions/code-style.md
    @.claude/conventions/testing.md
    @.claude/conventions/workflow.md

This is great for organizing. But know exactly what it buys you, because it's easy to get the wrong idea. When Claude launches, it expands those imported files inline, right where you referenced them. So imports help you keep things tidy, but everything still loads up front. They do not reduce the amount of context Claude has to read. Use imports to organize, not to shrink the load.

### Phrasing is what makes rules stick
Once you've decided a rule belongs in CLAUDE.md, whether Claude actually obeys it comes down to how you phrase it. Most rules fail because they're vague. Here's how to fix that.

#### Be specific and check-able

Don't write "follow best practices." Do you even know exactly what that means? If you can't check whether it was followed, neither can Claude. Compare these two:

    **Vague**: "Follow best practices for API routes."
    **Specific**: "Put new API routes in src/api/handlers, one per file."

The second one is explicit. You can look at the result and immediately tell if it was done right. That's the bar every rule should clear.

#### Name the replacement, don't just ban something
When you tell Claude not to do something, say what to do instead. Otherwise you've left the door open.

    **Leaves it open:** "Don't use default exports." Okay, but then what?
    **Closes it:** "Use named exports, not default exports."

The second version names the replacement, so there's nothing left to misinterpret.

#### Emphasis is a budget
Words like "IMPORTANT" and "YOU MUST" do raise a rule's priority. But only relative to everything quieter around it. If every rule shouts, then nothing stands out and the emphasis means nothing. So treat emphasis like a budget. Spend it on the two or three rules that really hurt when they get broken, and let the rest sit at normal volume.

#### Keep the file under revision
Your CLAUDE.md file is never finished. Treat it like living code that keeps getting edited.

When Claude does the wrong thing, don't just sigh and fix it by hand. Treat it as a bug report against your CLAUDE.md file. You can even tell Claude directly: "add that to the CLAUDE.md file," and it'll write the rule for you. That way the file gets better every time something goes wrong.

### The bottom line
Treat your CLAUDE.md like production code. If you can't justify a line, delete it. To keep the file lean and followable:

    Move hard rules to hooks, where they're actually enforced.
    Organize long files with imports (just remember they don't reduce context).
    Make every rule specific and checkable, and name the replacement.
    Spend your emphasis budget on the few rules that matter most.
    Keep revising the file whenever Claude gets something wrong.

The whole idea is simple. The leaner the file, the more of it Claude follows.

## Verification Skills
As your project grows, you start noticing the same work happening over and over. You already know skills are a good way to automate repeated work. In this lesson we look at one specific job that skills are great for: verifying your own work. If there's one skill worth building first, this is it.

### Why a verification skill is the one to build first
Think about how you normally check Claude's work. You ask it to refactor something, it finishes, and then you have to remember to double-check it. Maybe you ask it to run the tests. Maybe you read the diff yourself. The problem is that the checking depends on you remembering to ask for it. Skip that step once and bad code slips through.

A verification skill removes that dependency. Here's the shape of it. You ask Claude to refactor something. When it finishes, the change matches the skill's description, so the skill fires on its own. From there it:

    Runs the test suite.
    Reads the diff.
    Checks that no test was weakened just to make things pass.
    Reports pass or fail, with the evidence attached.

The whole flow runs without you asking. The description on the skill is what triggers it, and once triggered it walks the same steps every time.

Notice the last check in that chain. It's not enough to run the tests and see green. A test can be quietly loosened so it passes no matter what. So the skill reads the diff and confirms tests weren't weakened. "Done" isn't "the code looks right" from reading the diff alone. Done is the gates being run and observed, with the results stated explicitly.

This same shape carries any procedure your team repeats. A release checklist. A migration recipe. A pre-PR check. The rule of thumb: if you've typed the same multi-step instruction twice, that's a skill.

### A skill folder can hold more than instructions
A skill isn't just a single `skill.md` file. The folder around it can carry other things, and this is what makes skills powerful for verification.

    Drop a `reference.md` next to the skill for detailed material, then link to it from `skill.md`. Claude only reads it when it actually needs that depth. Your main file stays short.
    Put scripts in the folder too. Claude executes them rather than loading their contents into context. That means a skill can carry its own tooling, like a `check.sh` that runs all the gates.

The takeaway: keep `skill.md` itself lean. Push the heavy material, the long explanations and the executable scripts, into side files. The lean file describes what to do; the side files hold the depth and the tools.

### Which instruction surface owns which rule
<img width="700" height="400" alt="image" src="https://github.com/user-attachments/assets/b7b9835d-2abc-4d7b-b26f-e0a2ecfe2903" />

By now you've got three places to put instructions, and it's easy to mix them up. Here's a quick way to keep them straight.

Conventions that apply all the time, things like naming rules or where files go, belong in your `CLAUDE.md` file. Procedures and reference material tied to a particular kind of task belong in a skill.

There's a third case. A rule that Claude must not be able to skip belongs in a hook, not in either of the above. That's because `CLAUDE.md` and skills are both instructions that Claude follows, while a hook is code that actually runs. If skipping the rule isn't acceptable, don't leave it up to instruction-following.

### The recap
A `skill` is a folder with a `skill.md` inside it: a name, a description that triggers it, and the procedure itself. Only the descriptions load into context until a skill is actually needed, so there's no cost to packaging every procedure you repeat.

Start with verification. Build the skill, check it into your project's `.claude/skills`, and now the whole team inherits the same move. Everyone's work gets checked the same way, automatically, without anyone having to remember to ask.

## Permission Modes
Permission modes let you decide once what Claude is allowed to run without stopping to ask you. Instead of approving every action one prompt at a time, you pick a mode that matches the job and let Claude work at the level of trust you're comfortable with.

You've already met a few of these modes. Every time you hit shift-tab, you cycle through them: manual, accept edits, and plan. Those cover the everyday, hands-on work. The rest of the modes are where hands-off Claude Code really lives, and the one to reach for there is auto.

### The six permission modes
Here's the full set. Each mode draws a different line between what runs freely and what needs your sign-off.

- **Manual** reads only, without prompting. Everything else asks first.
- **Accept** edits runs reads, file edits, and common file system bash commands without asking. This is for iterating on code that you review after the fact.
- **Plan** reads only. It researches and proposes changes without editing anything.
- **Auto** accepts everything, with a separate classifier model reviewing each action before it runs.
- **Don't ask** allows only pre-approved tools. Everything else is auto-denied with no prompt.
- **Bypass permissions** skips all checks. This is the equivalent of the dangerously-skip-permissions flag. Only run it inside an isolated container or virtual machine.

### Cycling with shift-tab
You don't need to memorize a command for each mode. Press shift-tab to cycle through the everyday ones: manual, accept edits, plan, and auto. The status bar at the bottom always shows which mode you're currently in, so you can glance down and know exactly what Claude is allowed to do.
How auto mode works

Auto is the hands-off mode. Claude runs on its own, but before each action executes, a separate classifier model reviews it. The classifier guards intent. It's watching for moves that escalate beyond what you actually asked for.

Here's the kind of thing it's designed to block:
- Production deploys and migrations
- Force pushing, or piping downloaded code straight into a shell
- Sending sensitive data to external endpoints
- Destroying files that exist for the session

And it waves through the everyday work: local edits in your project, installing dependencies from your lock file, read-only requests, and pushing to your own branch.

### What the classifier can't do
The classifier checks intent, not correctness. It won't catch whether the code actually works. So if you ask Claude to refactor authentication and it writes broken authentication, the classifier waves it through, because broken isn't dangerous.

That's why you pair auto mode with a stop hook that runs your tests. The two work together:
- Auto mode watches what Claude is trying to do while it runs.
- The stop hook confirms the code actually runs once Claude finishes.

One guards intent before each action, the other guards correctness after. Auto mode's guardrails are still evolving, so check the docs for the current block and allow lists.

### Don't ask, for unattended runs
Don't ask is the right move whenever no human is around to approve prompts: CI pipelines, scheduled jobs, overnight batches. Only pre-approved tools are allowed, and anything off that list gets auto-denied with no prompt. That's the whole point. Your pipeline keeps moving instead of hanging on an approval no one is there to give.

### Match the mode to the job
There are several permission modes, and you reach the everyday ones by cycling shift-tab. To sum it up:

- Auto is the hands-off mode. The classifier checks intent before each action, and a stop hook checks correctness after.
- Don't ask covers unattended pipelines where no one is there to approve.
- Bypass permissions belongs only inside isolated containers and VMs.

Pick the mode that fits what you're doing, and let Claude run at that level.

