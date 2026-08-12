# Example workflow on GitHub

Here's an [example PR](https://github.com/METResearchGroup/mirrorView-task/pull/49) that goes through a flow similar to the following steps:

## Example: Ablations

Let's say I want to experiment between 3 variants of prompts and see which one gives me the most toxic post.

### Commit 1: Plan

I spend the greatest amount of time here. I open the branch, then I open a new folder to store the work for this experiment (maybe something like `experiments/prompt_engineering_2026_08_12/`). Then I open a markdown file, where I define my task in rigorous detail, as if I had to explain to an undergrad how to do it.

1. What the prompts are and how do they differ.
2. What I'm measuring (and what do we need to measure that?). For example, are we measuring toxicity via an LLM call? A third-party API? Looking for keywords? Something else?
3. Where's the data? And is the data usable as it is or does it need transformations? What are the key columns to look at?
4. Other considerations. For example, am I worried that the LLM API will outright refuse to answer my task because of its safety policies? What do I think of that and what would I do then?

I spend probably 80%-90% of time and effort on an experiment here. I also ask multiple LLMs to find holes in my logic and ways it can be clarified. Sometimes LLMs do this to a fault and start making stuff up and giving you noisy or unhelpful critiques, so some discernment is good here. I have a [prompt](https://github.com/mark-torres10/ai_tools/blob/main/skills/grill-me/SKILL.md) that I use that's helpful with this.

Once I come up with a set of requirements and designs, I ask the LLM for its step-by-step proposal of how to do this. This can just be a simple prompt, such as "give me steps for doing this plan". I have my own [prompt](https://github.com/mark-torres10/ai_tools/tree/main/skills/create-implementation-plan) for creating this plan systematically. The important thing is that you understand the plan and know what the LLM is intended to build.

This is also a good spot to ask the LLM what files it intends to write, what functions might look like (not implementation, but would be good to know the function signature), and the columns of any resulting datasets.

Once this is done, I commit the code, `git commit -m "initial plan"`, and push to the remote PR.

### Commit 2: Scaffolding, writing up the first bit of code

Once that plan is locked in, the AI can build in small chunks or pieces. I'd suggest to err on the side of asking the LLM to implement small chunks of work (e.g., 1-2 small functions, 20-50 lines), though what "small" means depends on the context and is up to interpretation. A principle I follow is "if someone woke me up in the middle of the night and put this code in front of me, can I follow every single line of code without getting confused?"

I have a [prompt](https://github.com/mark-torres10/ai_tools/tree/main/skills/interactive-implementation) that does a little bit of work, reports on its progress, and prompts the user to check the work. Adding friction when working with AI goes a long way to making sure it does what it's supposed to and that you're engaged along the way.

Once this is done, I commit the code and push to the remote PR.

### Commit 3: Debugging the code, make sure it compiles

I might find a few bugs along the way as I'm trying to run it. This is also a good spot to have a separate AI agent pore through the code and ask it to (1) find bugs and (2) find places of duplication or unnecessary complexity. More complex and verbose code means code that's more vulnerable to break later.

Once this is done, I commit the code and push to the remote PR.

### Commit 4: Run small versions of the tests

For jobs that either have to run a while or would cost a lot in API costs, I run a small version first. I might do, for example, 10 API calls and see if everything looks right (data is loaded and processed correctly, code runs as intended, the `.csv` output looks good).

Once this is done, I commit the code and push to the remote PR.

### Commits 5, 6, 7: Ablation 1, 2, and 3, save results

For each ablation, I run the code, get the results, and then commit each individually.

### Commit 8: Writeup + results

Then I write up the combined results and look at what the takeaways were. This can be in the form of a README.md file in the folder.

### More commits

Any subsequent commits are then just one-off catching bugs and revising the work. It could also come from me looking at some results, not liking it, and making some change or revision.

### Open the PR

I open the PR in GitHub. An example is something like [this PR](https://github.com/METResearchGroup/mirrorView-task/pull/49).

This gives me a chance to combine all the work that's been done here into a single package or a unit of work. This tells us "here's all the work I did to answer this one specific question", where the end deliverable might be an analysis, a visualization (or a set of visualizations), a trained model, etc.

This is also a good place to be able to summarize your work, so that when future you (or an AI agent) wants to know what you did, you can skim the PR description to know what you did and why.

This is also a good place to keep in mind future reproducibility, as ideally someone can look at your PR and if they wanted to, they could replicate the artifacts you generated based on the code and steps you defined in the PR itself. This is also another benefit of working with a PR, as all the code that you need to replicate a certain result should be a part of the same PR.
