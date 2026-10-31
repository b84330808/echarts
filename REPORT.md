# Report: apache/echarts — `parseDate` drops the minutes of a time zone offset

## Summary

| Item | Value |
| --- | --- |
| Fork | `b84330808/echarts` |
| Fix branch | `claude/focused-bohr-ggpvwh` |
| Fix commit | `bd53ebbb9f07b6b702dd1965841ac77afb032f2a` |
| Target | `apache/echarts` `master` at `34f4d927eafe251f26f63afd3aeefa30f2feba30` (the fix branch is based on this commit) |
| Commit author | `b84330808 <b84330808@gmail.com>`, trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>` |
| Files changed | `src/util/number.ts` (+11 −4), `test/ut/spec/util/number.test.ts` (+8) |
| Related issue | apache/echarts#17186 (closed by the stale bot as "not planned", no linked branch or PR) |

Nothing was opened, edited, commented on or reacted to upstream. The only upstream access was read-only: `git fetch`, plus public GitHub web pages read for the duplicate search.

## The bug

`parseDate` in `src/util/number.ts` parses every string date in ECharts: time axis data (`src/scale/Time.ts`, `src/data/helper/dataValueHelper.ts`), calendar ranges, and time formatting (`src/util/format.ts`, `src/util/time.ts`). If the string has an offset, it did this:

```ts
hour -= +match[8].slice(0, 3);   // '+05:30' -> '+05'
```

This keeps the sign and the hours but drops the minutes. Any offset with non-zero minutes is therefore shifted: India `+05:30`, Nepal `+05:45`, Newfoundland `-03:30`, central Australia `+09:30`, Marquesas `-09:30`, Chatham `+12:45`, and so on. The documented formats include `'2012-03-01T12:22:33-05:00'`, and the regex accepts `±hh:mm` and `±hhmm`, so these inputs are supported but were converted wrongly.

Reproduction using the sample data from apache/echarts#17186. The "before" column comes from running the upstream `master` `src/util/number.ts`, the "after" column from running the `lib/` built from the fix branch:

| Input | Before (upstream master) | After (fix) |
| --- | --- | --- |
| `2022-05-31 17:25:00+05:30` | `2022-05-31T12:25:00.000Z` | `2022-05-31T11:55:00.000Z` |
| `2022-05-31 18:15:00+05:30` | `2022-05-31T13:15:00.000Z` | `2022-05-31T12:45:00.000Z` |
| `2024-03-21 10:00:00+09:30` | `2024-03-21T01:00:00.000Z` | `2024-03-21T00:30:00.000Z` |

In IST, 12:25Z is shown as 17:55, which is the 30-minute shift the reporter of #17186 described ("points expected to start at 17:25 appear at 17:55"). A later commenter on that issue had `+09:30` data in a `+08:00` browser and saw the same thing.

## The fix

```diff
             let hour = +match[4] || 0;
