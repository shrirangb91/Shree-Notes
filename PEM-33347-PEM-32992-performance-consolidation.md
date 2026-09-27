# PEM-33347 and PEM-32992 Performance Consolidation

Date: 2026-09-27. Scope: Jira evidence, merged PR code, current PEM 1.0 and PEM 2.0 checkouts, and recorded development benchmarks.

## Executive Summary

The slow UI was not one rendering defect. At 25,000 partners and approximately 250,000 participant activity instances, each screen could trigger repeated authorization, expensive page selection, exact counts, full-population statistics, and additional browser initialization work. Concurrent activity execution amplified the cost of those reads. Later development investigations used approximately 500,000 matching instances, a different dataset from the original ticket.

The changes reduce this work at several levels: indexes support the actual joins and filters; database aggregates replace unbounded Java-side counting; batching removes per-row queries; page selection happens before display enrichment; the dashboard combines redundant calls; My Activities skips unused statistics and redundant context requests; and bounded first-rows hints make the database use an efficient shallow-page access path.

**Measured improvement is real, but full resolution is not established.** The initial index reduced a reported single-user response from 39 seconds to 4 seconds. Later Db2 comparisons reduced page selection from roughly 2.2-2.35 seconds to 3.4-4.7 milliseconds. However, exact counts and dashboard aggregates still took seconds. Jira's latest reports retain over-20-second single-user and over-12-second post-load observations. No final passing four-hour, 40-user retest of the current branch was found.

## 1. Evidence and Branch State

