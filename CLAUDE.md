# CLAUDE.md

Shared repository guidance lives in [AGENTS.md](AGENTS.md).

@AGENTS.md

## Fork notes (lawlesx/noop)

This is a personal fork of `ryanbr/noop`. Everything below is fork-specific and overrides nothing in
AGENTS.md except where it says so.

- **PRs go to this fork only.** Branch from `main`, push to `origin` (`lawlesx/noop`), open the PR
  against `lawlesx/noop` `main`. Never open PRs or issues on `ryanbr/noop` unless explicitly asked.
  The fork has issues disabled, so reference upstream issues as `ryanbr/noop#N`, never bare `#N`.
- **`gh` must act as `lawlesx`.** Check with `gh api user --jq .login` before any `gh` write.
- **Android first.** Verify with Gradle. Touch Swift only where the parity contract requires it (the
  pure twins under `Packages/`). Apple app targets are not built locally (no `xcodegen`).

### Local toolchain (Apple Silicon macOS)

| Need | Use |
|---|---|
| Gradle (needs JDK 17) | `export JAVA_HOME=/opt/homebrew/opt/openjdk@17/libexec/openjdk.jdk/Contents/Home` |
| `Tools/` scripts (need Python 3.12+) | `/opt/homebrew/bin/python3.12` — the system `python3` is 3.9 and fails parity tooling with `extractall() got an unexpected keyword argument 'filter'` |
| Android SDK | `~/Library/Android/sdk` (has `platforms;android-34`, `build-tools;34.0.0`) |
| Emulator | AVD `Pixel_10`. Demo debug package `com.noop.whoop.demo.debug`; strap log readable with `adb shell run-as com.noop.whoop.demo.debug cat files/strap-log/<file>.log`. The demo seeds daily rows only, no raw HR, so every day is skipped by scoring. |

### Known local quirks

- `RecoveryDriversTest` `driverPointRoundingUsesNearestWithHalfTiesAwayFromZero` and
  `issue51NegativeHalfTieUsesDefaultArg8WithoutChangingScoreOrDriverFields` fail on this machine on
  unmodified `main` (JDK 17 and 21). Pre-existing; not caused by fork changes.
- **Parity ledger.** After adding or renaming a declaration under `Packages/**` or the Kotlin analytics
  roots, run `/opt/homebrew/bin/python3.12 Tools/parity_ledger.py --refresh-derived --base main`
  (10+ minutes) and commit the regenerated `Tools/parity_twin_map.json`. Learned the hard way:
  - A twin claim must name the other side in backticks on ONE line: ``Byte-identical to the Swift twin `name`.``
  - Pairs are keyed by name and arity. Do not add a parameter to a paired function (that reads as a
    pair removal, which is never waivable); add a new paired function instead.
  - A paired function left with only test call sites is flagged `test-only-callsite`.
  - A nested helper `func` inside a Swift function counts as its own declaration.

## Fork changelog

Newest first. One entry per merged fork PR: what changed, whether it moves a shipped score, and what
evaluating it needs.

### 2026-10-27 — PR #1 (`cc58a094`): analytics shadow instrumentation

No shipped score changes. Swift twins included for parity.

1. **`IntelligenceEngine` header** (Kotlin + Swift): the comment quoted "HRV ~60%" and strain 0–21;
   corrected to `RecoveryScorer`'s real weights (HRV 0.55, RHR 0.20, sleep 0.15, resp 0.05, skin temp
   0.05) and the 0–100 Effort scale. Comment only.
2. **Charge with HRV on ln(RMSSD)** — `RecoveryScorerTrace.lnHrvShadowLine`. Charge z-scores raw-ms
   HRV; `ReadinessEngine` already uses lnRMSSD. With Test Centre's Recovery mode on, each scored night
   logs `charge day=… lnHrvShadow … wouldScore=… delta=…`: the real scorer re-run with ln(HRV) and an
   ln baseline (same history and epoch, `readiness_hrv_ln` config) in the HRV slot.
3. **Learned HRmax** (refs ryanbr/noop#2438) — `StrainScorer.learnedHRmax`. Effort uses the override or
   Tanaka (±10 bpm). Every scoring pass logs `effort hrmaxLearned … learned=… tanaka=… peakDays=…`:
   the 2nd-highest sustained daily peak in the window, adopted only above Tanaka (raise only).
   `DayResult.sustainedPeak` carries the value through the day cache. The default pass is 21 days, so
   the 90-day window is only reached on longer passes.

**Next:** run the Full build with a strap for a few weeks with Recovery mode on, then compare both
shadow lines against real nights before proposing either as a default-off Experimental toggle.
