# Browser roadmap after 1.4

This is the durable handoff for continuing the retro-web Browser work after
PR [#253](https://github.com/jggonz/os8088/pull/253). Keep the PR's final
description and review history as its record; this file carries the roadmap,
review outcomes and engineering lessons forward after the PR is merged.

## Baseline

- Browser 1.2b through 1.4 is the foundation: bounded HTTP redirects,
  sequential ASCII framesets, frame-relative links, basic GET/POST forms,
  text formatting, table spans/alignment, page titles and the existing
  fixed-memory text renderer.
- The PR branch includes the current `main` merge for EGA themes (#249). New
  work starts from PR head `238b00f1`, which contains upstream `main` at
  `3987ad05`; re-check `main` before any eventual PR or merge.
- HTTPS and JavaScript remain unsupported. The optional host proxy is not a
  dependency of the native Browser.
- The page remains bounded by the existing 20 KB document ceiling. Small
  pages must not pay for a larger fixed buffer; EMS/XMS remains optional work.

## Milestones

| Version | Scope | Status / direction |
|---|---|---|
| 1.2b | Quality of life: relative links, redirects, framesets, forms and navigation fixes | Integrated into PR #253 |
| 1.3 | HTML text formatting: emphasis, headings, lists, quotes, preformatted text and paragraph spacing | Integrated into PR #253; leading-space preservation is fixed and regression-tested on this branch |
| 1.4 | Table layout, alignment, additional form controls and simple POST | Integrated into PR #253; maintainer fixes and validation are recorded below |
| 1.5 | Navigation and usability: history, bookmarks, Find, View Source, keyboard navigation and better download handling | Next milestone; include safe text/binary downloads, audit existing behavior first, and keep each feature bounded |
| 1.6 | Memory-aware browsing: review the incremental 20 KB ceiling and investigate optional EMS/XMS-backed page data | After 1.5; additional memory must stay optional |
| 1.7 | Image handling, initially static GIF/JPEG, reusing PiXEL decoders where practical | After memory design; decode on demand with small reusable buffers |
| 1.8 | Compatibility and integration against early-Web pages; refine progressive rendering and interactions | Follow image and memory work |
| 1.9 | Full audit: code reuse, allocations, binary/RAM footprint and 8088 execution cost | Broad optimization after features stabilize |
| 2.0 | Consolidated stable Browser for early-Web use | Release milestone |

Modern CSS, JavaScript engines, Java applets and modern web frameworks remain
out of scope. The target is useful support for early-web pages on constrained
real machines, preserving the Browser's text-first behavior.

## PR #253 review record

The maintainer's review comment at
[issuecomment-6200134012](https://github.com/jggonz/os8088/pull/253#issuecomment-6200134012)
records the following validation: fast tier 44/44, `make small`, unchanged
kernel size baselines, floppy-image fsck, VGA and CGA QEMU checks, Browser
soak rows, `tests/brnav.py`, and one `make test-full` run. The `brrace` row
exceeded its wall limit both on this branch and at the merge base; it is a
test-budget issue on `main`, not a Browser regression. Vintage-hardware
validation remains a separate check.

The PR's roadmap comment is the source for the milestone scope above. The
maintainer replied that binary/text downloads and small, optionally loaded GIFs
were also desired, and suggested considering the 360 KB package budget when
deciding what ships where. Keep 1.5 focused on navigation/downloads; retain
memory work before inline images, and consider a network-disk build if an
optional feature materially grows the app. The maintainer said they expected to
merge PR #253 after review; as checked on 2026-10-15 it remains open and awaits
the requested code-owner approval.

The maintainer added five Browser fix commits after the original 1.4 commit:

- `2271e601`: count table columns correctly; preserve alignment and link
  payloads; make every document walker consume inline cell metadata; fix the
  1bpp styled-text ink; strengthen table assertions and register `br14`.
- `012cedb2`: submit the focused form on Return; include only the clicked
  submit button; retain form records across frames; refuse unsupported select
  controls visibly; decode entities in `img alt`; repair the Save As test.
- `ea2b93e4`: release the graphics lock on every frameset exit; decide
  redirects before handing off or shrinking page buffers; follow empty-body
  redirects; convert POST 301/302/303 to GET while retaining POST for 307/308;
  resynchronize the location control after redirects.
- `644eed64`: re-claim full-size page buffers before fetching frameset
  children; restore the frameset URL/title/host after the final child; test
  large children and a fetch after frameset completion.
- `238b00f1`: preserve each link's frame owner; underline links that begin
  mid-line; add a fixture and assertion combining `colspan` with `align`.

These are useful patterns for future Browser changes: audit every walker when
adding document markers; treat register ownership and clobbers as contracts;
balance graphics locks and page claims on every exit; test state transitions
and controls in tables/frames; and make assertions discriminate against the
failure mode rather than merely find a nearby string.

## Open review notes

These were explicitly non-blocking in the maintainer's review. Decide each in
its own change rather than quietly broadening 1.5:

- Back/Reload after a POST uses GET because history stores URLs, not request
  bodies. This branch now states that in the status line; preserving/replaying
  a POST body would require a separate design and memory budget.
- The apparent ~235-byte BSS opportunity was audited. Offsets +543..+545 and
  +548..+779 provide 235 free bytes, but the live tail fields (`br_wtitle`,
  table scratch, and the frameset URL) occupy 237 bytes. The holes are separated
  by fixed live slots, and `br_url` begins at +780; compacting this tail would
  require lifetime-sensitive aliasing or moving the tested fixed URL/worker
  layout for at most two bytes. It is therefore not a safe or useful BSS
  reduction as part of this Browser follow-up. The assembler assertions remain
  in place. Revisit only alongside a measured memory redesign with a complete
  overlap-lifetime audit.
- Stale Browser version comments and the unused submit-label string were
  cleaned up on this branch. Continue removing dead labels only after proving
  both storage and lifetime redundant.
- Keep the two-cell scrollbar margin: the maintainer considers it a visual
  choice. `docs/INDEX.md`'s duplicate README entry is the generator's basename
  behavior and already passes `--check`; don't hand-edit it. The `brrace`
  timeout is an upstream test-budget item.

## Follow-up audit fixes on `browser-1.5-roadmap`

- `<pre>` now preserves literal leading spaces; `br14`, `htmsim --selfcheck`,
  and SPEC.md cover the behavior.
- Back/Reload after a POST response still issues GET, as the URL-only history
  design requires, but the Browser now reports that the POST body was not
  replayed. `brnav.py` covers the request and status message.
- Removed obsolete BSS aliases and their parser writes, the unused submit
  label, stale 1.2b wording, and corrected the encoder comment. No fixed worker
  offsets or package image/BSS layout were moved.
- Latest `upstream/main` was checked at `3987ad05` (EGA themes #249); it is
  already an ancestor of the 1.4 Browser head used for this work.

Host-side validation for these fixes: `os88index.py --check`, `checkdocs.py`,
`htmsim.py --selfcheck`, Python byte-compilation of the affected harnesses,
NASM assembly with the Browser recipe's include paths, and `git diff --check`
all pass. The full `make browsertest`/guest run remains to be repeated when a
Linux/QEMU test environment is available; this Windows session's WSL distro
cannot start because virtualization support was unavailable.