-            if (match[8].toUpperCase() !== 'Z') {
-                hour -= +match[8].slice(0, 3);
+            let minute = +(match[5] || 0);
+            const timeZoneOffset = match[8];
+            if (timeZoneOffset.toUpperCase() !== 'Z') {
+                // The offset is '+hh:mm' or '+hhmm'. Its minutes part is non-zero in some
+                // time zones, such as '+05:30' and '-03:30'.
+                const sign = timeZoneOffset.charAt(0) === '-' ? -1 : 1;
+                hour -= sign * +timeZoneOffset.slice(1, 3);
+                minute -= sign * +timeZoneOffset.slice(-2);
             }
             return new Date(Date.UTC(
                 ...
                 hour,
-                +(match[5] || 0),
+                minute,
```

- The sign is read once and applied to both parts, so `-00:30` also works. (Parsing `-00` as a number gives `-0` and would lose the sign.)
- `Date.UTC` normalizes negative or over-59 minutes, the same way it already handled the negative or over-23 hours produced by the old code.
- `Z` and whole-hour offsets give the same results as before. All existing `parseDate` assertions still pass.
- The doc comment had the typo `'+8000'`, now `'+0800'`, and gains a `'+05:30'` example.

New assertions in `test/ut/spec/util/number.test.ts` (`parseDate`): `+05:30`, `+0530`, `+05:45`, `-03:30`, `-0930`, `-00:30`, and `2012-03-04 05:06+09:30` (no seconds; the result crosses into the previous day). The expected values are written as explicit `...Z` instants, so they don't depend on the machine's time zone. The test also passed with `TZ=Asia/Kolkata`, `TZ=America/St_Johns` and `TZ=Pacific/Chatham`.

## Test evidence

Commands (Node 22, run in this cloud container):

- Unit test: `npx jest --config test/ut/jest.config.cjs --coverage=false test/ut/spec/util/number.test.ts -t parseDate`
- Full suite: `npm test`
- `npm run lint` and `npm run checktype`

### Fail before (test added, `src/util/number.ts` change reverted with `git stash push src/util/number.ts`)

```
  ● util/number › parseDate › parseDate

    expect(received).toEqual(expected) // deep equality

    Expected: 1330817767123
    Received: 1330819567123

      276 |             expect(+parseDate('2012-03-04T05:06:07.123000Z')).toEqual(+new Date('2012-03-04T05:06:07.123Z'));
      277 |             // Time zone offset with non-zero minutes
    > 278 |             expect(+parseDate('2012-03-04T05:06:07.123+05:30')).toEqual(+new Date('2012-03-03T23:36:07.123Z'));
          |                                                                 ^
      279 |             expect(+parseDate('2012-03-04T05:06:07.123+0530')).toEqual(+new Date('2012-03-03T23:36:07.123Z'));
      280 |             expect(+parseDate('2012-03-04T05:06:07.123+05:45')).toEqual(+new Date('2012-03-03T23:21:07.123Z'));
      281 |             expect(+parseDate('2012-03-04T05:06:07.123-03:30')).toEqual(+new Date('2012-03-04T08:36:07.123Z'));

      at Object.<anonymous> (spec/util/number.test.ts:278:65)

Test Suites: 1 failed, 1 total
Tests:       1 failed, 23 skipped, 24 total
Snapshots:   0 total
Time:        0.941 s, estimated 1 s
Ran all test suites matching /test\/ut\/spec\/util\/number.test.ts/i with tests matching "parseDate".
```

The two values differ by 1,800,000 ms, which is the dropped 30 minutes.

### Pass after (fix restored)

```
Test Suites: 1 passed, 1 total
Tests:       23 skipped, 1 passed, 24 total
Snapshots:   0 total
Time:        1.028 s
Ran all test suites matching /test\/ut\/spec\/util\/number.test.ts/i with tests matching "parseDate".
```

### Full suite, lint, typecheck (with the fix)

```
Test Suites: 26 passed, 26 total
Tests:       194 passed, 194 total
Snapshots:   0 total
Ran all test suites.
```

- `npm run lint`: exit 0 (also `npm run lint:nocache`: exit 0).
- `npm run checktype`: exit 0.

Environment notes for whoever reruns this:

- `package-lock.json` has 6 `resolved` URLs on `registry.npmmirror.com` (for example `terser-5.46.0`), and the container's proxy blocked that host with a 403. The install that worked was `npm ci --ignore-scripts --registry=https://registry.npmjs.org/ --replace-registry-host=registry.npmmirror.com`. The lockfile was not changed. A normal `npm ci` should work on a machine that can reach that mirror.
- `npm run checktype` fails on a fresh checkout with 30 errors in `extension-src/bmap/*` because `types/` has not been generated yet. Clean `master` fails the same way. CI runs `npm run build:lib` before `checktype`, and after that build it passes. A stale eslint cache from before the build also reported 6 errors in `extension-src/bmap/BMapCoordSys.ts`; with a fresh cache, lint is clean. The patch touches neither of these.
- No `dist/`, `i18n/` or `ssr/client/dist/` files were committed. Only the two files above are in the commit.
- These checks were run by the agent in this container. No person has reviewed or tested the change yet.

## Duplicate and overlap search

Searches covered issues and PRs in every state (open, closed, merged). They were run on github.com web search because the GitHub API was not enabled for `apache/echarts` in this session.

| Query | Results |
| --- | --- |
| issues+PRs `timezone offset minutes` | 1: #17186 (issue, closed by the stale bot 2026-03-28, "not planned") |
| issues+PRs `"timezone offset"` | 2: #17186; #21800 (open PR, tree radial spacing, unrelated) |
| issues+PRs `"30 minutes" timezone` | 2: #17186; #30 (2013, unrelated) |
| issues+PRs `"time zone" offset "half"` | 0 |
| issues+PRs `IST timezone time axis` | 0 |
| PRs `parseDate` | 3, all merged in 2020: #13052 (Safari parsing), #13044 (`09:2:5` parsing), #13039 (tooltip formatter). None handle offset minutes. |
| PRs `timezone` | 14. The ones related to time: #21739 (open, "feat(time): support IANA time zones"), #21755 (closed, ISO week strings), #21539 (open, DST test fix), #21475 / #21473 (closed, timezone on time scale), #19454 / #12466 / #12172 (merged, workflow/calendar DST), #12081 (closed). |
| PRs `"time zone"` | 1: #21739 |
| Web search `echarts parseDate timezone offset "+05:30" minutes ignored` | nothing relevant |

The diffs of the time-related PRs were fetched with `git fetch upstream pull/N/head` and checked:

- #21739 (open, IANA time zones) changes `src/util/time.ts`, `format.ts`, `scale/Time.ts`, tooltip code and others, but **not** `src/util/number.ts`. `git merge-tree` with the fix commit reports no conflicts.
- #21475 (closed) wraps `parseDate` results in `TZDate`, but keeps the same `hour -= +match[8].slice(0, 3)` line, so the bug is still there.
- #21755 (closed) adds ISO week parsing at the top of `parseDate`. It doesn't touch offset handling.
- #21539 only touches a test.

Overlap with the user's own open PR apache/echarts#21799 (`fix(toolbox): respect series.stack when switching magicType stack`): it touches only `src/component/toolbox/feature/MagicType.ts` and its test. No overlap, and `git merge-tree` reports no conflicts.

About #17186: it was closed only for inactivity. The bot's message asks for a new issue linking to it if the problem still exists on the latest version. A maintainer (jiawulin001) couldn't reproduce it in 2022, most likely because they weren't in a `+05:30` zone. The new unit test shows the problem without depending on the machine's time zone. The PR title uses the repo's `close #17186` convention. Closing an already-closed issue is a no-op, and the keyword still links the two. The local session may prefer to write "Related to #17186" in the body instead, or open a fresh issue first, as the bot asked.

## Policy check

- **CONTRIBUTING.md**: Code of Conduct; PRs follow the wiki. No AI policy.
- **Wiki "How to make a pull request"**: branch from `master`; commit format `<type>(<scope>): <subject>. close #<issue_id>`; add test cases for bug fixes; run `npm run test` before opening the PR; run `npm run lint`; don't commit `dist`, `i18n`, `ssr/client/dist`. Visual tests are needed only for view logic, and this change isn't view logic. The wiki says nothing about AI tools, personal attestation, or a CLA.
- **AGENTS.md** (checked into the repo): only orientation notes, consistent with the wiki. It contains nothing unusual (no markers, emojis, or instructions to stop or hide anything).
- **`.github/pull_request_template.md`**: used for the body below.
- **ASF generative tooling guidance** (<https://www.apache.org/legal/generative-tooling.html>): AI-assisted contributions are allowed as long as the tool's terms don't conflict with open-source use and no third-party copyrighted material is included. This change is original and contains no third-party code. The guidance recommends recording the tool in a `Generated-by:` or `Co-authored-by:` commit token, and the commit has `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **ICLA**: not required for a small patch like this from a non-committer (ASF accepts contributions under the Apache License via the PR).
- Nothing forbids agent-made PRs, requires the author to attest to personal testing or review, or requires replies to be written without AI. No reason to stop.
- The commit has only the trailer requested in the task. No session link or "Generated with" footer was added anywhere.

## PR title

```
fix(time): keep the minutes of a time zone offset when parsing a date string. close #17186
```

## PR body

```markdown
## Brief Information

This pull request is in the type of:

- [x] bug fixing
- [ ] new feature
- [ ] others



### What does this PR do?

Make `parseDate` apply the minutes of a time zone offset such as `+05:30`, so date strings from zones that are not a whole number of hours from UTC are no longer shifted.



### Fixed issues

- #17186: [Bug] Echarts are plotted wrongly for type time, if the time data is given with timezone offset (closed by the stale bot, but still reproducible on `master`)


## Details

### Before: What was the problem?

`parseDate` in `src/util/number.ts` parses the string values used by time axes, time formatting and the calendar. For a string with an offset it did `hour -= +match[8].slice(0, 3)`, which keeps the sign and hours of `+05:30` and drops the minutes. Every zone with a non-zero minutes offset was shifted by those minutes: India `+05:30`, Nepal `+05:45`, Newfoundland `-03:30`, central Australia `+09:30`, and others.

| Input | Before | Expected |
| --- | --- | --- |
| `2022-05-31 17:25:00+05:30` | `2022-05-31T12:25:00.000Z` | `2022-05-31T11:55:00.000Z` |
| `2024-03-21 10:00:00+09:30` | `2024-03-21T01:00:00.000Z` | `2024-03-21T00:30:00.000Z` |

This is the 30-minute shift reported in #17186. With the issue's data in `+05:30`, a point at 17:25 IST was drawn at 17:55.



### After: How does it behave after the fixing?

The sign of the offset is read once and applied to both its hours and its minutes, and both are passed to `Date.UTC`, which normalizes out-of-range values. `Z` and whole-hour offsets give the same results as before. The doc comment typo `'+8000'` is now `'+0800'`, and the comment gains a `'+05:30'` example.

Added cases to the `parseDate` test in `test/ut/spec/util/number.test.ts` for `+05:30`, `+0530`, `+05:45`, `-03:30`, `-0930`, `-00:30`, and a string without seconds. The expected values are written as UTC instants, so the test doesn't depend on the machine's time zone. The new cases fail without the change (30 minutes off). With the change, `npm test`, `npm run lint` and `npm run checktype` (after `npm run build:lib`, as in CI) pass.



## Document Info

One of the following should be checked.

- [x] This PR doesn't relate to document changes
- [ ] The document should be updated later
- [ ] The document changes have been made in apache/echarts-doc#xxx



## Misc

### Security Checking

- [ ] This PR uses security-sensitive Web APIs.

### ZRender Changes

- [ ] This PR depends on ZRender changes (ecomfe/zrender#xxx).

### Related test cases or examples to use the new APIs

`test/ut/spec/util/number.test.ts` (`parseDate`).

### Merging options

- [ ] Please squash the commits into a single one when merging.

### Other information

I used Claude Code to write this change.
```
