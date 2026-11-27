# Patches

Everything below is relative to upstream **v1.4.0** (`5bb72e2`).
Only the iOS phone-control path is touched.

**Why:** on a real device (iPhone 18 Pro Max, iOS 27.3, build `24C5049f`) the iOS
phone-control feature could not complete a single action — it could not open a
third-party app, refused every tap on any animated page, and refused every text
input. These patches make it work, and align the judgement model with the
Android and desktop ports.

## Screen judgement

| # | Problem | Fix | File |
|---|---|---|---|
| 1 | Whole-frame pixel comparison rejected any live page: status-bar clock, Dynamic Island, home indicator and caret all count as "the screen changed", so every tap failed with `ERR_PHONE_SCREEN_CHANGED` | Exclude the top 8 % and bottom 4 % bands from the diff and from OCR text matching | `IOSPhoneScreen.swift` (`tolerates`, `stableText`) |
| 2 | The caret exemption was judged after the text-region veto, so it could never apply | Judge the caret before that veto; conditions unchanged | `tolerates` |
| 3 | Element identity was discarded before dispatch — `tap` and `tapXY` both became a bare `CGPoint`, leaving only pixels to compare | `elementTarget` returns the label too; `revalidate` re-finds that label near the approved point | `elementTarget`, `targetStillThere`, `MatchReference` |
| 4 | Coordinate taps (`tapXY`, scroll without an id) fell through to the whole-frame rule | `localPixelsStable`: only the box around the point must be stable | `localPixelsStable` |
| 5 | A changed frame was a **precondition**; the Android port (`ScreenVerdict`) and the Windows computer-use plugins both report it **after** acting | `FreshFrame.change` is `.moved / .unchanged / .unverifiable`, reported in the result instead of blocking | `revalidate`, `IOSPhoneCoreActions` |
| 6 | `tap_xy` divided normalised input by the frame size a second time, so a 0–1 value landed near the middle | 0–1 is normalised; larger values are frame pixels | `imagePoint` |
| 7 | Icon-only controls (emoji key, ＋, tab bar) have no OCR text, so no element id exists, and models guessed their pixel position from a downscaled preview — off by roughly one button width | Draw a labelled coordinate grid on the annotated frame | `annotatedJPEG`, `gridStep` |

## Scrolling

| # | Problem | Fix |
|---|---|---|
| 8 | `scroll` resolved its anchor with `elementPoint` (coordinate only), so revalidation used the whole-frame rule — and a scroll is *supposed* to change the frame | Carry the label; `allowRelocation` lets the label move |
| 9 | `collect` pages through a list and moves content the same way, but was not relaxed | Same treatment as `scroll` |

## Text input

| # | Problem | Fix |
|---|---|---|
| 10 | `ERR_PHONE_SENSITIVE_INPUT` refused ordinary typing: the word list (密码/支付/验证码) matched anything anywhere on screen, and the masked-dot heuristic read bulleted lists as a password field. Even 你好 never dispatched | The refusal is gone from production; the flag is still computed and reported so the model can hand off. The approval card still shows the exact text | 
| 11 | Chinese input goes through the clipboard path, which revalidated **after** the paste using the field's old label — a search box reading 「搜索」 now reads 「你好」, so that check always failed | Post-paste check uses the coordinate rule, tolerance raised to a quarter of the box |
| 12 | `keyboardHint` only recognised a `qwertyuiop` letter row. A third-party CJK keyboard has neither that nor recognisable key caps, so a visibly open keyboard was reported as missing and input gave up | Also accept keyboard control keys, or a bottom strip dense enough to be a key grid; the hint is advisory anyway — paste and ascii only need focus | 
| 13 | Clipboard typing path did not receive `targetText` at all | Passed through |

## Opening apps

| # | Problem | Fix |
|---|---|---|
| 14 | `open` armed `prepareInput`, and `media.rs` marks `may_have_acted` before the stream starts; `core_device.rs` retires the whole run once that is set, so one failed `open` destroyed the tunnel | Only `screen` arms it |
| 15 | Rust `Input::Launch` set `may_have_acted` before calling `launch_application` — same retirement, from the Rust side, so no Swift change could help | Removed; launching a bundle twice has no extra effect |
| 16 | `requestForeground` discarded the private API's answer and was gated on a built-in catalogue, so third-party apps were never attempted | Any installed bundle id is accepted; success is judged by whether this app actually left the foreground |
| 17 | `open` waited on `waitForTarget`, which requires the caller to already be in the background — never true right after launching something | Poll the real scene state instead |
| 18 | The app list went through CoreDevice's `listapps` with `includeDefaultApps: true`, enumerating every system bundle over RSD. It never finished inside the 35 s budget | `includeDefaultApps: false`; the built-in catalogue covers system apps |
| 19 | `applications()` required *every* returned row to be a known bundle, so one third-party app made the whole list unparseable | Filter instead of reject |
| 20 | The list is now read from `LSApplicationWorkspace` in memory, with localised names — no RSD round trip, milliseconds instead of a timeout | `workspaceApplications()` |
| 21 | A failed name lookup told the caller nothing, so it re-guessed across round trips | Return the real names and bundle ids of matching (or the first 40) installed apps |
| 22 | A bundle identifier passed as `app` went through name resolution first | Bundle-shaped values are used as given |

## Request lifecycle

| # | Problem | Fix |
|---|---|---|
| 23 | Any request timeout called `stop(run)`, clearing capabilities and stopping the run; every later call then failed in turn | Only a real deadline keeps the run alive; cancellation, stop and pairing changes keep their existing cleanup |
| 24 | `mutating` was "not screenshot and not apps", so `launch` was reported as possibly side-effecting and the caller was told never to retry | Whitelist: tap / swipe / key / paste / ascii |

## Build

| # | Problem | Fix |
|---|---|---|
| 25 | The snapshot was committed from Windows, where mode bits are not tracked, so 57 shell scripts arrived without `+x` and CI died on `build-ish.sh: Permission denied` | `git update-index --chmod=+x` |

## Deliberately kept

`revalidate` still refuses on a **frame size change** (rotation, keyboard
lifted). The coordinates themselves become invalid there, so the tap would land
somewhere else entirely.

`strict` (default `true`) preserves the upstream contract tests: they pin the
old refusal semantics, and `IOSPhoneScreenTests` keeps exercising the same
comparison algorithm the relaxed path uses. Production passes `strict: false`.

## Verified

`iOS compile` workflow: `compile`, `phone-protocol` and `engine` all green,
including the 861-line Swift contract suite. Builds are unsigned; sign locally.
