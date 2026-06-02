---
name: create-wikipedia-ios-ui-test
description: Build, extend, or refactor UI and E2E XCTest coverage in `/Users/pvzig/Repositories/wikipedia-ios`. Use when adding Wikipedia iOS UI tests, onboarding/article/gallery flows, robots, accessibility identifiers, fixture-backed test data, E2E smoke routing, `UITestConfiguration` launch plumbing, or related validation commands.
---

# Create Wikipedia iOS UI Test

## Overview

Use this skill when adding or reshaping UI-test coverage in `wikipedia-ios`. The useful pattern from prior sessions is: start from the lane and user journey, keep tests intent-level, put mechanics in robots, centralize launch/network setup, use stable app/test accessibility contracts, and validate the narrowest real lane before broadening.

## Workflow

### 1. Orient on the current lane

Start by reading the local repo state, not memory alone:

```bash
git status --short
sed -n '1,220p' WikipediaUITests/README.md
sed -n '1,220p' WikipediaUITests/ROBOTS.md
sed -n '1,220p' WikipediaUITests/Config/UITestConfiguration.swift
sed -n '1,220p' "Test Plans/UITests.xctestplan"
```

Then read the specific test, robot, app surface, fixture manifest, and workflow files touched by the requested flow. Preserve unrelated dirty work.

Default assumptions for this repo:

- Fixture-backed UI tests run through `WikipediaUITests`, `UITests`, and `English (Light)`.
- E2E tests use the same scheme/test plan with the `English (Light, E2E)` configuration.
- `UITestConfiguration` owns launch/runtime configuration only. Do not put feature fixture catalogs, localized selector tables, or test-domain decisions there.
- Keep workflow/test-plan choices in checked-in files. If the repo has `WikipediaUITests/E2ESmokeTests.txt`, use that file for the E2E smoke subset instead of hiding the subset in test code or workflow-only flags.

### 2. Choose fixture-backed or E2E coverage

Prefer fixture-backed UI tests for deterministic app behavior and regression coverage. Use E2E only when the assertion genuinely needs live services, remote images, server behavior, or real integration.

When one behavior should run in both lanes, keep one XCTest method and branch only at the lane boundary:

```swift
try XCTSkipUnless(
    uiTestConfiguration.httpClientProfile == UITestHTTPClientProfile.e2e.rawValue,
    "This flow requires live E2E networking."
)
```

Use `XCTSkipIf` or `XCTSkipUnless` for configurations where the behavior is intentionally irrelevant. Avoid building large conditional fixture flag matrices inside the test body.

### 3. Write the test as a user journey

The test file should read as intent, not automation mechanics:

```swift
func testBohemiaLeadImagePresentsImageGallery() throws {
    try XCTSkipUnless(
        uiTestConfiguration.httpClientProfile == UITestHTTPClientProfile.e2e.rawValue,
        "Lead-image gallery coverage requires live E2E networking."
    )

    launchWikipediaAppRobot(onboardingState: .completed)
        .explore
        .assertVisible()
        .openSearch()
        .openArticle(named: "Bohemia")
        .openLeadImageGallery()
        .assertImagePresented()
}
```

Keep these mechanics out of test methods:

- raw `XCUIApplication` selectors
- scroll loops
- modal dismissal
- localized fallback selection
- RTL/theme branching
- screenshot attachment plumbing
- repeated wait/tap code
- app termination cleanup

### 4. Put mechanics in the right robot

Use one robot per screen or cohesive flow. Return the next robot from navigation actions so tests compose like user journeys.

Common ownership:

- `WikipediaAppRobot`: launch root and first screen access.
- `UITestRobot`: shared primitives such as waits, taps, drag gestures, screenshots, back-button resolution, and failure descriptions.
- Screen robots: screen-specific selectors, waits, scrolling, localized fallbacks, and semantic assertions.

Forward `file` and `line` through robot assertions so failures point at the calling test. Keep robots thin; do not turn one helper into a second framework.

For WebView/article DOM interactions, remember prior failures: XCTest can see labeled WebKit elements that are offscreen or falsely non-hittable. Prefer a robot-owned time-bounded lookup that scrolls until the target's tappable point is on screen, then taps the resolved coordinate.

Do not use `app.terminate()` as routine cleanup. Dismiss transient surfaces, wait for them to disappear, and assert the underlying screen remains visible. Split independent flows into separate tests when a clean launch is needed.

### 5. Use stable accessibility contracts

Prefer shared accessibility identifiers over visible text. The shared identifier home is:

```text
WMFComponents/Sources/WMFComponents/Utility/AccessibilityIdentifiers.swift
```

