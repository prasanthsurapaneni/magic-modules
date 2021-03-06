Concourse CI tools for MagicModules and Google Providers
===

These tools manage the downstream repositories of [magic-modules](https://github.com/GoogleCloudPlatform/magic-modules), and are collectively referred to as "The Magician".

Currently, they manage:

* terraform
* ansible

# CI For Downstream Developers
If you're interested in developing the repositories that MagicModules manages, here are the things you'll want to know.

## What The Magician Does
The Magician takes the PR you write against MagicModules and creates the downstream (generated) commits that are a result of your changes.  It pushes the downstream commits to the relevant downstream repositories, and opens PRs against them, referencing your open PR against MagicModules.  When those PRs are merged, it updates the submodules under `build/` and merges your MagicModules PR.

## Your Workflow

You'll write some commits, likely modifying files under `provider` or `products`, and push a PR.  Since The Magician needs to be able to add commits to your branch to update submodules, **make sure the branch of your PR is in `GoogleCloudPlatform/magic-modules`, not your personal fork**.

The Magician will take over, running the generator and running the downstream tests.  If the tests fail, you'll see "check failed" status on your PR<!--TODO(@ndmckinley) - better reporting and logs of test failures.-->.  When all the tests pass, the PR or PRs created by The Magician will link back to your MagicModules PR.  From there, it is your responsibility to engage with *all* the downstream PRs - The Magician doesn't test or merge those unless otherwise indicated.  You cannot merge your MagicModules PR until *all* downstreams accept the changes.  If a downstream requests changes and you need to modify your MagicModules PR, simply push another commit to your MagicModules PR and The Magician will push any generated updates to the downstream PRs.  Note that if you merge one PR, then another downstream asks for changes which affect the already-merged PR, you will have a messy situation - use caution when many repositories are impacted by your change.

## Useful `fly` Invocations

If you want to see what The Magician is going to do to your PR, you'll want to use the `fly` CLI.  You can find documentation about how to use `fly` [here](https://concourse-ci.org/fly.html), but the following invocations may be useful.

First, log in to https://terraform.ci.cloud-graphite.com and download `fly` from there.  You'll need to authenticate `fly` with that target, using `fly login`.  `fly login -t main` will prompt you interactively for the username and password.  [Here are the docs about fly login](https://concourse-ci.org/fly.html#fly-login).

After you're authenticated, the most useful command will be `fly execute`.  [Execute is documented here](https://concourse-ci.org/running-tasks.html).

**If you want to see the generated terraform repository**, run this command from the root of your repository (after ensuring that the downloaded `fly` is executable and in your PATH):

<!--TODO(ndmckinley) Make these commands resilient to missing files and require no setup.-->

```
fly execute -t main --config .ci/magic-modules/branch.yml --input magic-modules=. --output magic-modules-branched=/tmp/magic-modules-branched/ --include-ignored
fly execute -t main --config .ci/magic-modules/generate-terraform.yml --input magic-modules-branched=/tmp/magic-modules-branched  --output terraform-generated=/tmp/terraform-output/  --include-ignored
```

This tells Concourse to upload the current directory (including files that `git` treats as irrelevant, like the `.git` directory).  It treats the repo as if it were the PR that the Magician is running against, creates a branch, and then generates the terraform repository, placing it in /tmp/terraform-output.

**If you want to run the terraform tests on the generated code**, run this command:

```
fly execute -t main --config .ci/unit-tests/test-terraform.yml --input terraform=/tmp/terraform-output  --include-ignored
```

**If you want to run the terraform tests on the existing submodule**, run this command:

```
fly execute -t main --config .ci/unit-tests/task.yml --input magic-modules=. --include-ignored
```

# CI For MagicModules Developers
If you develop MagicModules (code generation features), here are the things you'll want to know.

## Deploying the pipeline
The pipeline config is generated using jinja2.  You'll want `j2cli`: `pip install j2cli`.  To generate the pipeline, use `j2 .ci/ci.yml.tmpl`.  To deploy it, use `~/fly -t prod sp -c <(j2 .ci/ci.yml.tmpl) -p magic-modules`.

## Adding Tests
You can easily add things to the test suites.  The version of the `.yml` and `.sh` files in this subdirectory which are run when you call `fly execute` are the versions at HEAD in your local copy of the repository.  If you write new tests which require some setup, you can just add them to the shell scripts which are already being executed - if the overall shell script exits with a nonzero code, the task will be marked as failing.

# CI For CI Developers
If you're working on enhancing these CI tools, you'll need to know about the structure of the pipeline and about how to develop and test.

## Resources, Jobs, and Tasks
Concourse has three primitives - Resources, Jobs, and Tasks.

* Resources represent state external to Concourse, such as a git repository, a GitHub repo's pull request queue, or a Docker Hub page.  Resources have versions, which, taken together with the resource definition, represent a single immutable piece of state.  If the state changes, the version also changes.  For this reason, versions are usually hashes of the state: A commit hash would be a version, or a PR number / merge SHA pair, or a container SHA.
* Tasks are run to produce state internal to Concourse.  A shell script, running inside a container, with pre-specified input and output directories, is a task.  A task can have many inputs and outputs.  The inputs are usually state specified by a resource version, or an output of a previous task.  The outputs are usually used to update resource versions, or as inputs to a later task.
* Jobs tie together Resources and Tasks.  One task that we have can be summarized as "run the MagicModules compiler to generate Terraform, and output the resulting Terraform directory.  The associated Job is much more complicated - it can be summarized as:
  * Download a version of the MagicModules PR Resource.
  * Run it through one task to prepare it.
  * Run it through the task to generate Terraform.
  * Upload the output to the Terraform Resource as a new version.
  * Run the task to cause the output MagicModules repository's Terraform submodule to point to the new Terraform version.
  * Upload the output MagicModules repository as a new version to the MagicModules Resource.

## Containers

Concourse is strongly based on containers, and consequently it's worth keeping track of which containers the pipeline uses.

### nmckinley/concourse-github-pr-resource
This is based on https://github.com/jtarchie/github-pullrequest-resource.  It contains a few features that haven't been merged into that repo yet, which you can see at this fork: https://github.com/ndmckinley/github-pullrequest-resource.  It is probably not necessary to update this container for now, and we should switch to using jtarchie/concourse-github-pr-resource ASAP, but if there are problems in the meantime, message @ndmckinley.

### nmckinley/concourse-git-resource
This is based on https://github.com/concourse/git-resource/pull/172.  This PR can't be accepted into that repository as-is, because it makes drastic changes, but the proposed replacements aren't ready yet.  This container will also go away when those changes are made.  It shouldn't be necessary to update, but if, for instance, you need a feature which was added to the git resource after the container was built, message @ndmckinley.

-------

#### A side note on tracking submodules
The merge job sets the submodules back to tracking their downstream repositories on `master`, by updating the submodule commits to point to the most recent commit on `master` for their respective downstream repositories.  This may not be ideal - other commits may have been made to `master` since the downstream PR was merged - however, this is the best way to ensure that we do not go backwards.  Imagine the following situation:

`magic-modules` PR #1 is created.  `downstream-repo-a` PR #7 and `downstream-repo-b` PR #8 are created from that PR.  `magic-modules` PR #2 is created.  `downstream-repo-a` PR #9 is created from that PR.  PR #7 is merged, then PR #9.  Since all of PR #2's downstream PRs are merged, PR #2 is merged, and it includes the changes from PR #1.  PR #8 is finally merged, and so PR #1 is ready to be merged.

When merging PR #1, if we update `downstream-repo-a` to the merge commit for PR #7, we will go backwards, erasing PR #2.  If we update to `master`, we will definitely include both the changes of both PR #1 and PR #2.

-------

# Troubleshooting The Magician

Sometimes, the Magician won't do what you expect!  When that happens, try some of these things.

## My PR didn't get a Magician run!
There are several possible reasons.  Does your PR have a merge conflict?  Is it written by someone who is a collaborator / contributor / member of the `magic-modules` repo?

### Check the `magic-modules-new-prs` resource.
Find your PR number and commit hash, and click on that version to get a detail view.
#### It didn't work!
If you don't see your PR number and commit hash, verify the following things:
- The PR is mergeable.
  - If not, fix the merge conflicts.
- The author of the PR has the `member`, `contributor`, or `collaborator` badge on the PR page.
  - If not, **manually verify that the PR is appropriate to run through the magician**, and then run the `authorize-single-rev` task on the PR number and hash.

### Find the number of the `mm-generate` command that was run.
It should be prominent in the detail view.
#### It didn't work!
Press the 'power' button on the PRs above it and click the '+' button in `mm-generate` - it's very rare, but sometimes Concourse falls behind and drops some things it ought to do.  It's also plausible that Concourse was briefly paused for maintenance and more PRs stacked up on top of yours, which prevented Concourse from ever reaching it.  In any event, it should continue now.  If you *ever* have to do this, tell a Magician maintainer - this is one of those things that we could build alarms for.

### Open `mm-generate` and find your run.
It should be green and list a bunch of tasks that were performed to generate your downstream repos.

#### It didn't work!
If the run isn't listed there, probably it's the case that it's still pending (grey) or there's been a global Magician reboot - find a Magician maintainer to solve the mystery.  If the run is listed there but it's red, the error message should help you out.  If you're having trouble figuring out what's gone wrong (e.g. it's a puppet / chef build and the error message is swallowed by Concourse), try `fly -t prod hijack magic-modules/mm-generate`.  You may have to configure Fly first - see up the page for help on that.

