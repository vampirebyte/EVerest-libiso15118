# EVerest-libiso15118

The ISO 15118 libraries of [EVerest](https://github.com/EVerest/EVerest), on their own, with their full history:

| Folder | Library | In EVerest |
|---|---|---|
| [`iso15118/`](iso15118) | libiso15118: SDP, the ISO 15118-20, ISO 15118-2 and DIN 70121 sessions | `lib/everest/iso15118` |
| [`cbv2g/`](cbv2g) | libcbv2g: EXI encoders and decoders | `lib/everest/cbv2g` |
| [`slac/`](slac) | libslac: SLAC (ISO 15118-3) | `lib/everest/slac` |
| [`util/`](util) | everest-util: helpers the libraries above use | `lib/everest/util` |

This is an unofficial mirror, not a project of LF Energy or EVerest. Bugs in the libraries belong upstream:
[EVerest/EVerest](https://github.com/EVerest/EVerest). It exists for firmware that uses these four libraries without
the rest of EVerest, for example on a microcontroller: a small repo to pin as a git submodule, with every change from
upstream reviewed in a pull request.

The folders are EVerest's, file for file, plus the few patches in [`patches/`](patches), which are checked on every
pull request.

## Branches and tags

| Ref | Content |
|---|---|
| `upstream` | EVerest main, reduced to the four libraries and `LICENSE`. Generated, never edited by hand. |
| `esp32/main` (default) | `upstream` with the patches applied, this README and the CI. |
| `esp32-YYYY.MM.DD` | Releases of `esp32/main`. Pin these. |
| `libiso15118-v0.9.1`, `libcbv2g-v0.3.2`, `libslac-v0.3.1` | The standalone releases EVerest integrated. |

## Patches

Each deviation from EVerest main is one patch in [`patches/`](patches), with its author, date and sign-off in the
header. They are applied in order onto `upstream`.

| Patch | Why | Until |
|---|---|---|
| `0001` libslac's state machines on `fsm::v2` instead of Boost.MSM, from [EVerest/EVerest#3106](https://github.com/EVerest/EVerest/pull/3106) | No Boost.MSM templates: plain state classes that build on embedded toolchains with less code. | EVerest adopts fsm::v2 |
| `0002` EVerest main's later fixes to the EVSE machine, ported onto `fsm::v2` (EVerest/EVerest#3133, EVerest/EVerest#3166, EVerest/EVerest#3328, EVerest/EVerest#3334) | They were made to the Boost.MSM machine only. | with `0001` |
| `0003` no `SO_REUSEPORT` on lwIP in the SDP server | lwIP (ESP-IDF) does not implement the option; the SDP server would not start. | the ESP-IDF in use supports `SO_REUSEPORT` |

Everything else is identical to EVerest main, including the README, `THIRD_PARTY.md` and `LICENSE` files of the
libraries. `patches.sh verify` checks on every pull request that `esp32/main` is exactly `upstream`
plus the patches, so a library file changed without its patch fails.

### Changing a patch

The library folders are generated: change the patches, not the folders.
[`patches.sh`](.github/sync/patches.sh) turns them into a branch and back:

```bash
.github/sync/patches.sh edit      # worktree on patches/edit = upstream + one commit per patch, authors kept
# in the printed worktree: commit, git commit --amend, git rebase -i, ...
.github/sync/patches.sh export    # writes patches/ from patches/edit
.github/sync/patches.sh apply     # regenerates the library folders
```

A new deviation is a new commit on `patches/edit`. A patch that EVerest merged is deleted from `patches/`; the sync
says so when it finds one.

## Using it

As a submodule, pinned to a release tag:

```bash
git submodule add https://github.com/vampirebyte/EVerest-libiso15118.git third_party/everest
git -C third_party/everest checkout esp32-YYYY.MM.DD
git config -f .gitmodules submodule.third_party/everest.shallow true
```

The include paths are `iso15118/include`, `cbv2g/include`, `slac/include` and `util/include`; the sources are under
`src/` (`cbv2g/lib/` for cbv2g).

## History

The history goes back to the first commit of each library, with every original author, date and message:

- `iso15118/`, `cbv2g/`: the standalone repos [EVerest/libiso15118](https://github.com/EVerest/libiso15118) up to
  v0.9.1 and [EVerest/libcbv2g](https://github.com/EVerest/libcbv2g) up to v0.3.2, then EVerest. The commit that
  integrated each one into EVerest is a merge with the standalone history as its second parent.
- `slac/`: EVerest's SLAC module (from 2020), `lib/staging/slac`, then `lib/everest/slac`; plus
  [EVerest/libslac](https://github.com/EVerest/libslac) up to v0.3.1, joined at the commit that integrated it.
- `util/`: EVerest's `lib/staging/util`, then `lib/everest/util`, plus files moved in from elsewhere.

Rewriting the history to four folders gives new commit hashes, so:

- PR and issue references name their repo, for example `EVerest/EVerest#3385` or `EVerest/libiso15118#45`, so that
  they link there and not to this repo's pull requests.
- References to other commits in messages ("This reverts commit ...") point to the new hashes.
- GPG signatures of the original commits do not carry over. The originals stay verifiable in EVerest.

Some files were rewritten so heavily in one commit that git's default rename detection loses them. Lower the
threshold to follow them back, for example:

```bash
git log --follow -M10% -- slac/src/fsm/evse/context.cpp
```

## Syncing with EVerest

[`sync.yml`](.github/workflows/sync.yml) runs every Monday and on demand:

1. Rebuilds `upstream` from EVerest main and the standalone repos with
   [`build_upstream.sh`](.github/sync/build_upstream.sh) (git filter-repo). The result is deterministic: the same
   inputs give the same hashes, so a sync only adds commits to `upstream`. Of each standalone repo, only the release
   listed in [`upstream.conf`](.github/sync/upstream.conf) is fetched; if it is no longer on GitHub, the history comes
   from this repo's tags (`libiso15118-v0.9.1`, ...) and gives the same hashes.
2. Applies the patches onto it and commits the result on the branch `sync/everest-main` made from `esp32/main`, with
   `upstream` as second parent, so `esp32/main` keeps the whole history. It runs the patch check and opens a pull
   request with the list of upstream commits, or updates the open one. When a patch no longer applies, or is in
   EVerest now, the pull request names it and gives the steps to finish the sync. An open sync pull request that people
   already committed to is left as it is, with a comment.
3. A person reviews and merges it with "Create a merge commit", then tags a release. Squash or rebase merging would
   drop `upstream` as a parent, and with it the EVerest history.

It stops for a person when EVerest moves another folder into the four libraries
([`trace_paths.py`](.github/sync/trace_paths.py) finds new history sources), because updating
[`paths.txt`](.github/sync/paths.txt) rewrites every hash.

Locally: `pip install git-filter-repo==2.47.0`, then `.github/sync/sync.sh /tmp/sources` in a checkout with the
branches `upstream` and `esp32/main`.

Pull requests the workflow opens start no workflows by themselves (a GitHub rule for its token), so it starts
`verify.yml` and `build.yml` for the sync branch itself; their results show as the pull request's checks.

A sync that stops without a pull request (new history sources, not a fast-forward, an error) opens the issue "Sync
needs attention"; the next sync that gets through closes it.

GitHub disables scheduled workflows in a public repository after 60 days without activity. If EVerest leaves the four
libraries alone that long, the Monday sync stops quietly: re-enable it under Actions → "Sync with EVerest main" →
"Enable workflow", or run it by hand there.

[`dependabot.yml`](.github/dependabot.yml) keeps the actions, which are pinned to commits, up to date with a monthly
pull request.

## Build and test

[`build.yml`](.github/workflows/build.yml) runs on every pull request and push to `esp32/main`: it takes EVerest at the
commit `upstream` was built from (`everest` in [`upstream.conf`](.github/sync/upstream.conf), updated by each sync),
replaces its four libraries with this repo's, and builds and tests them in EVerest's CI build kit with
`EVEREST_ENABLE_COMPILE_WARNINGS=ON` ([`build_test.sh`](.github/sync/build_test.sh)). A patch that applies but no longer
fits EVerest fails here. ccache is kept between runs.

## License

Apache-2.0, like EVerest: see [`LICENSE`](LICENSE). Third-party code and its licenses are listed in
[`iso15118/THIRD_PARTY.md`](iso15118/THIRD_PARTY.md) and [`cbv2g/THIRD_PARTY.md`](cbv2g/THIRD_PARTY.md).

The libraries are the work of the EVerest contributors; see the history and
[EVerest](https://github.com/EVerest/EVerest).