Use the Objective-C bridge only for identifiers legacy Objective-C code needs to set. Swift-only app/test identifiers should remain Swift constants.

Use localized visible text only when localization is itself the behavior under test, or as a robot-layer fallback for legacy system controls that do not expose stable identifiers. Do not scatter localized selector fallbacks in test files.

For article WebView test hooks, annotate real clickable DOM elements only. Do not create fake tappable nodes or change production behavior just to make a test pass. Keep language-specific labels/targets in focused configuration near the article robot or page-content test hook, not in `UITestConfiguration`.

### 6. Keep launch and network wiring centralized

Launch with:

```swift
launchWikipediaAppRobot(onboardingState: .completed)
```

Add launch settings through `UITestConfiguration` and `UITestLaunchArgument`, not ad hoc `XCUIApplication.launchArguments` in individual tests. Let the test plan own language, locale, theme, and appearance.

Network mode belongs in process arguments and app-boundary wiring:

- Fixture mode forwards `-WMFUITestHTTPClientProfile fixture-strict` to the app.
- E2E mode passes `e2e` to the UI-test process and should not activate app-process fixtures.
- Unknown non-empty fixture profile values should fail closed, not silently fall through.

When adding fixture-backed coverage, route both legacy `Session` traffic and newer `WMFBasicService(urlSession:)` traffic through the existing fixture seam when relevant.

### 7. Add fixtures like production evidence

For fixture-backed tests, fixtures should be exact API response bodies unless the user explicitly asks otherwise. Put recurring manifest routes in:

```text
WikipediaUnitTests/Fixtures/TestNetworkFixtures.json
```

If this checkout still uses an older manifest name, inspect `UITestNetworkFixtureStore` and use the current manifest resource name.

Guidelines:

- Store feature fixtures under feature folders such as `WikipediaUnitTests/Fixtures/ArticleControls/<language-code>`.
- Add every required subresource route, including PCS JS/CSS, image bytes, summaries, media lists, and secondary article/history calls.
- Keep localized fixtures intentional. Skip irrelevant language/config cases with XCTest skip APIs instead of encoding broad conditional behavior in the test.
- Use `ignoreQuery` or structured `queryItems` only when the query is actually unstable.
- Validate manifest JSON after edits.

### 8. Validate narrowly, then broaden only when warranted

For UI-test helper or launch-contract changes:

```bash
scripts/lint-ui-tests.sh
```

For focused local iteration:

```bash
xcodebuild test \
  -scheme WikipediaUITests \
  -project Wikipedia.xcodeproj \
  -testPlan UITests \
  -only-test-configuration "English (Light)" \
  -only-testing:WikipediaUITests/<ClassName>/<testMethod> \
  -destination "platform=iOS Simulator,name=iPhone 16" \
  -resultBundlePath /tmp/<case-name>/result.xcresult
```

For E2E-specific coverage:

```bash
xcodebuild test \
  -scheme WikipediaUITests \
  -project Wikipedia.xcodeproj \
  -testPlan UITests \
  -only-test-configuration "English (Light, E2E)" \
  -only-testing:WikipediaUITests/<ClassName>/<testMethod> \
  -destination "platform=iOS Simulator,name=iPhone 16" \
  -resultBundlePath /tmp/<case-name>/e2e-result.xcresult
```

When the change affects language, theme, RTL behavior, fixture skip logic, or the final confidence surface requested by the user, run the selected test or class through the applicable test-plan configurations without restricting to one configuration. Do not run the whole plan by default when a selected method/class proves the contract.

If a local or CI run fails, inspect the `.xcresult` first:

```bash
xcrun xcresulttool get test-results summary --path /tmp/<case-name>/result.xcresult
```

For visual or intermittent failures, switch to `$debug-ui-test-failure`: enable visible touches, record the simulator, inspect `ffmpeg` contact sheets, and fix from observed behavior.

## Pitfalls

- Do not run `swift-format` unless the user explicitly asks for formatting.
- Do not put language/theme/appearance overrides directly in tests; use the test plan and `UITestConfiguration`.
- Do not sample pixels/screenshots for color/theme checks when UI-test accessible properties or app-owned identifiers can expose the state directly.
- Do not choose localized labels when a stable identifier exists.
- Do not change app behavior just because a UI test is flaky; first prove the UI contract from result bundles, screenshots, recordings, or robot evidence.
- Do not add broad scaffolding when a small robot method, fixture route, or accessibility identifier fixes the actual gap.
- Check `git diff --check` after edits, and watch for build phases that may run lint/correction and churn unrelated Swift files.
- Update `SPEC.md` and tracked UI-test docs when the implementation establishes a durable contract, but do not count markdown/spec edits as validation.
