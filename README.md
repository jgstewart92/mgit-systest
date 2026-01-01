# mgit-systest

Synthetic test data for mgit, a minimal git subset written in C. Nothing here is a program: every file was made by a
small deterministic generator (one fixed seed, the same bytes on every run) to exercise the parts of git's object
format, pack files and transfer over ssh and https that mgit implements. Commit and tag dates are synthetic as well
(one minute apart from 2026-01-01 00:00 UTC), so the commit ids come out the same on every build.

## How each round is checked

A round adds commits with mgit and pushes them with `mgit push` (after `mgit push --dry-run`). Then fresh clones are
made with mgit and with git, over ssh and over https, and each is compared with the repository the commits were made
in:

- every file, directory placeholder and symlink: type, mode, size, and sha256 or link target;
- commit and tag ids, the log of every ref (`git log --all`), mgit's first-parent `log` against git's, `mgit tag`
  against `git tag`;
- `git fsck --strict`, and a clean `git status` and `mgit status`.

The previous round's clones then `mgit fetch` and `mgit pull` the new commits and must match the fresh clones.

## What each part tests

| directory or commits | what it tests | how it is checked |
| --- | --- | --- |
| `start/`, commit 1 | a root commit (no parent) on `main` | ids and log |
| `many/`, commit 2 | 3013 files: 3000 small ones in 30 directories, 24 levels of directories, file names of 200 to 255 bytes (ASCII, 2-byte and 3-byte UTF-8), a 255-byte directory name holding a 255-byte file name, a path of about 3800 bytes | the file list and every file's sha256 |
| `names/`, commit 3 | 49 file names: leading, trailing and inner spaces, a tab, one name in both NFC and NFD, Latin, CJK, Hangul, Greek, Cyrillic, Hebrew, emoji and a zero-width space, bytes that are not UTF-8, names starting with `-`, `--` and dots, names that differ only in case (`Case.txt`, `case.txt`, `CASE.txt`; `Dir/` and `dir/`), quotes, a backslash and other shell and URL metacharacters | the exact name bytes in every clone |
| `empty/`, commits 4-6 | empty files, directories kept by an empty `.keep`, a file that becomes a directory and then a file again, a directory that becomes an empty file and then a directory again | types and sizes; no stray or missing directory after each step |
| `links/`, commits 7-9 | symlinks: relative, upward, absolute (to `/etc/hostname`), dangling, to a directory, to another link; a link retargeted; a file and a directory each replaced by a link, then both back; the executable bit set, cleared and set again | link targets and modes (100644, 100755, 120000) |
| `binary/`, commit 10 | incompressible data from 0 B to 20 MiB, on and beside the 4 KiB and 1 MiB boundaries; highly compressible data: zeros (1 MiB and 20 MiB), one 4 KiB block repeated to 20 MiB, one line of text repeated to 4 MiB | every file's sha256; pack sizes |
| `delta/`, commits 11-61 ("delta step N of 51") | one 1 MiB text file with lines edited, inserted and removed at each step, so packs and fetches can carry deltas | the file at every step; objects and bytes sent by each fetch |
| `history/`, commits 62-271 ("history step N of 210") | a long history: a file added at each step, amended three steps later, every tenth one moved to `history/renamed/` five steps later, every fifteenth one deleted seven steps later | the log of every ref |
| tags | lightweight `v9001.1.1` to `v9001.1.4` (history steps 50, 100, 150, 200) and annotated `v9001.2.1` to `v9001.2.5` (steps 25, 75, 125, 175 and 210, the last with a second paragraph) | tag ids, peeled ids and messages; `mgit tag` against `git tag` |
| `merge/`, commits 272-276 (made with git) | three side commits, a two-parent merge and a three-parent (octopus) merge, the same content at two paths; mgit brings them in as a fast-forward | `git log --all` with parents; mgit's first-parent log |
| `notice/` and `.gitmodules`, commit 277 (made with git) | entries mgit stores but does not act on: a submodule entry (a gitlink whose URL, https://example.invalid/sub.git, does not resolve), an LFS pointer whose object was never uploaded, and `.gitattributes` filters | `mgit clone` prints a notice for each; the submodule stays an empty directory and the pointer stays as stored |
| `big50/`, commit 278 | one incompressible file just over 50 MiB, where GitHub starts to warn | the pre-push check warns and the push goes on |
| `README.md`, commit 279 | this file | |

Rounds after the first push add commits on top of these (more delta steps, a rename wave, new tags).

## Cases kept out of this repository

The refusals are tried only in throwaway clones and never committed here: a file over 100 MiB, CR LF line endings, a
string shaped like an access key, a name the pre-push rules forbid, a branch other than `main`, a tag not shaped
vX.Y.Z, and a non-fast-forward. Each must be refused before anything is sent: the remote's refs (`git ls-remote`) are
the same before and after.
