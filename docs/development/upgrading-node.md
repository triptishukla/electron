# Upgrading Node

## Discussion

Chromium and Node.js both depend on V8, and Electron contains
only a single copy of V8, so it's important to ensure that version
of V8 is compatible with both Node.js and the version of Chromium
in a build.

Upgrading Node is much easier than upgrading Chromium, so fewer
conflicts arise if one upgrades Chromium first, then chooses the
upstream Node release whose version of V8 is closest to the one
Chromium contains.

Electron has its own [Node fork](https://github.com/electron/node)
with modifications for the V8 build details mentioned above
and for exposing API needed by Electron. Once an upstream Node
release is chosen, it's placed in a branch in Electron's Node fork
and any Electron Node patches are applied there.

Another factor is that the Node project patches its version of V8.
As mentioned above, Electron builds everything with a single copy
of V8, so Node's V8 patches must be ported to that copy.

Once all of Electron's dependencies are building and using the same
copy of V8, the next step is to fix any Electron code issues caused
by the Node upgrade.

[FIXME] something about a Node debugger in Atom that we (e.g. deepak)
use and need to confirm doesn't break with the Node upgrade?

So in short, the primary steps are:

1. Update Electron's Node fork to the desired version
2. Backport Node's V8 patches to our copy of V8
3. Update the GN build files, porting changes from node's GYP files
3. Update Electron's DEPS to use new version of Node

## Updating Electron's Node [fork](https://github.com/electron/node)

1. Ensure that `master` on `electron/node` has updated release tags from `nodejs/node`
2. Create a branch in https://github.com/electron/node: `electron-node-vX.X.X` where the base that you're branching from is the tag for the desired update
  - `vX.X.X` Must use a version of node compatible with our current version of chromium
3. Re-apply our commits from the previous version of node we were using (`vY.Y.Y`) to `v.X.X.X`
  - Check release tag and select the range of commits we need to re-apply
  - Cherry-pick commit range:
    1. Checkout both `vY.Y.Y` & `v.X.X.X`
    2. `git cherry-pick FIRST_COMMIT_HASH..LAST_COMMIT_HASH`
  - Resolve merge conflicts in each file encountered, then:
    1. `git add <conflict-file>`
    2. `git cherry-pick --continue`
    3. Repeat until finished


## Updating [V8](https://github.com/electron/node/src/V8) Patches

We need to generate a patch file from each patch applied to V8.

3. Remove our copies of the old Node v8 patches
  - Read `patches/common/v8/README.md` to see which patchfiles
    were created during the last update
  - Remove those files from `patches/common/v8/`:
    - `git rm` the patchfiles
    - edit `patches/common/v8/README.md`
    - commit these removals
4. Inspect Node [repo](https://github.com/electron/node) to see what patches upstream Node
  used with their v8 after bumping its version
  - `git log --oneline "deps/v8"`
  - The commit message "update V8 to <version>" represents replacing `deps/v8` with the named version of V8, exactly as it is in upstream. The commit message "patch V8 to <version>" isn't a complete replacement of `deps/v8`, but just applying the changes in that version.
5. Create a checklist of the patches. This is useful for tracking your work and for
  having a quick reference of commit hashes to use in the `git diff-tree` step below.
6. Apply all patches with the [`get-patch` script](https://github.com/electron/electron/blob/master/script/README.md#get-patch):
  - `./script/get-patch --repo src/v8 --output-dir patches/v8 --commit abc123 def456 ...`
7. Update `patches/common/v8/README.md` with references to all new patches that have been added so that the next person will know which need to be removed.
8. Update Electron's [DEPS](https://github.com/electron/electron/blob/master/DEPS#L5) with the commit hash of the latest commit in the `electron-node-vX.Y.Z` branch.

## Notes

- Node maintains its own fork of V8
  - They backport a small amount of things as needed
  - Documentation in node about how [they work with V8](https://nodejs.org/api/v8.html)
- We update code such that we only use one copy of V8 across all of electron
  - E.g electron, chromium, and node
- We don’t track upstream closely due to logistics:
   - Upstream uses multiple repos and so merging into a single repo
   would result in lost history. So we only update when we’re planning
   a node version bump in electron.
- Chromium is large and time-consuming to update, so we typically
  choose the node version based on which of its releases has a version
  of V8 that’s closest to the version in Chromium that we’re using.
  - We sometimes have to wait for the next periodic Node release
   because it will sync more closely with the version of V8 in the new Chromium
 - Electron keeps all its patches in the repo because it’s simpler than
   maintaining different repos for patches for each upstream project.
   - Crashpad, node, chromium, skia etc. patches are all kept in the same place
 - Building node:
   - We maintain our own GN build files for node.js to make it easier to ensure
     that node, chromium and electron are all built with the same compiler flags.
     This means that every time we upgrade node we have to do a modest amount of
     work to synchronize the GN files with the upstream GYP files.