| Source                                                  | Inspected State                                                  | Interpretation                                                               |
| ------------------------------------------------------- | ---------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| [PEM-32992](https://jira.syncsort.com/browse/PEM-32992) | Reopened; fix version 6.3.1.0                                    | Original My Activities complaint and later sustained-load failures           |
| [PEM-33347](https://jira.syncsort.com/browse/PEM-33347) | In Progress; fix version 6.3.1.0                                 | Broader dashboard/login and post-load slowness                               |
| PEM 1.0 repository                                      | `shrirang/staging/pem20`, HEAD `cfc01fb75e`                      | Shared database migration changes supporting PEM 2.0                         |
| PEM 2.0 repository                                      | `shrirang/staging/pem20`, HEAD `de5add338`                       | Backend, UI, query-plan and request-reduction changes                        |
| Local integration references                            | PEM `origin/staging/pem20` at `42ae7a718d`; PEM20 at `771605a08` | Current personal branches contain additional commits beyond these references |

Both local HEADs match their locally recorded `origin/shrirang/staging/pem20` references. Remote refs were not refreshed during this analysis. Jira development-panel retrieval was unavailable; PR code was inspected from local Git merge commits and their parent diffs, supported by existing PR review reports. This is not a fresh review of all remote PR discussion threads.

Existing uncommitted configuration, authentication, proxy and generated-file changes were not modified or counted as released performance fixes.

### PR Lineage

| Repository | PR / Commit                              | Change                                                                                                            |
| ---------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| PEM        | PR #4521, merge `ebd48437b8`, 2026-08-04 | PEM-32992 sponsor/modify-time index                                                                               |
| PEM        | PR #4644, merge `eade8d9a56`, 2026-09-16 | Four PEM-33347 activity indexes; schema-only PR                                                                   |
| PEM20      | PR #2920, merge `fc03194fb`, 2026-09-17  | Main backend/UI performance PR; 28 files in the merge delta                                                       |
| PEM        | `4093510ef2`                             | Later resource-domain covering index on the current personal branch                                               |
| PEM20      | `41994f456`, `7bb41212e`, `6701b60f6`    | Later lean/two-stage list and dashboard request optimizations                                                     |
| PEM20      | `47a507ea5`                              | Permission batching and lean user-detail retrieval                                                                |
| PEM20      | `fccff601d`, `d121ff01d`                 | Combined dashboard endpoint, timing instrumentation, dialect hints, removal of obsolete standalone dashboard mode |
| PEM20      | `849178669`, `de5add338`                 | Regular-list first-page hint and My Activities context reuse                                                      |

The last five rows of follow-up work are present in the inspected personal branches but not in the corresponding local `origin/staging/pem20` history. Build 1265 must not be assumed to contain them.

## 2. Ticket and Measurement Chronology

| Date / Source                              | Workload or Build                                   | Observation                                                                                                                         | What It Establishes                                                                              |
| ------------------------------------------ | --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| July, PEM-32992                            | 25K partners, 10 rollouts each                      | My Activities over 39 seconds with one user                                                                                         | Original scale problem                                                                           |
| August 5-11, PEM-32992                     | Build 1088 after the first index                    | Single-user response 39 to 4 seconds                                                                                                | About 90% lower elapsed time in the reported comparison, not a universal result                  |
| August 11, PEM-32992                       | 40 concurrent users, four hours                     | 95th-percentile response still over 39 seconds                                                                                      | Index-only fix did not resolve sustained-load behavior                                           |
| PEM-33347 description                      | Fresh deployment versus four-hour load              | Login/UI initially 2-5 seconds, then 25-30 seconds                                                                                  | Reported degradation after workload; not proof of a memory leak                                  |
| August 27, PEM-33347                       | Personal build .39, 250K instances                  | Dashboard requests 41 seconds, 22 seconds with DELAYED, 10 seconds with inactivity; My Activities about 4 seconds                   | Different request shapes have different costs                                                    |
| September 8, PEM-33347                     | Personal build .53, 10K partners                    | API still over 10 seconds; indexes confirmed present                                                                                | Index existence alone did not ensure a fast plan                                                 |
| September 11, both tickets                 | Comment names build .60 and identity cycle_160      | Backend results described as expected; visible UI lag remained after responses                                                      | Backend and post-response rendering delays were separate issues                                  |
| September 15, both tickets                 | Local UI changes; screenshot attachment             | Significant improvement reported; My Activities screenshot shows 250K items and approximately 5.00 seconds for its activity request | Interactive evidence, not concurrent-load certification                                          |
| September 18, both tickets                 | Build 1265                                          | Main code reported merged and deployed                                                                                              | Deployment statement for the original PR set                                                     |
| September 19, PEM-33347                    | Build 1265, identity cycle_173, over 250K instances | Still over 20 seconds with one user                                                                                                 | Main merged PR set was not sufficient in that environment                                        |
| September 24, PEM-32992                    | Latest 6.3.1.0 image, 40 users, four hours          | My Activities still over 12 seconds                                                                                                 | Latest ticket-level sustained-load failure                                                       |
| September 24-25, development investigation | Cashbank, approximately 500K matches                | Page-selection improvements measured; count/aggregate remained dominant                                                             | Stronger phase-specific evidence for subsequent changes, not a replacement for the Jira workload |

The September 11 attachment filename references image 53 while the comment names image 60. Preserve that ambiguity rather than assuming the filename identifies the tested deployment.

The September 15 [UI test attachment](https://jira.syncsort.com/secure/attachment/1920437/PEM33347-TestingPerformanceBugWithUIChanges.docx) contains screenshots, not a percentile table. Its dashboard shows 250,000 items and activity request timings including approximately 4.75 seconds. The screenshots contain accumulated Network entries, so they do not establish a precise complete page-load duration or isolated call-count benchmark. All attached HAR files were not independently parsed in this consolidation.

## 3. Why a Ten-Row Page Could Take Seconds

`pageSize=10` limits returned rows, not all work performed for the response:

1. Authentication and authorization can issue several role, permission and ownership queries before the activity query starts.
2. The database may scan, join and sort hundreds of thousands of candidates before selecting ten keys.
3. An exact pagination count evaluates the entire matching population.
4. Statistics evaluate the matching population again, even though the result is one object.
5. Several dashboard requests can repeat those operations concurrently for the same scope.
6. The browser can delay rendering after data arrives while initialization and locale resources complete.

At sponsor scope, almost every row can qualify. A sponsor index is then not selective, and an optimizer can choose a scan/hash-join/sort plan despite the requested small page. Partner-scoped and sponsor-scoped timings are not interchangeable. A sponsor user's missing partner key is not automatically a scoping bug: the view can legitimately span the sponsor's activity population.

Repeated scans and round trips increase shared database and connection demand under concurrency. This is a source-supported explanation for poor scaling, not proof of the exact CPU, lock, pool, GC or I/O cause of the four-hour degradation.

## 4. PEM 1.0: Shared Schema Changes

The relevant changes in the PEM 1.0 repository are database migrations, not a rewrite of the legacy UI. PEM 2.0 queries use these shared tables and indexes.

Source: [changelog-61.0.xml](../pem/com.ibm.vch.systemdata/PEM/changelog/changelog-61.0.xml).

| Change Set    | Index Columns                                                                     | Reason                                                                             |
| ------------- | --------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `08042026_01` | `VCH_PCPT_ACTIVITY_INST (SPONSOR_KEY, MODIFY_TS DESC)`                            | Supports sponsor filtering and newest-first page access; initial PEM-32992 fix     |
| `20260903_01` | `VCH_PCPT_ACTIVITY_INST (ACTIVITY_INST_DETAILS)`                                  | Indexes the actual participant-to-activity association column                      |
| `20260904_01` | `VCH_ACTIVITY_INST (ACTIVITY_DEFN_VERSION_KEY)`                                   | Supports the PEM2 eligibility predicate excluding legacy rows                      |
| `20260904_02` | `VCH_PCPT_ACTIVITY_INST (PARTNER_KEY, PCPT_ACTIVITY_INST_STATUS, MODIFY_TS DESC)` | Supports partner/status-filtered My Activities access                              |
| `20260904_03` | `VCH_ACTIVITY_INST (SPONSOR_KEY, ACTIVITY_DEFN_VERSION_KEY, MODIFY_TS DESC)`      | Supports sponsor/version filtering in Rolled Out Activities                        |
| `20260924_03` | `VCH_RESOURCE_DOMAIN (RESOURCE_TYPE, DOMAIN, RESOURCE_KEY)`                       | Covers type/domain membership lookups with the resource key available in the index |

Indexes involving PEM2-only columns are restricted to Db2, Oracle and SQL Server, avoiding columns absent from the Derby test schema. The original/base-column indexes do not need that restriction.

Two important qualifications:

- The physical activity join uses `ACTIVITY_INST_DETAILS -> ACTIVITY_INST_KEY`, not the participant's separate plain `ACTIVITY_INST_KEY` field.
- A composite index with version/status before timestamp does not necessarily satisfy global timestamp ordering when multiple versions/statuses qualify. Its presence is not proof that sorting disappears. The [schema PR review](pr-4644-pem-PEM-33347.md) explicitly records this limitation.

## 5. PEM 2.0: Main Merged PR

### Database-Side Statistics

In Rolled Out Activities, `ActivityInstCustomRepoImpl.getActivityInstStats` previously selected every matching activity tuple, including unused display information, and counted statuses with Java streams. It now returns conditional sums from one database aggregate. This removes population-sized transfer, object allocation and application-side counting. The database still examines qualifying rows, but the JVM receives only aggregate values.

Participant statistics were consolidated from status/progress queries plus a separate completed-row fetch into one aggregate. Completed on-time/late classification also runs in the database instead of fetching completion/due dates for every completed instance. Four delayed-status breakdown fields are available for chart reuse.

The final date conversion is **not** the intermediate `extract(EPOCH)` approach described by an early commit title. It reconstructs the due timestamp with bounded whole-day and millisecond-remainder duration arithmetic, addressing SQL Server range concerns. Numeric aggregate results are read through `Number`, avoiding assumptions that every sum returns `Long`. Inactivity predicates are applied consistently to list and count, and request-local time/arguments replace shared mutable inactivity handling.

Sources: [ActivityInstCustomRepoImpl.java](../pem20/pem-api/pem-entity/src/main/java/com/precisely/pem/repositories/ActivityInstCustomRepoImpl.java), [ParticipantCustomRepoImpl.java](../pem20/pem-api/pem-entity/src/main/java/com/precisely/pem/repositories/ParticipantCustomRepoImpl.java#L575).

### Fewer Lookups and Better Joins

- Partner counts for an activity page use one `IN (...) GROUP BY activityInstKey` query instead of one `COUNT` per row. For ten activity rows, ten count calls become one grouped call.
- Correlated activity-definition and partner/company-name subqueries become reusable LEFT JOINs, including name filtering and sorting.
- Read-only, lazy entity associations enable those joins without changing which fields own writes. A detail-response mapping adjustment prevents the new partner association from losing existing partner information.

Sources: [PcptInstRepo.java](../pem20/pem-api/pem-entity/src/main/java/com/precisely/pem/repositories/PcptInstRepo.java), [ActivityInstServiceImpl.java](../pem20/pem-api/pem-services/src/main/java/com/precisely/pem/services/ActivityInstServiceImpl.java), [ActivityInst.java](../pem20/pem-api/pem-entity/src/main/java/com/precisely/pem/models/ActivityInst.java), [PcptActivityInst.java](../pem20/pem-api/pem-entity/src/main/java/com/precisely/pem/models/PcptActivityInst.java).

### Separate List and Statistics Work

The existing `/v2/pcptActivities` API gained `statsOnly=true` and `includeStats=false`. Statistics-only calls skip rows, pagination counts and enrichment. List-only calls skip aggregation. Defaults retain the earlier behavior for consumers not opting in; authorization and scope still apply.

The original merged dashboard used statistics-only calls for tiles/charts and list-only calls for rows. It did **not** eliminate all six separate statistics calls: that reduction belongs to subsequent branch work. Monitoring maintains separate statistics models, avoids re-fetching identical statistics merely for pagination/sort changes, and protects against older statistics responses overwriting newer filters. Its internal sponsor-action tile also uses the internal statistics model.

Sources: [ParticipantActivityInstanceController.java](../pem20/pem-api/pem-rest/src/main/java/com/precisely/pem/controller/ParticipantActivityInstanceController.java#L52), [monitoring-dashboard-page.js](../pem20/pem-ui/apps/web/src/modules/monitoring/monitoring-dashboard-page.js).

### UI Initialization

The merged PR upgrades `@b2bi/shell` from 1.1.55 to 1.1.57 and resolves `@b2bi/components` from 1.1.37 to 1.1.39. It also adds `Cache-Control: public, max-age=3600` for locale resources. Browser reuse can avoid re-downloading roughly 45 locale namespace files on repeated initialization; it does not eliminate a cold first load or make a slow SQL statement faster. Individual latency gains cannot be attributed to the package upgrades from the version change alone.

Source: [LocaleCacheControlFilter.java](../pem20/pem-ui/ui-app/src/main/java/com/precisely/pem/ui/filter/LocaleCacheControlFilter.java), [web.xml](../pem20/pem-ui/ui-app/src/main/webapp/WEB-INF/web.xml), [package.json](../pem20/pem-ui/apps/web/package.json).

## 6. Current-Branch Follow-Ups

### Select Keys Before Enrichment

The regular list now first selects only this page's participant keys, joining additional tables only when filtering/sorting requires them. A second query enriches those keys. Display-only joins no longer travel through the full candidate-set sort. The dashboard selects eight fields instead of the regular list's 22 and skips activity-definition display joins and unused owning-organization name resolution.

The general list can infer a total from a short page rather than running a count. This does not help a full first page at the reported scale. It is also not the same as count-free pagination: ordinary full-page requests still await their exact total.

Sources: [page selection and enrichment](../pem20/pem-api/pem-entity/src/main/java/com/precisely/pem/repositories/ParticipantCustomRepoImpl.java#L243), [dashboard mapper](../pem20/pem-api/pem-services/src/main/java/com/precisely/pem/services/ParticipantActivityInstServiceImpl.java#L395).

### One Combined Dashboard Request

`GET /v2/pcptActivities/dashboard` returns lean rows, exact filtered pagination and scope-wide statistics. One aggregate computes the fourteen statistics plus a conditional `listTotal`; that total replaces a separate pagination count. For a nonempty page, the business-query sequence is aggregate, page keys, enrichment and batched domain decoration.

The UI loads only the visible partner/internal scope, derives tiles and donut charts from the same response, and loads the other scope on demand. The old pattern could issue two list requests plus six statistics requests. The current initial visible-scope path uses one combined activity request, excluding unrelated application requests. It also removes the extra dashboard user-context fetch. This reduces repeated authorization and broad database work, not just HTTP overhead.

Status/progress/activity/name filters constrain rows and their total; sponsor, exact owning domain, partner key and internal/partner scope constrain both rows and statistics. The aggregate and page run sequentially with a shared time boundary. A read-only transaction does not imply a repeatable snapshot under concurrent writes. Rows now wait for the combined aggregate; reduced total work does not guarantee earlier first-row display in every workload.

`Server-Timing` exposes page-key and aggregate execution/fetch durations, excluding authentication, enrichment, mapping, serialization and network overhead. The obsolete public `dashboardView=true` mode was removed; internal repository dashboard projection remains active.

Sources: [combined controller](../pem20/pem-api/pem-rest/src/main/java/com/precisely/pem/controller/ParticipantActivityInstanceController.java#L104), [combined repository](../pem20/pem-api/pem-entity/src/main/java/com/precisely/pem/repositories/ParticipantCustomRepoImpl.java#L223), [dashboard.page.js](../pem20/pem-ui/apps/web/src/modules/dashboard/dashboard.page.js), [dashboard-data-loader-config.js](../pem20/pem-ui/apps/web/src/modules/dashboard/dashboard-data-loader-config.js).

### First-Rows Query Plans

The database is explicitly told to favor early rows for eligible shallow page-key queries: Db2 uses `OPTIMIZE FOR n ROWS`, Oracle uses `FIRST_ROWS(n)`, and SQL Server uses `FAST n`. A registered dialect resolver selects capable dialects; an explicit standard dialect override can bypass this behavior.

The preference is bounded to an offset-plus-size target of at most 1,000 and a single `modifyTs` sort. Dashboard requests additionally require a known in-range total; regular lists permit it only on page zero. Counts, aggregates and enrichment queries are not hinted. This is a plan preference, not a limit on accessible data or a relaxation of filters.

Source: [hint eligibility](../pem20/pem-api/pem-entity/src/main/java/com/precisely/pem/repositories/ParticipantCustomRepoImpl.java#L405), [DashboardDialectResolver.java](../pem20/pem-api/pem-entity/src/main/java/com/precisely/pem/dialect/DashboardDialectResolver.java).

### Authorization and User Context

Permission evaluation batches and deduplicates root roles, expands shared subroles together, loads permissions once for candidate resource paths, and reuses caller-role data across relevant checks. User-specific grants remain a fallback. These are invocation-local reductions, not cross-request permission caching.

Single-user detail no longer joins unused role tables that multiplied rows; UI role details still load separately. Sponsor display names reuse the current tenant name when the ownership keys match. Case-insensitive key predicates remain, so this is not a claim that all user lookups became index seeks.

My Activities now uses `userContext.partnerKey` instead of awaiting another user-detail/context call, and sends `includeStats=false`. The participant lookup on the task page also disables statistics it never reads. These changes remove work on frequently visited screens.

Sources: [PemPermissionsService.java](../pem20/pem-api/pem-services/src/main/java/com/precisely/pem/services/PemPermissionsService.java), [single-user lookup](../pem20/pem-api/pem-entity/src/main/java/com/precisely/pem/repositories/ParticipantCustomRepoImpl.java#L1295), [UserServiceImpl.java](../pem20/pem-api/pem-services/src/main/java/com/precisely/pem/services/UserServiceImpl.java), [partner-activites-dashboard-page.js](../pem20/pem-ui/apps/web/src/modules/partner-activities/partner-activites-dashboard-page.js#L512), [partner-tasks-page.js](../pem20/pem-ui/apps/web/src/modules/partner-activities/partner-tasks-page.js#L623).

### Correctness Changes Must Be Identified Separately

The current branch also removes inherited/descendant participant-activity API flags and consistently uses exact owning-domain visibility. This is a scope/contract change, not merely a faster SQL implementation; test comparisons must use equivalent authorized scopes.

Delayed chart aggregation now recognizes normalized all-progress input and uses projected delay consistent with the aggregate. The explicit `progress=DELAYED` list filter still uses a passed-due-date definition. This repairs chart data, but must not be described as every progress definition being identical. The internal dashboard filter also targets the internal table rather than the partner table.

## 7. Quantified Development Results

The following figures come from the existing [Cashbank investigation](api-performance-cashbank-20260924.md), not new benchmarks performed for this consolidation. They use approximately 500,000 matches and must not be mixed with the original 250K-instance Jira workload as one controlled before/after series.

| Measurement                                    | Before                                        | After / Remaining                     | Meaning                                                          |
| ---------------------------------------------- | --------------------------------------------- | ------------------------------------- | ---------------------------------------------------------------- |
| Combined endpoint, user-supplied phase timings | Page 2,923 ms; aggregate 3,780 ms             | About 6.7 seconds combined phase time | Both phases initially expensive                                  |
| Three-round page-key JDBC comparison, median   | 1,943.39 ms                                   | 5.17 ms with first-rows preference    | Measured page-plan improvement; not HTTP latency                 |
| Integrated dashboard query, three rounds       | 2,370.32 / 2,106.89 / 2,235.56 ms             | 10.07 / 4.59 / 3.97 ms                | Actual repository-generated hint verified                        |
| Combined aggregate in integrated rounds        | Unchanged                                     | 2,640.03 / 2,628.55 / 2,874.03 ms     | Still prevents a two-second complete response in these samples   |
| Integrated regular-list page, three rounds     | 2,203.4 / 2,349.1 / 2,290.5 ms                | 4.7 / 4.2 / 3.4 ms                    | Same page optimization extended to My Activities route           |
| Regular-list exact count in those rounds       | Still required                                | 1,968.9 / 1,945.3 / 1,464.3 ms        | Dominant remaining synchronous work                              |
| Implemented regular-list repository total      | Original warmed comparisons about 3.8 seconds | 2,002.8 / 1,977.8 / 1,495.5 ms        | Substantial repository improvement, not a one-second HTTP result |

The captured dashboard plan changed from activity-table scans, hash joins and sorting to ordered access through the existing sponsor/modify-time index, followed by nested-loop eligibility/domain probes, with no sort. This explains the measured shallow-page improvement. First-rows optimization trades first-result latency against full-result throughput; it is not universally preferable.

For the regular-list comparisons, all totals matched but some page key sets differed because many rows had the same `MODIFY_TS`. Ordering only by that timestamp does not define a stable tie order. Current code retains that behavior; deterministic multi-page traversal needs separate validation and any additional ordering must be rebenchmarked.

## 8. Reverted, Experimental or Absent Work

| Work                                                    | Verified Interpretation                                                                                                                                                          |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Intermediate `extract(EPOCH)` completed-date conversion | Replaced by bounded duration arithmetic; do not document it as final code                                                                                                        |
| Resource-domain `IN` / `= ANY` experiment               | Reverted after no demonstrated improvement; current code uses correlated `EXISTS`                                                                                                |
| Cross-request participant statistics cache              | Removed; current aggregate is a live query                                                                                                                                       |
| Eleven-column participant aggregate covering index      | Reported latency regression to about 7 seconds versus about 3.4 after removal; not present in the current migration                                                              |
| Related activity key/version covering index             | Not in the current migration, but September 25 catalog evidence recorded it still present in the development DB; source and deployed schema must be checked separately           |
| Four-column diagnostic count index                      | Tested and dropped: estimated plan cost improved about 48%, but repository medians were 1,563.9 ms before, 1,720.1 with it, and 1,766.1 after removal; no reliable material gain |
| Simple count INNER JOIN / activity EXISTS rewrites      | Median roughly 2.27 / 2.32 / 2.20 seconds; insufficient gain to justify adoption                                                                                                 |
| Transactionally maintained counts                       | Prepared historical design was removed and never activated; not a current performance mechanism                                                                                  |
| Rows-first `/rows` and asynchronous `/count`            | Recorded as locally implemented in the older investigation's Section 19, but absent from the current controller and My Activities loader path                                    |

The final row is an important reconciliation issue. The current worktree contains an untracked [ParticipantActivityInstCountResp.java](../pem20/pem-api/pem-entity/src/main/java/com/precisely/pem/dtos/responses/ParticipantActivityInstCountResp.java), but that record alone does not provide either route or wire the UI. The historical notes do not establish why the implementation is absent now. Do not claim the current My Activities screen renders independently of its exact count.

The separately proposed `ACTIVITY_DEF_KEY` index is also absent from the inspected migration. Intermediate server/container switches and unrelated local JWT/proxy/build settings are not credited as final performance fixes.

## 9. Validation and Remaining Gates

Recorded validation includes 312 tests for an earlier PR review, 380 for the combined workflow, and 123 for the later regular-list hint. These are overlapping historical runs at different revisions, not an additive test total or a fresh validation of today's worktree. The current files contain repository/H2, service/controller and dialect regression coverage. The [PR review](pr-2920-pem20-PEM-33347-r3.md) and [development investigation](api-performance-cashbank-20260924.md) describe the exact scopes and limitations.

Coverage addresses aggregate/date boundaries, joined filters, scope isolation, duplicate ownership records, filtered totals versus scope-wide statistics, API parameters, query counts and dialect hint rendering. H2 and generated vendor SQL do not establish live Oracle/SQL Server behavior, production migration success or load-test performance. No application tests, database changes or new live benchmarks were run for this documentation-only consolidation.

Before declaring these defects resolved:

1. Establish an exact deployment manifest containing both schema and application revisions; distinguish build 1265 from subsequent branch work. Reconcile the missing rows/count implementation if it is intended for release.
2. Repeat the original 25K-partner, ten-rollout, 40-user, four-hour scenario. Record cold/warm and before/during/after-load p50/p95/p99, error rates and visible-page timing, not just individual SQL timings.
3. Correlate authenticated HTTP latency with authorization, connection waits, page selection, enrichment, exact count, aggregate and rendering. Existing dashboard `Server-Timing` covers only two of those phases.
4. Validate hints and actual index use on deployed databases; test sparse filters, ties, deep pages and concurrent writes. Record write overhead from retained indexes.
5. Address the remaining exact-count/aggregate budget explicitly. Rows-first rendering with a separate exact count changes the response workflow, not the cost of counting; maintained summaries require an approved consistency/write-path design. Neither benefit should be claimed from absent code.

## Conclusion

The defensible outcome is: **the work removes several demonstrated scalability bottlenecks and materially improves page-selection and single-user performance, but it has not yet demonstrated resolution of the original sustained-load problem.**

The main causal improvement is less repeated work: fewer network and authorization cycles, fewer per-row lookups, no population-sized Java fetch for the rewritten statistics paths, page-bounded enrichment, and efficient early-row access. The remaining limiting operations are exact totals and full-population aggregates, plus any runtime contention and browser delays that a correlated load test still needs to quantify.
