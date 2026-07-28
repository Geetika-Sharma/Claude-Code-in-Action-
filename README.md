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
<img width="1491" height="685" alt="image" src="https://github.com/user-attachments/assets/ec3c2253-85d3-47e0-9757-340e9a220630" />
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

