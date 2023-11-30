# Renovate submodules issues repro

This is a minimal setup to reproduce the issue I encountered with Renovate
updating stables branches on repositories having submodules with different
version between branches.

## Explanations

### Situation

1. main branch has a submodule pointing to commit A.
2. stable branch has the same submodule pointing to commit B.
3. renovate needs to update a dependency in the stable branch.
4. renovate checkouts the repository on the main branch using `cloneSubmodules`.
5. renovate checkouts the repository to stable (**and it does not update the submodule!** This is unfortunately the default behavior for git.).
6. renovate bumps the dependency.
7. renovate performs a `postUpgradeTask` for that specific update that has `fileFilters` to `**/**`.
8. renovate save the files as part of post-upgrade since git sees a difference (submodules points to commit A now instead of B.).
9. the PR includes an unwanted (and wrong) submodule change.

### Result

The following is the debug log we get when renovate saves this *wrong* file:
```
DEBUG: Post-upgrade file saved (branch="renovate/stable-golang.org-x-sys-0.x")
{
  "baseBranch": "stable"
  "dep": "golang.org/x/sys"
  "file": "renovate"
  "pattern": "**/**"
}
```

See the third file included in the [stable branch PR dep upgrade](https://github.com/mtardy/renovate-submodules-repro/pull/5/files):
<img width="1243" alt="image" src="https://github.com/mtardy/renovate-submodules-repro/assets/11256051/147547be-2b5d-4adb-a2a8-2b17022b17ab">

### Configuration

The minimal configuration for this is:
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "config:recommended"
  ],
  "baseBranches": [
    "main",
    "stable"
  ],
  "cloneSubmodules": true,
  "packageRules": [
    {
      "matchManagers": ["gomod"],
      "postUpgradeTasks": {
        "commands": ["pwd"],
        "fileFilters": ["**/**"],
        "executionMode": "branch"
      }
    }
  ]
}
```

It seems that this is a bug created by the fact that `cloneSubmodules` and
`packageRules[].postUpgradeTasks.fileFilters = **/**` working together are
creating this situation.

### How to fix this

I guess a simple solution would be to renovate to use the following git config
to true:
```
git config submodule.recurse true
```

This makes sure that when you checkout to another branch you also point your
submodules to the correct commit, basically automating doing `git checkout
another-branch --recurse-submodules`.

### Workaround

I just realized by writing this that I could write a workaround using a
different `fileFilters` excluding the submodule file.

---

## Original message

I'm maintaining a project with Renovate that has a git submodule (and use this
submodule in go.mod replace directive). Thanks to the option cloneSubmodules it
works when I'm working on the main branch.

But when renovate needs to work on a stable branch, let's say v1.0, it `git
checkout v1.0` but does not update the submodule along (with `git submodule
update --init <module>` for example). So I end up on the `v1.0` stable branch
with the content of the submodule folder corresponding to the one on the main
branch. It then creates a PR with the changes and thus bump the submodule
version to the one of the main branch because it never updated it to the
correct commit.

I hope this a clear explanation! Is there a way to fix that behavior?
Apparently, there are ways to immediately update submodules on checkout:

- https://stackoverflow.com/a/55631474/4561420
- https://stackoverflow.com/a/49427199/4561420
- tl;dr -> git config submodule.recurse true.

Is there a way to add git configs to the renovate run to pass git config
submodule.recurse true?
