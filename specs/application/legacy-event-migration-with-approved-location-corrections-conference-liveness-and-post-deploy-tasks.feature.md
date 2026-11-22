# Legacy event migration with approved location corrections, conference liveness and post-deploy tasks

_Extracted from `src/main/java/dev/ted/jittertravel/application`._

A one-time migration rewrites stored events into their current payload shape and schema version, renames retired type names, and applies location corrections Ted approved one by one. It reports its plan in a preview, refuses in read-only mode or when any row cannot be migrated, and checks the store again after writing to confirm the result. This part also covers the rule that decides whether a conference is still live, family notifications for cancelled trips, and post-deploy one-off tasks that Ted ticks off.

## S1: Migration refuses in read-only mode without touching the database @v1 [proposed]

- Given: the application is in read-only mode
- When: the migration is run
- Then: the result is marked as refused for read-only mode and has errors
- and: there is exactly one error, and it mentions "read-only"
- and: the event store is neither read nor written

_Source: `src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationReadOnlyTest.java`, `src/main/java/dev/ted/jittertravel/application/LegacyEventMigration.java`._

[test: migrateInReadOnlyModeRefusesAndWritesNothing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationReadOnlyTest.java#L21 ]

## S2: A legacy hotel event is rewritten to country codes and stamped with the current version while keeping its identity @v1 [proposed]

- Given: a stored HotelBooked event at schema version 2 whose address country is "UK"
- When: the migration is run
- Then: the run has no errors, one row is rewritten and none is only stamped
- and: the stored payload now has country "GB" and schema version 3
- and: the zoned check-in passes through unchanged
- and: the event's sequence, event id, command id and timestamp stay the same

_Source: `src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java`, `src/main/java/dev/ted/jittertravel/application/LegacyEventMigration.java`._

[test: migratesAStoredHotelToCountryCodesStampsCurrentVersionAndKeepsIdentity : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java#L111 ]

## S3: Running the migration a second time rewrites nothing @v1 [proposed]

- Given: the migration has already run over a legacy hotel event
- When: the migration is run again
- Then: there are no errors, and no rows are rewritten or stamped
- and: a preview reports zero events left to migrate

_Source: `src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java`, `src/main/java/dev/ted/jittertravel/application/LegacyEventMigration.java`._

[test: isIdempotentSecondRunRewritesNothing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java#L147 ]

## S4: An approved correction is applied in the same write as the shape upgrade @v1 [proposed]

- Given: the stored Devnexus 2027 ConferencePlanned event at version 3, with country "United States" and an empty region
- When: the migration is run
- Then: the stored payload has country "US" and region "GA"
- and: the row is stamped with schema version 4

_Source: `src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java`, `src/main/java/dev/ted/jittertravel/application/LocationDataCorrections.java`, `src/main/java/dev/ted/jittertravel/application/LegacyEventMigration.java`._

[test: appliesAnApprovedCorrectionInTheSameWriteAsTheRung : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java#L163 ]

## S5: The preview lists every approved correction and where it stands in this database @v1 [proposed]

- Given: a database holding only the Devnexus event, at sequence 7, out of the ten approved corrections
- When: a preview is taken
- Then: ten corrections are expected and one will be made
- and: the corrections are not all accounted for
- and: the Devnexus line names event 7 with the state WILL_BE_MADE
- and: the other nine lines have the state EVENT_NOT_FOUND and no event number

_Source: `src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java`, `src/main/java/dev/ted/jittertravel/application/LegacyEventMigration.java`._

[test: previewListsEveryApprovedCorrectionWithWhereItStandsHere : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java#L185 ]

## S6: After writing, the migration reads the store again to check that every row and correction landed @v1 [proposed]

- Given: a stored Devnexus event that needs a correction and a stored legacy hotel event
- When: the migration is run
- Then: the verification gives expected against actual: "Rows written" 2/2, "Rows still needing migration" 0/0, "Corrections now in the database" 1/1, "Airport ends now filled in" 0/0
- and: the result is verified
- and: a later preview finds the correction already made

_Source: `src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java`, `src/main/java/dev/ted/jittertravel/application/LegacyEventMigration.java`._

[test: migrateReReadsTheStoreAndConfirmsEveryRowAndFixLanded : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java#L212 ]

## S7: A second run does not count corrections made by the first run @v1 [proposed]

- Given: a first migration run has already applied the Devnexus correction
- When: the migration is run again
- Then: the verification shows "Corrections now in the database" as 0 expected and 0 actual

_Source: `src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java`, `src/main/java/dev/ted/jittertravel/application/LegacyEventMigration.java`._

[test: aSecondRunExpectsNoNewCorrectionsAndFindsNone : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java#L234 ]

## S8: A row that cannot be migrated aborts the whole migration with nothing written @v1 [proposed]

- Given: a stored event that cannot be upgraded or bound to its current shape, such as a hotel whose country cannot be named
- When: the migration is run
- Then: no row is written
- and: each failing row is reported as "Event <sequence> (<type>) cannot be migrated: <reason>"

_Source: `src/main/java/dev/ted/jittertravel/application/LegacyEventMigration.java`, `src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java`._

[test: oneUnbindableRowAbortsTheWholeMigrationLeavingEveryRowUntouched : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LegacyEventMigrationTest.java#L353 ]

## S9: A named correction changes its fields only while they still hold their approved old values @v1 [proposed]

- Given: an event that has an approved correction
- When: the correction is applied during migration
- Then: if every field holds its old value, every field is changed and the correction counts as applied
- and: if every field already holds the new value, it counts as already made
- and: otherwise nothing changes and the values are reported as changed since approval

_Source: `src/main/java/dev/ted/jittertravel/application/LocationDataCorrections.java`._

[test: anApprovedCorrectionChangesEveryFieldItNames : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LocationDataCorrectionsTest.java#L30 ]
[test: aCorrectionWhoseValuesHaveSinceChangedIsLeftAlone : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LocationDataCorrectionsTest.java#L42 ]
[test: reportsANamedFixAsAppliedThenAsAlreadyApplied : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LocationDataCorrectionsTest.java#L78 ]
[test: reportsANamedFixWhoseValuesHaveChangedAsDiffering : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LocationDataCorrectionsTest.java#L93 ]
[test: applyingItAgainChangesNothing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LocationDataCorrectionsTest.java#L66 ]
[test: aTopLevelFieldIsCorrectedAlongsideTheNestedOnes : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LocationDataCorrectionsTest.java#L138 ]

## S10: Ground transfer airport ends with no country are filled in from the airport table @v1 [proposed]

- Given: a GroundTransferPlanned event whose origin or destination has an airport code but no country
- When: the migration applies corrections
- Then: that end takes its country from the airport table, and its region too if the region is blank
- and: an airport code missing from the table is reported as not filled, with a blank region and country

_Source: `src/main/java/dev/ted/jittertravel/application/LocationDataCorrections.java`._

[test: anAirportEndWithNoCountryIsCompletedFromItsAirportCode : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LocationDataCorrectionsTest.java#L158 ]
[test: reportsEachAirportEndItFillsInAndOneItCannot : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LocationDataCorrectionsTest.java#L121 ]
[test: anEndWithoutAnAirportCodeIsNeverTouched : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LocationDataCorrectionsTest.java#L177 ]
[test: anAirportEndThatAlreadyHasACountryIsNeverTouched : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LocationDataCorrectionsTest.java#L191 ]
[test: anAirportCodeOnAnotherTypeIsNotTheAirportRule : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LocationDataCorrectionsTest.java#L205 ]

## S11: A conference is live only while it is planned and neither cancelled nor declined @v1 [proposed]

- Given: the event stream for a conference
- When: the commands check whether the conference is live
- Then: a planned conference is live and carries its plan
- and: a conference that was never planned, or that was cancelled or declined, is not live
- and: another conference's plan, cancellation or decline does not affect this one
- and: a TalkRejected leaves the conference live

_Source: `src/test/java/dev/ted/jittertravel/application/LiveConferencePlanTest.java`, `src/main/java/dev/ted/jittertravel/application/LiveConferencePlan.java`._

[test: aPlannedConferenceIsLiveAndCarriesItsPlan : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LiveConferencePlanTest.java#L35 ]
[test: aConferenceNeverPlannedIsNotLive : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LiveConferencePlanTest.java#L44 ]
[test: aCancelledConferenceIsNotLive : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LiveConferencePlanTest.java#L51 ]
[test: aDeclinedConferenceIsNotLive : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LiveConferencePlanTest.java#L59 ]
[test: anotherConferenceGoingAwayLeavesThisOneLive : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LiveConferencePlanTest.java#L67 ]
[test: aRejectionLeavesTheConferenceLive : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/LiveConferencePlanTest.java#L79 ]

## S12: A one-off task is recorded as completed only once and only if it is declared @v1 [proposed]

- Given: the post-deploy tasks declared in code
- When: Ted completes a task
- Then: a completion is recorded only if the task is declared and not already completed
- and: a repeat completion, or one for a task that is not declared, records nothing and reports that nothing was recorded
- and: the task list shows outstanding tasks first, oldest declaration first

_Source: `src/main/java/dev/ted/jittertravel/application/OneOffTasks.java`, `src/main/java/dev/ted/jittertravel/application/OneOffTaskProjector.java`, `src/main/java/dev/ted/jittertravel/application/OneOffTaskRegistry.java`, `src/main/java/dev/ted/jittertravel/application/OneOffTaskView.java`._

[test: completingATaskAppendsTheCompletionEvent : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/OneOffTasksTest.java#L67 ]
[test: completingAnAlreadyCompletedTaskWritesNothing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/OneOffTasksTest.java#L80 ]
[test: completingAnUndeclaredTaskWritesNothing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/OneOffTasksTest.java#L92 ]
[test: outstandingTasksComeFirstAndOldestDeclarationLeads : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/OneOffTasksTest.java#L58 ]
[test: aDeclaredTaskWithNoCompletionIsOutstanding : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/OneOffTasksTest.java#L34 ]
[test: aCompletedTaskDropsOutOfOutstandingButStaysInTheFullList : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/OneOffTasksTest.java#L43 ]
