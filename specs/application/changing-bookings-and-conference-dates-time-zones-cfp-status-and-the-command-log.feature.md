# Changing bookings and conference dates, time zones, CFP status and the command log

_Extracted from `src/main/java/dev/ted/jittertravel/application`._

This part covers how flight, train, hotel, gathering, conference and private-event changes become recorded commands. A flight endpoint takes its time zone from its own airport code. A gathering or train takes its zone from the place given on the form. A zone picked explicitly always takes precedence. Moved conference dates keep the zone the conference was planned in. Every write runs through a command log that is saved before any events are stored, records why a command failed, and refuses all writes in read-only mode. This part also covers whether a CFP is open, how places are written as short city labels, and the CFP checks run on the plan-conference form before anything is written.

## S1: Each flight endpoint resolves its zone from its own airport code @v1 [proposed]

- Given: a flight request from SFO departing 2026-09-15 15:55 to FRA arriving 2026-09-16 11:45, with no zones picked
- When: the flight is booked
- Then: the departure is stamped in America/Los_Angeles and the arrival in Europe/Berlin
- and: the departure instant is 2026-09-15T22:55Z and the arrival instant is 2026-09-16T09:45Z

_Source: `src/test/java/dev/ted/jittertravel/application/BookFlightHandlerTest.java`._

## S2: An explicit zone pick wins over the airport code and may be a raw IANA zone @v1 [proposed]

- Given: a flight request from SFO to FRA
- When: the departure zone is picked as US_CENTRAL
- Then: the departure is stamped in America/Chicago and the arrival stays in Europe/Berlin
- and: when the arrival zone is picked as "Asia/Tokyo", the arrival is stamped in Asia/Tokyo

_Source: `src/test/java/dev/ted/jittertravel/application/BookFlightHandlerTest.java`._

## S3: Malformed and unknown airport codes are refused @v1 [proposed]

- Given: a flight request whose departure airport is "BADCODE"
- When: the flight is booked or changed
- Then: it is refused as an invalid airport code before any zone is resolved
- and: an unknown code such as "ZZZ" with no zone picked is refused as a zone resolution failure
- and: "ZZZ" with the zone US_EASTERN picked is accepted and stamped in America/New_York

_Source: `src/test/java/dev/ted/jittertravel/application/BookFlightHandlerTest.java`, `src/test/java/dev/ted/jittertravel/application/ChangeFlightHandlerTest.java`, `src/main/java/dev/ted/jittertravel/application/ChangeFlightHandler.java`._

## S4: A flight change reason is trimmed, and a missing reason becomes empty @v1 [proposed]

- Given: a change-flight request
- When: the reason is null or only spaces
- Then: the recorded reason is empty
- and: a reason of "  Schedule shifted by airline  " is recorded as "Schedule shifted by airline"

_Source: `src/test/java/dev/ted/jittertravel/application/ChangeFlightHandlerTest.java`, `src/main/java/dev/ted/jittertravel/application/ChangeFlightHandler.java`._

## S5: Changing a gathering works out its zone again from the submitted location @v1 [proposed]

- Given: an existing gathering and a change form with its venue in Lisbon, PT, starting at 18:00 on 2026-09-15
- When: the gathering is changed
- Then: its start is stamped in Europe/Lisbon and keeps the submitted wall-clock time 2026-09-15T18:00
- and: picking the zone UK stamps it in Europe/London instead
- and: a location that cannot be resolved, such as Springfield, Freedonia, with no zone picked is refused

_Source: `src/test/java/dev/ted/jittertravel/application/ChangeGatheringHandlerTest.java`, `src/main/java/dev/ted/jittertravel/application/ChangeGatheringHandler.java`._

## S6: Moved conference dates are stamped in the zone the conference was planned in @v1 [proposed]

- Given: a conference planned in America/New_York
- When: its dates are changed to 2027-03-29 09:00 through 2027-03-31 17:00
- Then: one ConferenceDatesChanged event is recorded with both dates in America/New_York

_Source: `src/test/java/dev/ted/jittertravel/application/ChangeConferenceDatesTest.java`, `src/main/java/dev/ted/jittertravel/application/ChangeConferenceDates.java`._

## S7: A date change for an unknown, declined or cancelled conference is refused as not found @v1 [proposed]

- Given: a conference that was never planned, was planned and then declined or cancelled, or is a different conference from the one that was planned
- When: its dates are changed
- Then: the change is refused as conference not found

_Source: `src/test/java/dev/ted/jittertravel/application/ChangeConferenceDatesTest.java`, `src/main/java/dev/ted/jittertravel/application/ChangeConferenceDates.java`._

## S8: The command is logged as pending before its events are stored, and failures are marked @v1 [proposed]

- Given: a command to run
- When: it runs successfully
- Then: the command, not the form request, is saved as pending first and its events are then stored under the command id
- and: when the command rejects, it is marked FAILED_DOMAIN with the error message and no events are stored
- and: when storing the events fails, it is marked FAILED_PERSIST with the error message and the error is passed on

_Source: `src/test/java/dev/ted/jittertravel/application/CommandExecutorTest.java`, `src/main/java/dev/ted/jittertravel/application/CommandExecutor.java`._

## S9: An external action runs only after its command is logged, and a failed send stores nothing @v1 [proposed]

- Given: work that reaches an outside system, such as sending mail
- When: it is executed
- Then: the command is saved as pending before the action runs, and the action's events are stored afterwards
- and: if the action throws, for example "Brevo returned 502", the command is marked FAILED_SEND and no events are stored
- and: if storing fails after a successful action, the command is marked FAILED_PERSIST

_Source: `src/test/java/dev/ted/jittertravel/application/CommandExecutorTest.java`, `src/main/java/dev/ted/jittertravel/application/CommandExecutor.java`._

## S10: Nothing is written in read-only mode @v1 [proposed]

- Given: the event store is in read-only mode
- When: a command, a direct event append or an external action is attempted
- Then: it is refused with a read-only mode error
- and: no command row is saved, no events are stored, and an external action does not run

_Source: `src/test/java/dev/ted/jittertravel/application/CommandExecutorTest.java`, `src/main/java/dev/ted/jittertravel/application/CommandExecutor.java`._

## S11: A CFP is open only while a recorded deadline has not passed @v1 [proposed]

- Given: a conference with a CFP deadline of 2026-09-12T21:59Z
- When: it is checked one second before the deadline
- Then: the CFP is open
- and: at the deadline instant itself and afterwards it is closed
- and: a conference with no recorded deadline is not open

_Source: `src/test/java/dev/ted/jittertravel/application/CfpDeadlineViewTest.java`, `src/main/java/dev/ted/jittertravel/application/CfpDeadlineView.java`._

## S12: A place reads as "City, state" in the US and "City, Country" everywhere else @v1 [proposed]

- Given: an address with a city, region and ISO country code
- When: it is labelled
- Then: Denver, CO, US reads "Denver, CO" and Atlanta with no state in US reads "Atlanta, USA"
- and: Toronto, ON, CA reads "Toronto, Canada" and London, Westminster Borough, GB reads "London, United Kingdom", so regions outside the US are not shown
- and: with no country the city stands alone, and with no city the country name stands alone
- and: country codes are matched case-insensitively ("gb" reads "United Kingdom"), and an unknown code such as "XX" is shown as stored

_Source: `src/test/java/dev/ted/jittertravel/application/CityLabelTest.java`, `src/main/java/dev/ted/jittertravel/application/CityLabel.java`._
