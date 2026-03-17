+++
date = 2026-03-17
title = "Deep cleaning of secrets in Gitlab"
[taxonomies]
tags = [ 'git' ]
+++

So, we accidentally `git commit -a -m "I don't know how to use git"`. \
Or we just missed the fact that after a frantic debugging session, the new commit contain credentials for some particularly important service that shouldn't be there.\
We're all humans, after all.\
So now we're going to figure out how to minimize the damage.

# Scenario One: Nothing seems wrong yet

We noticed in time that something went wrong and haven't done `git push` yet.\
Give yourself a gold star for paying attention, edit the file, do `git commit --amend`, and enjoy peaceful life.\
But that's only if the slip-up happened in the very last commit.

If mistake happened several commits ago and we still haven't done `git push`, there are two more or less good options.
* `git reset --soft <commit_sha>`, edit the file, and create a clean commit again.
* `git rebase -i <commit_sha>` and enjoy the thrilling process of an interactive rebase.

Also there is an alternative route using fixups, but you would need to have them in your workflow pipeline from the beginning.

# Scenario Two: Why am I an idiot?

We didn't notice in time that something went wrong and did `git push` anyway.

First of all — **any credentials that made it into the public must be revoked as quickly as possible**.

Secondly — get ready to get beaten. Maybe even with legs.\
To make the beating less intense, do everything from the first case, and then `git push --force-with-lease`.

---

Usually, this is where most of the "What to do if I pushed creds to git?" guides end, and where the actual content that I want to share begins.\
Highly knowledgeable people know that Git has a reflog, and a rewritten commit still stays on the disk, available for recovery. \
And people who make Gitlab know this too, which is why Gitlab periodically runs housekeeping tasks that clean these dangling commits.

To avoid waiting for the janitor to sweep up after you, you can clear the trash in `Settings > General > Advanced > Prune unreachable objects`.\
However, there are cases where this button, despite its dangerous red color, doesn't wipe the dangling commit.\
You can check this by trying to open the commit with a direct link. Something like `gitlab.example.com/group/project/-/commit/46a9d30f7a7d86adf34299367b0be7ffd0700eb60b079f3cad246c57409bd471`

Does it show 404? Everything is fine (relatively). But sometimes it shows the old code, and here they are, our creds in plain text again.

# Scenario Three: Oh God, why?

Here we should take a detour and talk about things that aren't in Git, but are in Gitlab. CI/CD and Merge Requests.\
Somehow we can see the list of commits, diffs, and all of that in the MR.\
And it doesn't go anywhere. Even when the branch is deleted, the MR is closed, and everyone has already forgotten about it.

It's especially "pleasant" to find out when a repository has existed for several years and someone once left an API key in a separate branch, in one of the very first MRs.\
Bonus points if repository was inherited and the account access was lost five years ago, and SOC 2 audit is right around the corner.

And this is exactly the scenario we're going to talk about now. Specifically in the context of self-hosted Gitlab, because no one is giving us root on the cloud.

Also, a small disclaimer: this applies to Gitlab 18.6.2, and I'm not the only one annoyed by the keep-around mechanism, so everything might change in the future.

## Cleaning up the main history

**MAKE A BACKUP!!!**\
Just like this in caps with three exclamation marks. Losing important data while rewriting history is a very real possibility, and you'd better have a recovery point.

Next, we need to run a scanner of our choice and record the full hashes of the commits that contained the creds.\
Just because it's convenient, and running a scanner each time we did something is not a particularly fast process.

And after that, you can run [BFG](https://rtyley.github.io/bfg-repo-cleaner/), [git-filter-repo](https://github.com/newren/git-filter-repo), or some other tool to purge unwanted strings from the history. It doesn't really matter which one.

Then `git push --force --all`, and all of this from the tool's manual.

## Finding entities

Before starting to clean the remote repository, we should track down where these commits are used at all, to not accidentally break the Gitlab UI.

So we do `gitlab-rails console` and start remembering Ruby.

```ruby
commit_sha = "<target_commit_hash>"

# List of pipeline IDs
Ci::Pipeline.where(sha: commit_sha).map(&:id)

# List of MRs
MergeRequestDiffCommit.where(sha: commit_sha).map(&:merge_request_diff).map(&:id).uniq
```

Not very pretty but easy to debug.

I'm not sure if these things must be deleted, but just to be safe. Who knows where Gitlab can break later.\
You can do this through the console, but at this stage, paranoia usually reaches the level "it's better to do this in UI".

## Finding the repository

Before opening the console, you need to go to the UI and find the Project ID. It's in `Settings > General > Naming, descriptions, topics > Project ID`.

Then return to the console and do:

```ruby
Project.find(PROJECT_ID).repository.disk_path
```

This will give us a relative path like `@hashed/5d/0d/5d0db47fcc1b04bcf18b2f6d7d2f8d9c40589f3bf329b55baf7084cb553e8da8.git`.\
It's relative to the storage path, so if you're on NixOS, it's `/var/gitlab/state/repositories/` by default. For other setups — check the blueprint.

Glue it together and let's go:
```bash
cd /var/gitlab/state/repositories/@hashed/5d/0d/5d0db47fcc1b04bcf18b2f6d7d2f8d9c40589f3bf329b55baf7084cb553e8da8.git

# Branches with the commit
git branch --contains <commit_sha>

# Tags with the commit
git tag --contains <commit_sha>

# Refs with the commit
git for-each-ref --contains <commit_sha>
```

The search in branches and tags must return an empty result. If it's not empty, you missed something and it's time to go back to step one.

But the list of refs from the last command are the very things that prevent Gitlab from properly cleaning up commits. It looks something like this:
```
d6cb0ea8746201dfb0a9303563bd52e31555541a25fc9e1dbfda8f757a9c92bb    commit  refs/keep-around/d6cb0ea8746201dfb0a9303563bd52e31555541a25fc9e1dbfda8f757a9c92bb
4ef7ba090eb92e6a95f727b73cf2f1979ba83ee6bb63a4dc902790793b9274c6    commit  refs/keep-around/4ef7ba090eb92e6a95f727b73cf2f1979ba83ee6bb63a4dc902790793b9274c6
fd3fa98ec1d67c155f2e9900023460bd4c41f6b87b2344188a4a1b1b51ee6fbf    commit  refs/keep-arount/fd3fa98ec1d67c155f2e9900023460bd4c41f6b87b2344188a4a1b1b51ee6fbf
d602c43e76a1e51e77b9aa441f5250d14b05ce55b213214aa287610b22b8df5a    commit  refs/merge-requests/512/merge
ae2e6f410385c1e78c2be5bb839299b14db296a9048ddb04a5605e035e023816    commit  refs/merge-requests/712/merge
```

It's logical to assume that `refs/merge-requests` relate to merge requests, but `refs/keep-around` is a slightly different story. As far as I understand (and I didn't understand very well), Gitlab creates these refs for CI tasks, but apparently not only for that. Therefore, cleaning them up might break something besides the CI, which we (in theory) cleaned up in the previous step.

So, at your own risk:
```bash
git update-ref -d <ref_name>
```

And finally:
```bash
git reflog expire --expire=now --all
git gc --aggressive --prune=now
```

# Scenario Four: Hans, get the flammenwerfer!

```
git@sweetiebelle in /var/gitlab/state/repositories/52/66/5266a2866c3d458909112a2a85958e690e5b5b10ac3ef8e97b50f195ff145d74.git λ git for-each-ref --contains b682940 | wc -l
6251
```

**FUCK!!!**

I'll just create a new one.
