# Backup and restore, booked-travel lists, and cancellations

_Extracted from `src/main/java/dev/ted/jittertravel/application`._

Part 1 of the application layer covers four things. The first is the event-oriented backup document: its metadata, filename and version, plus a restore that checks the file before writing and is refused in read-only mode. The second is the read models behind the booked flights, hotels, trains and itineraries lists, including flight change history and the show/hide switch for cancelled rows. The third is the folds that decide whether a flight, hotel, train, ground transfer, private event or itinerary can still be cancelled. The fourth is a small value rule on itinerary evaluations.

## S1: Restore in read-only mode is refused and writes nothing @v1 [proposed]

- Given: the instance is in read-only mode
- When: a backup document with version 2, no commands and no events is restored
- Then: the restore result has errors
- and: the only error mentions "read-only"
- and: nothing is written to the database

_Source: `src/test/java/dev/ted/jittertravel/application/BackupServiceReadOnlyTest.java`, `src/main/java/dev/ted/jittertravel/application/BackupService.java`._

## S2: Backup metadata records the source and the creation instant @v1 [proposed]

- Given: a backup is taken at 2026-08-11T14:30:00Z on the "production" instance
- When: the backup JSON is written
- Then: the metadata `source` is "production"
- and: the metadata `createdAt` is the instant 2026-08-11T14:30:00Z

_Source: `src/test/java/dev/ted/jittertravel/application/BackupServiceTest.java`, `src/main/java/dev/ted/jittertravel/application/BackupService.java`._

## S3: Backup filename and createdAt are normalised to UTC @v1 [proposed]

- Given: a backup is taken at 2026-08-11T16:30:00+02:00 on the "production" instance
- When: the backup is created
- Then: the filename is "jittertravel-backup-production-2026-08-11T143000Z.json"
- and: the metadata `createdAt` is the instant 2026-08-11T14:30:00Z
- and: a backup taken at 2026-08-11T14:30:00Z on the "local" instance is named "jittertravel-backup-local-2026-08-11T143000Z.json"

_Source: `src/test/java/dev/ted/jittertravel/application/BackupServiceTest.java`, `src/main/java/dev/ted/jittertravel/application/BackupService.java`._

## S4: Backups are written as version 3, and each event keeps its schema-version stamp @v1 [proposed]

- Given: the event log holds one HotelBooked event stamped with schema version 2
- When: the backup JSON is written
- Then: the document `version` is 3
- and: the event's `schemaVersion` is 2

_Source: `src/test/java/dev/ted/jittertravel/application/BackupServiceTest.java`, `src/main/java/dev/ted/jittertravel/application/BackupService.java`._

## S5: Restore checks the whole file and reports every problem before writing anything @v1 [proposed]

- Given: a backup whose version is missing or is not 2 or 3, or whose commands or events break the integrity rules
- When: the backup is restored or dry-run validated
- Then: an unsupported or missing version gives the error "Unsupported or missing backup version", and nothing else in the file is checked
- and: these problems each give one error: a duplicate command id, a missing commandId, a duplicate event sequence, a duplicate event id, an event that refers to a command not in the backup, a missing payload, and a payload that no longer deserializes
- and: when there are any errors, nothing is written
- and: when there are no errors, commands and then events are inserted exactly as they are in the file, and ids that already exist are skipped and counted as skipped

_Source: `src/main/java/dev/ted/jittertravel/application/BackupService.java`._

## S6: The backup source label depends on the hosting-environment marker @v1 [proposed]

- Given: the environment marker is "production"
- When: the backup source label is resolved
- Then: the label is "production"
- and: when the marker is absent or blank, the label is "local"

_Source: `src/test/java/dev/ted/jittertravel/application/BackupSourceTest.java`, `src/main/java/dev/ted/jittertravel/application/BackupSource.java`._

## S7: A flight change overwrites the booked flight's row @v1 [proposed]

