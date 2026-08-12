# GitHub basics

To learn how GitHub works, I recommend [this resource](https://learn.github.com/skills).

But here are a few basics:

## What is Git? And what is GitHub?

### What is Git?

Git is a "version-control system". What that effectively means is that it tracks snapshots of your work so you can go back to the state of your work at a given time.

Version control isn't an unfamiliar concept. Data backups are a type of version control. Your MacBook likely takes some form of "snapshot" of your computer state (for example, an automated backup), so that in case your computer crashes, you don't have to reboot from scratch. Google Docs also has version control, in the form of a clock image (if you hover over it, it even says "Version Control").

[Google Docs version control](google_docs_version_control.png)

Git is one way of doing version control.

Going over Git is out of the scope of this writeup, but I'd really recommend taking an hour to go through [this resource](https://docs.github.com/en/get-started/start-your-journey/git-and-github-learning-resources) and trying to get familiar with how Git works.

### What is GitHub?

GitHub is a hosting platform built on Git. It lets you store Git repositories and collaborate with other people who are also working on their folders.

An analogy is Google Drive: imagine uploading all your work, once you were done, into a Google Drive folder, and then sharing the link to that Google Drive folder to someone else. That person can see your files, they can see the folder name, and they can open files as well.

But now, instead of passing around a Google Drive link, you can send a link to your GitHub repository instead.

But GitHub is much more powerful than Google Drive. It lets you:

1. Manage changes to a repository using PRs and commits, which are extensions of Git's version control.
2. Track the full history of every file, so you can see exactly who changed what, when, and why, and revert to any earlier version if needed.
3. Branch off from the main project to try a new analysis or edit without disturbing the original work, then merge those changes back in once they're validated.
4. Go off someone else's repository to create your own copy under your account, build on their work independently, and propose your changes back to the original project via a pull request.
5. Establish a verifiable record of exactly which code and data produced which results, which is central to making computational research reproducible.
6. Collaborate with co-authors or labmates on the same codebase simultaneously without emailing files back and forth or overwriting each other's work.

GitHub isn't the only website that lets you store and collaborate on Git repositories; alternatives like GitLab and Bitbucket offer similar hosting with their own feature sets, but GitHub remains the most widely used platform.

If Git is like Microsoft Word tracking your edits, GitHub is like Google Docs, a shared place to write, revise, and collaborate with a complete history of who changed what and when.

## What are some key terms to know?

1. Repository
2. PR
3. Commit

## What is a repository?

Think of a repository as a folder. When you run `git init .` in a folder, that folder now officially becomes a Git repository. This means that Git tracks all changes to that folder. Then, you can upload that folder into GitHub and it becomes its own GitHub repository.

For example, [this GitHub repository](https://github.com/METResearchGroup/bluesky-research) is a single folder that lives in my computer. Since all this work was part of the same project, I had git track all the changes to this one folder and then I uploaded that to GitHub.

## What are PRs?

A PR, short for "pull request", is a proposed set of changes to a repository. Usually, you make those changes on a separate branch rather than directly in the main version of the project.

For academic work, it can help to think of a PR as a draft submission for a small piece of research work. The PR should make clear:

1. The question or task being addressed.
2. The approach taken and the important decisions made.
3. The code, data references, and configuration needed to run the work.
4. The outputs or artifacts produced, such as tables, figures, processed data, or model checkpoints.
5. The interpretation of the results and any important limitations.

The PR description provides the high-level narrative of why you did the work you did, while the commits show the work's progression over time. For example, the first commit might add an experiment plan; later commits might add the implementation, run a small validation, execute the full experiment, and write up the results. Together, these create a traceable record of both the final outcome and the decisions that led to it.

This is also a good place to keep in mind future reproducibility, as ideally someone can look at your PR and if they wanted to, they could replicate the artifacts you generated based on the code and steps you defined in the PR itself. This is also another benefit of working with a PR, as all the code that you need to replicate a certain result should be a part of the same PR.

## Git commits

### What is a commit?

A commit is a saved snapshot of changes to a Git repository. When you make a commit, you are telling Git: “Record the current state of these selected files as one meaningful step in the project.”

For example, rather than making one large commit at the end of an experiment, you might create several commits:

1. `add experiment plan`
2. `add prompt variants and evaluation configuration`
3. `add data-processing script`
4. `run small validation sample`
5. `save full experiment results`
6. `write up findings`

This gives the project a readable history. Someone looking at the repository can see not only the final code and outputs, but also the sequence of work that produced them.

### When should you commit code?

Some good analogies for when to commit to a branch and push to the PR include:

1. **Whenever you feel like you'd be comfortable stepping away to take a break and be interrupted.** If what you're doing requires so much context that you'd be inconvenienced if someone popped their head into your office to interrupt you, it's probably not a good time to commit yet. But if you'd be OK if someone interrupted you, that could be a good time to commit.
2. **When you want to log off and continue something later**: when you're so tired that you'd want to step away for a bit.
3. **Whenever you'd normally want to go for a "Save" button**: this depends on how often you want to press "Save". But there are times when you may want to press "Save" or catch yourself thinking "it would be a bummer if what I did here would be wiped out". This might be a good time to "commit".
4. **When you have a unit of work that could be a part of a slide**: this includes a specific statistic, a visual, a graph, etc. and whatever code or work was needed for it.

## How does this all fit together?

See [this example](GITHUB_WORKFLOW_EXAMPLE_1.md)
