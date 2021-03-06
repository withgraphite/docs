# Merging your pull requests

## Merging a single PR

Once your PR has been approved & is passing CI (+ any other checks/merge protections you've enabled in GitHub), you'll be able to merge it from Graphite.  Merging a single PR is easy - just hit "Merge" on the right side of the PR title bar.

![Click "Merge" to merge a PR from Graphite - you can also use the dropdown to choose a different merge strategy](../../.gitbook/assets/merge\_100.gif)

In this merge modal, you'll have the option to:
* Select your preferred merge strategy.
  * By default, Graphite will use your default merge strategy from GitHub.  
* Edit a custom commit title and message for your PR.
  * By default, Graphite will use your PR title and message as the commit title and message.
* If applicable, use your GitHub admin merge priviledges to merge past certain blockers.

Once your PR has been merged, you'll see a confirmation snackbar in the bottom left corner of the screen.

{% hint style="info" %}
If you're using the Graphite CLI, you'll want to make sure to run `gt repo sync` immediately after merging in a change to your remote trunk branch - this will zip through and delete your merged branches, ensuring that your local environment is up-to-date and ready for you to keep developing!
{% endhint %}

## Merging a stack of PRs

If you've ever tried to merge a stack of PRs manually through GitHub, you'll know that this can be a process is anything but perfect. Merging your stack can be time-consuming and involve lots of context switching - as you merge, rebase, wait for CI to pass, and merge again all the way up the stack - or force you to ditch the clean PRs you've carefully created, squashing your changes back into a single mega-PR for the sake of merge speed.

After dealing with this pain intenally at Graphite, we've built an automated solution that gives you the best of both worlds.

### What does a stack merge do?

`Merge all (n)` lets you fire-and-forget merging your stack; when you click the button, Graphite will merge each of your PRs one-by-one, automatically:
* rebasing PRs on an as-needed basis (to avoid merge conflicts generated by GitHub's lack of stack support)
* waiting for GitHub checks to pass at each step of the way before merging

### How do I initiate a stack merge?

When you view a PR that is part of a stack (and not the bottom-most PR in that stack), instead of the `Merge` button you'll see `Merge all (n)`, where "n" is the number of PRs below the current PR in the stack plus the current PR. 

{% hint style="info" %}
Note that the merge operation is relative to the PR where the button is displayed. This has two implications: 1) you should always initiate the `Merge all (n)` from the furtheest-upstack PR which you'd like to merge and 2) not all PRs in the stack need to be accepted and passing checks to merge; for example, if only PRs 1, 2, and 3 in a stack of 5 have been accepted and are passing checks, you can still utilize `Merge all (n)` on PR 3 to merge those into trunk.
{% endhint %}

![Merge an entire stack of PRs at once](../../.gitbook/assets/merge\_all\_100.gif)

Merging a stack offers the same customizability as merging a single PR. Through Graphite you can:
* select your preferred merge strategy
* use your GitHub admin merge privileges
* specify custom PR commit titles and messages

### How do I track the progress of my merge job?

To allow you (and your colleagues) to track progress of your merge job, `Merge all (n)` adds a comment to the affected PRs in GitHub.

![](../../.gitbook/assets/merge\_all\_conflict\_github\_2.png)

These statuses are similarly displayed on your PR on the Graphite dashboard.

{% hint style="info" %}
If `Merge all (n)` fails due to a rebase conflict, go to your terminal and run `gt repo sync && gt stack fix && gt stack submit` (resolving any conflicts along the way and running `gt continue`).  Once you've done this, go back to the affected PR in Graphite (or to the same place in the stack where you initially kicked off the `Merge all (n)` job), and click `Merge all (n)` one more time to re-queue the PR for merging.
{% endhint %}

### What happens behind the scenes during the merge job?

Each time the cron job processes a merge job it runs through the following decision tree:
* If the base PR in the stack has pending GitHub checks, do nothing.
* If base PR in the stack is passing all GitHub checks and can be merged, merge. The next PR in the stack is now the new base PR.
* If the base PR in the stack has merge conflicts, rebase the PR and re-submit it (re-entering the waiting-for-CI phase).

During the merge process, Graphite prioritizes:
1) Speed of overall stack merge.
2) Minimizing the number of total CI runs.