### Find the `magic-modules` output hash.
It should be at the bottom of the page, after `point-to-submodules`.

#### It didn't work!
Something has gone strangely and you should find a Magician maintainer.  Make them update this page.  :)

### Open the `magic-modules` resource from the main page.
Find your version and click on it to get a detail view.  Confirm that the unit tests have run and are green.

#### It didn't work!
If the unit tests didn't run, see above for what happens if `mm-generate` didn't run.  If they didn't *pass* ... :)

## There was a confusing git error message
Git's error messages can be obtuse!  If you see something about how the `git push` failed, because a rev cannot be pushed if it already exists, you probably have accidentally triggered two Magician runs at the same time - two force pushes which try to push to the same rev at the same time will conflict with each other.  If you see something about how the `git clone` failed because an object is unadvertised, you probably have done a `push` / `amend` / `push -f` sequence.  Both these errors can be ignored if the pipeline is progressing.  If the pipeline has stopped progressing, try to run the failing step again with the '+' button.

## The Magician failed while running on my submitted PR
If Concourse notices the Magician's pushes to your PR before it notices that the Magician has merged your PR, it might try to generate again - this is always safe, don't worry.  :)

## I submitted some generated PRs, but not all of them, and I need to make a change.
This is the most common issue and the solution isn't complicated.  If you delete all the `depends: ...` lines from the Magician's comments, the Magician will think that it is starting over from scratch.  It'll push to the `codegen-pr-##` branch, instead of the `codegen-sha-######` branches, and it'll try to open PRs from those branches.  If you want a clean experience, you can close your existing PRs (just hit 'Close' at the bottom of the PR, don't merge them) and The Magician will do the right thing.

If you want to leave those PRs open for some reason (preserve comments, because an external partner is watching them, etc), no problem, the magician will still do the right thing, it'll just update the PRs in the `mm-generate` step (*before* the tests) rather than in `create-prs` (*after* the tests) - your PR will still get updated even if the generated code fails tests.

## Other: Consult your local Magician Expert
Remind them to update this page.