- Given: flight UA59 on United from SFO to FRA is booked
- When: the flight is changed to Lufthansa LH441 from SFO to MUC
- Then: the booked flights list has one row
- and: that row shows airline Lufthansa, flight number LH441 and route "SFO→MUC"

_Source: `src/test/java/dev/ted/jittertravel/application/BookedFlightsProjectorFlightChangedTest.java`, `src/main/java/dev/ted/jittertravel/application/BookedFlightsProjector.java`, `src/main/java/dev/ted/jittertravel/application/BookedFlightView.java`._

## S8: Flight history lists the booking first, then each change in order @v1 [proposed]

- Given: a flight is booked on 2026-05-20 12:22 UTC
- and: it is changed on 2026-06-01 9:05 with the reason "Wanted to fly home earlier"
- and: it is changed again on 2026-06-15 16:30 with no reason
- When: the booked flights list is viewed
- Then: the history reads "Booked on 2026-05-20 12:22PM", then "Wanted to fly home earlier (changed on 2026-06-01 9:05AM)", then "Changed on 2026-06-15 4:30PM"
- and: the row has changes, and its latest change reads "Changed on 2026-06-15 4:30PM"
- and: a flight that was only booked has no changes and one history entry starting "Booked on "

_Source: `src/test/java/dev/ted/jittertravel/application/BookedFlightsProjectorFlightChangedTest.java`, `src/main/java/dev/ted/jittertravel/application/BookedFlightsProjector.java`, `src/main/java/dev/ted/jittertravel/application/BookedFlightView.java`._

## S9: Booked itineraries are found while live, and their flights stay linked to them @v1 [proposed]

- Given: United Airlines itinerary MD7LKB is booked with an outbound and a return flight
- When: the itinerary is looked up
- Then: it is found with its airline, confirmation code and flights, and it is not cancelled
- and: an itinerary that was never booked is not found
- and: once the itinerary is cancelled, it is no longer found as live
- and: cancelling a different itinerary leaves this one live
- and: after this itinerary is cancelled, its return flight is still found as belonging to MD7LKB
- and: a flight that no itinerary names belongs to no itinerary

_Source: `src/test/java/dev/ted/jittertravel/application/BookedItinerariesProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/BookedItinerariesProjector.java`, `src/main/java/dev/ted/jittertravel/application/BookedItineraryView.java`._

## S10: A schedule change or a rebooking replaces the itinerary's flights @v1 [proposed]

- Given: an itinerary is booked with an outbound and a return flight
- When: a schedule change or a rebooking lists the outbound, the return and an added flight
- Then: the live itinerary's flights are exactly those three
- and: the added flight is found as belonging to the itinerary's confirmation code

_Source: `src/test/java/dev/ted/jittertravel/application/BookedItinerariesProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/BookedItinerariesProjector.java`._

## S11: The ?cancelled parameter shows or hides cancelled rows @v1 [proposed]

- Given: the `?cancelled` request parameter is "show", "SHOW" or "Show"
- When: it is resolved
- Then: it resolves to SHOW
- and: "hide" resolves to HIDE
- and: an absent, empty or unrecognized value such as "banana" or "shown" resolves to HIDE
- and: with HIDE, cancelled flights and hotel stays are left out of the list, though the list still counts the cancelled rows the time filter admits

_Source: `src/test/java/dev/ted/jittertravel/application/CancelledViewTest.java`, `src/main/java/dev/ted/jittertravel/application/CancelledView.java`, `src/main/java/dev/ted/jittertravel/application/BookedFlightsProjector.java`, `src/main/java/dev/ted/jittertravel/application/BookedHotelsProjector.java`._

## S12: Itinerary evaluations default a missing confirmation code to empty and are not bookable without a command @v1 [proposed]

- Given: an itinerary evaluation is made with no confirmation code
- When: its confirmation code is read
- Then: it is the empty string, never null
- and: a given confirmation code such as "MD7LKB" is kept
- and: an evaluation that could not be parsed, for example with the message "No flights found", is not bookable

_Source: `src/test/java/dev/ted/jittertravel/application/ItineraryEvaluationTest.java`._