To achieve these principles, it's important to note that:
* Graphite eagerly merges rather than rebasing each individual PR before merging.
* Graphite only rebases PRs lazily. When Graphite detects a merge conflict on a PR, Graphite only rebases *that* PR - and not the additional PRs further up the stack. This means that if a stack has `m` merge conflicts, there will only be `m` total rebases (and additional CI runs) kicked off by the merge process.

### How long does a merge job take?

Graphite's cron job to process outstanding merge jobs runs at a cadence of once per minute.

As a result, the length of a merge job depends on how many PRs need to be rebased; if there are no merge conflicts, `Merge all (n)` will take `n` minutes, but if there are merge conflicts, job time is a byproduct of the time it takes to run GitHub checks on a PR and how many PRs encounter merge conflicts.

### What types of merge conflicts can stack merge resolve?

When PRs are merged on GitHub using a squash and merge or a rebase and merge, GitHub [creates a commit/set of commits](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/about-pull-request-merges#squashing-and-merging-a-long-running-branch) for those merged changes.

This means that if you have a stack with PR A at the base, followed by PR B and PR C, when PR A is merged into trunk:
* GitHub creates a commit or set of commits on trunk for the changes in PR A.
* Critically, because this is a new commit, the common ancestor of PR B and trunk *does not change*.
* As a result, GitHub now thinks PR B now includes the commits of both PR B *and* the already-merged commits of PR A.

This behavior becomes problematic when you have a PR A and PR B that modify the same lines and you merge both PRs using one of the aforementioned merge strategies. For any PR C that is stacked atop PR A and B the following transpires:
* The latest version of the change lines on trunk are those in PR B.
* GitHub believes that PR C contains the commits in PR A and PR B.
* When GitHub tests mergability of PR C, it first tries to apply PR A - and now gets a merge conflict - even though we know that in reality we know that we're simply replaying history and that there's no new change.

If you use the Graphite CLI, you'll notice that the CLI handles this scenario for you intelligently. When GitHub reports these sorts of merge conflicts, a `gt repo sync` will pull down the latest changes and rebase PR C for you, cutting out the problematic commmits; a subsequent re-submit will then eliminate the detected merge conflict for PR C.

`Merge all (n)` is designed to similarly automatically resolve this flavor of merge conflict without the need for manual intervention of monitoring. When a merge conflict is found, the merge cron performs a shallow clone of the repo, containing just the stack commits and the trunk commit and utilizes Graphite's knowledge of the stack to perform the same set of operations.

{% hint style="info" %}
Note that this means Graphite can't resolve any legitimate merge conflicts as a result of racing PRs on trunk that require human intervention. If a Graphite rebase fails, Graphite will cancel the merge job. You can restart the job by fixing the issue locally, re-submitting the PR, and enqueueing a new merge job.
{% endhint %}

### Does stack merge work with my merge queue?

Today, merge stack supports label-based merge queues, with future plans to support GitHub's (beta) merge queue.

If you're not sure whether `Merge all (n)` will work with your team's merge process, feel free to reach out to nick@graphite.dev - we'd love to help you unlock this tool.

## Merging without Graphite UI

We find that merging with the Graphite UI saves our engineers - and the engineers that have used it - a substantial amount of time. Anecdotally, we've a number of stories from our users about how they used to block out "half days" just to merge their stacks before using the Graphite UI.

However, if you'd like to manually usher your PRs out the door, you can find a set of instructions on how to merge a stack using our Graphite CLI [here](https://docs.graphite.dev/guides/graphite-cli/landing-a-stack).

{% hint style="warning" %}
Note that we recommend always merging from the bottom of the stack; while there are other techniques, we've found that this is the most intuitive and safe model for our users.

Merging in reverse order from the middle or top of the stack, collapsing all of the PRs into one, is the fastest way to merge an entire stack, but in our experience, there are a number of pitfalls for users - namely around syncing this merged state locally (to continue developing on any upstack PRs) or undoing these changes if a user decides not to merge a PR - that can lead to perilous situations where users have felt like they've lost code or can't re-create their previous state.

While certainly not impossible, it's also harder to re-derive the original stack of PRs when looking at the git history.
{% endhint %}
