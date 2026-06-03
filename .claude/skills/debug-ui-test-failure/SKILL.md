
---
name: debug-ui-test-failure
description: Record, inspect, and diagnose failing iOS Simulator UI tests from evidence. Use when Xcode/XCTest UI tests fail, especially intermittent or visual failures where simulator state, taps, WebView content, accessibility labels, fixture data, animations, or CI-only behavior must be understood before changing code.
---

# Debug UI Test Failure

## Overview

Use this skill to turn a UI test failure into inspectable evidence before fixing it. The core loop is: identify the exact failing surface, enable visible touches, record the simulator while reproducing the failure, inspect the recording with `ffmpeg`, correlate it with `.xcresult` attachments and logs, then make the smallest evidence-backed fix.

## Principles

- Prefer the exact failing CI run, test plan configuration, destination, logs, screenshots, and `.xcresult` over source-only theories.
- Keep reproduction narrow while diagnosing: one failing class or a few failing tests, the same test-plan configuration, and the same simulator runtime where possible.
- Store videos, contact sheets, exported attachments, and derived data under `/tmp/<case-name>` unless the user asks otherwise. Do not commit recordings or generated contact sheets.
- Do not fix from a guessed cause. First prove what the user saw: the visible state, tap target, accessibility snapshot, fixture response, timeout point, or navigation state.
- After the fix, rerun the same failing surface and record again when visual behavior was part of the diagnosis.

## Workflow

### 1. Identify the failing surface

Gather the minimum facts needed to reproduce the same failure:

```bash
git status --short
gh pr checks <PR> --watch=false
gh run view <RUN_ID> --log-failed
xcrun xcresulttool get test-results summary --path <result>.xcresult
```

Record the failing test names, test plan configuration, destination/runtime, failure messages, and whether the failure came from fixture-backed or live/E2E mode.

### 2. Prepare the simulator and artifact paths

Use a booted simulator that matches the failure. Prefer a destination `id=` over a broad name/OS string once a matching simulator exists.

```bash
xcrun simctl list devices booted
mkdir -p /tmp/<case-name>/recordings /tmp/<case-name>/results /tmp/<case-name>/attachments
```

If no simulator is booted, boot the intended one from the project's normal workflow.

### 3. Enable visible touches

Enable touch indicators before recording so taps can be correlated with UI response:

```bash
defaults write com.apple.iphonesimulator ShowSingleTouches -bool YES
```

If touches do not appear, quit and reopen Simulator, then rerun the recording. Do not spend time diagnosing tap behavior from a recording where taps are invisible unless the user explicitly asks to skip this.

### 4. Record the failure while running the test

Start screen recording before launching the test. Keep the recording process ID so it can be stopped cleanly.

```bash
UDID=<simulator-udid>
VIDEO=/tmp/<case-name>/recordings/failure.mp4
PIDFILE=/tmp/<case-name>/recordings/record.pid

xcrun simctl io "$UDID" recordVideo --codec=h264 --force "$VIDEO" &
echo $! > "$PIDFILE"
```

Run the narrow failing XCTest command with a fresh result bundle:

```bash
xcodebuild test \
  -scheme <UITestScheme> \
  -project <Project>.xcodeproj \
  -testPlan <TestPlan> \
  -only-test-configuration "<Configuration>" \
  -only-testing:<Target>/<Class>/<testMethod> \
  -destination "platform=iOS Simulator,id=$UDID" \
  -derivedDataPath "/tmp/<case-name>/DerivedData" \
  -resultBundlePath "/tmp/<case-name>/results/failure.xcresult" \
  -quiet
```

Stop the recording after XCTest exits:

```bash
kill -INT "$(cat "$PIDFILE")"
wait "$(cat "$PIDFILE")" 2>/dev/null || true
```

If the recording process is still running, poll briefly before moving on; incomplete `.mp4` files can be misleading.

### 5. Summarize the result bundle

Read the machine-readable summary before opening source files:

```bash
xcrun xcresulttool get test-results summary \
  --path /tmp/<case-name>/results/failure.xcresult
```

Use exported screenshots, accessibility snapshots, and logs when available. Prefer the project's existing attachment-export command if it has one; otherwise use `xcresulttool` to inspect the result bundle and locate attachments.

### 6. Inspect the video with ffmpeg

Start with metadata:

```bash
ffprobe -hide_banner "$VIDEO"
```

Generate contact sheets at coarse and focused intervals:

```bash
ffmpeg -y -i "$VIDEO" \
  -vf "fps=1/5,scale=360:-1,tile=4x5" \
  -q:v 2 /tmp/<case-name>/recordings/contact-5s.jpg

ffmpeg -y -ss <start-seconds> -t <duration-seconds> -i "$VIDEO" \
  -vf "fps=1/2,scale=360:-1,tile=5x6" \
  -q:v 2 /tmp/<case-name>/recordings/contact-focused.jpg
```

If `drawtext` is unavailable in the local `ffmpeg`, do not block on timestamps. Use row-major ordering and make additional focused sheets around the failure window.

Open the contact sheets or representative frames and answer these questions:

- What was visible immediately before the failure?
- Where did the test tap, and did the app respond?
- Was the expected element absent, offscreen, tiny, covered, disabled, untranslated, or present under a different label?
- Did fixture-backed content load all required subresources, scripts, styles, images, or bridge events?
- Did the test wait for the wrong screen, stale element, wrong locale, or wrong theme state?

### 7. Correlate video, XCTest, and source

Line up the visual evidence with:

- XCTest failure time and message.
- `.xcresult` screenshots and accessibility snapshots.
- App logs and network/fixture logs.
- The test robot or locator code.
- The app surface that owns the behavior.

For WebView or fixture-backed failures, check both the primary API response and the secondary resources referenced by the rendered content. Missing scripts or styles can produce partially visible pages with broken interactions.

### 8. Fix from the observed contract

Choose the smallest idiomatic fix that matches the evidence:

- Fix fixture routes or payloads when strict fixture mode blocked a required resource.
- Fix accessibility identifiers or locators when the UI is correct but the test contract is brittle.
- Fix robot taps or scroll strategy when the target is present but interaction is wrong.
- Fix app behavior only when the recording shows the user-facing behavior is actually broken.
- Avoid sleeps unless the evidence shows a real asynchronous wait contract with no better signal.

Keep the patch scoped. Preserve unrelated local work.

### 9. Validate the same path

Rerun the original failing tests against the patched build and read the result bundle summary:

```bash
xcodebuild test-without-building \
  -scheme <UITestScheme> \
  -project <Project>.xcodeproj \
  -testPlan <TestPlan> \
  -only-test-configuration "<Configuration>" \
  -only-testing:<Target>/<Class>/<testMethod> \
  -destination "platform=iOS Simulator,id=$UDID" \
  -derivedDataPath "/tmp/<case-name>/DerivedData" \
  -resultBundlePath "/tmp/<case-name>/results/after-fix.xcresult" \
  -quiet

xcrun xcresulttool get test-results summary \
  --path /tmp/<case-name>/results/after-fix.xcresult
```

When the first fix exposes broader configuration brittleness, run the narrow test across the applicable test plan configurations rather than assuming the single-locale pass is enough.

## Final Report Checklist

Report only the evidence that matters:

- Failure recording path and any contact sheets inspected.
- The observed failure behavior from the recording.
- The root cause linked to the observed behavior.
- Files changed and why.
- Exact validation command scope and result counts.
- Any remaining unvalidated scope or known CI/local differences.
