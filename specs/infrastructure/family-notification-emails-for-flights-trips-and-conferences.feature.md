# Family notification emails for flights, trips and conferences

_Extracted from `src/main/java/dev/ted/jittertravel/infrastructure`._

Each newly appended batch of events is read as a whole to decide what family should be told: a single flight, one whole trip, a trip change, a cancellation, or a conference whose attendance commitment may have changed. Each notification is then rendered as a plain-text email using the approved wording. Each send is tried on its own: a failure is counted on "family.notification.failed" and does not stop the batch's other notifications.

## S1: A flight booked on its own is told as a flight @v1 [proposed]

- Given: a batch holding only one FlightBooked
- When: the batch is reacted to
- Then: family is notified once with the flight as subject and the fact FLIGHT_BOOKED
- and: the notification carries that one leg and the current time

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuator.java`, `src/test/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuatorTest.java`._

## S2: A pasted itinerary is told once as a trip, with its legs in departure order @v1 [proposed]

- Given: a batch holding three FlightBooked legs and a FlightItineraryBooked that names them in any order
- When: the batch is reacted to
- Then: family is notified exactly once, with the itinerary as subject and the fact ITINERARY_BOOKED
- and: the legs are ordered by departure time
- and: no leg is sent as a flight of its own

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuator.java`, `src/test/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuatorTest.java`._

## S3: An itinerary that names a flight missing from the batch fails on its own @v1 [proposed]

- Given: a batch where a FlightItineraryBooked names a flight that is not booked in the same batch, and another flight is booked on its own
- When: the batch is reacted to
- Then: no trip notification is sent for that itinerary
- and: the counter "family.notification.failed" goes up by one
- and: the other flight is still told as FLIGHT_BOOKED

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuator.java`, `src/test/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuatorTest.java`._

## S4: A schedule change or rebooking is told as an itinerary change @v1 [proposed]

- Given: a batch holding a FlightItineraryChanged or a FlightItineraryRebooked together with a newly booked leg of that trip
- When: the batch is reacted to
- Then: family is notified of a changed itinerary, with the command id of the change's append
- and: the new leg is not sent as a flight of its own
- and: a flight cancelled in the same batch is not told as a lone cancellation

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuator.java`, `src/test/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuatorTest.java`._

## S5: Cancelling one flight on its own is told as a cancelled flight @v1 [proposed]

- Given: a batch holding a FlightCancelled and no itinerary event
- When: the batch is reacted to
- Then: family is notified of that cancelled flight, with the command id of the append that cancelled it
- and: no other notification is sent

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuator.java`, `src/test/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuatorTest.java`._

## S6: Cancelling a whole itinerary is told once with the legs cancelled alongside it @v1 [proposed]

- Given: a batch holding two FlightCancelled events and a FlightItineraryCancelled
- When: the batch is reacted to
- Then: family is notified of the cancelled itinerary exactly once, with the set of flights cancelled in the same batch
- and: no lone flight cancellation is sent

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuator.java`, `src/test/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuatorTest.java`._

## S7: A failed send is counted and the batch's other notifications still go @v1 [proposed]

- Given: the notification service refuses a change, a cancellation or a flight notification (for example "Brevo returned 502")
- When: the batch is reacted to
- Then: the counter "family.notification.failed" goes up by one for each refusal
- and: the error does not escape the reactor
- and: the batch's other notifications are still sent

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuator.java`, `src/test/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuatorTest.java`._

## S8: Some events send nothing, and only attendance-changing conference events notify @v1 [proposed]

- Given: a batch holding only a FlightChanged for one flight
- When: the batch is reacted to
- Then: family is not notified
- Given: a batch holding ConferenceAttendanceConfirmed, ConferenceAttendanceDeclined, ConferenceCancelled, TalkAccepted or TalkRejected
- When: the batch is reacted to
- Then: family is notified about each such conference once, however many of these events arrived for it
- and: a submitted or withdrawn talk, an invitation to speak, planning a conference or changing its dates notifies nothing

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuator.java`, `src/test/java/dev/ted/jittertravel/infrastructure/FamilyNotificationActuatorTest.java`._

## S9: A single-flight email names the route, the cities and each time's airport @v1 [proposed]

- Given: a booked flight UA2091 from SFO at 6:10 AM to ORD at 12:45 PM on 18 Oct 2026
- When: the FLIGHT_BOOKED email is built
- Then: the subject is "(JitterTravel) Ted booked a new flight: SFO → ORD"
- and: the body gives the calendar link, the flight number, "San Francisco (SFO) → Chicago (ORD)", "Departs  Sun 18 Oct 2026, 6:10 AM (SFO)" and "Arrives  Sun 18 Oct 2026, 12:45 PM (ORD)"
- and: a FLIGHT_CANCELLED email has the subject "(JitterTravel) Ted canceled a flight: SFO → ORD", has no calendar link, and uses "Was due to depart" and "Was due to arrive"
- and: an airport missing from the city table is shown by its code, as in "ZZZ (ZZZ)"

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/FamilyNotificationMessages.java`, `src/test/java/dev/ted/jittertravel/infrastructure/FamilyNotificationMessagesTest.java`._

## S10: A trip email chains the airports and numbers each leg @v1 [proposed]

- Given: a trip SFO→ORD, ORD→YOW, YOW→ORD, ORD→SFO
- When: the ITINERARY_BOOKED email is built
- Then: the subject is "(JitterTravel) Ted booked a new trip: SFO → ORD → YOW → ORD → SFO"
- and: the body says how many flights there are and lists each leg numbered, with legs separated by a blank line
- and: a leg that does not start where the previous one ended starts a new run after a comma, as in "SFO → ORD, FRA → HAM"
- and: an ITINERARY_CANCELLED email says "Ted canceled a trip" with the same airport chain, and lists each leg as "Was due to depart" and "Was due to arrive"

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/FamilyNotificationMessages.java`, `src/test/java/dev/ted/jittertravel/infrastructure/FamilyNotificationMessagesTest.java`._

## S11: Trip emails never name the confirmation code, the airline or a reason @v1 [proposed]

- Given: a trip booked with United Airlines under confirmation code MD7LKB
- When: its booked or cancelled email is built
- Then: neither the subject nor the body contains the confirmation code, the airline, the word "Confirmation", a cancellation reason or "schedule change"

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/FamilyNotificationMessages.java`, `src/test/java/dev/ted/jittertravel/infrastructure/FamilyNotificationMessagesTest.java`._

## S12: The calendar link opens the calendar at the first departure's local day @v1 [proposed]

- Given: a base URL is configured
- When: the link for a flight or trip is built
- Then: it is "<base>/calendar?day=<date>", where the date is the local date of the earliest departure, whatever order the legs are given in
- and: a flight leaving at 11:30 PM on the 18th in San Francisco links to the 18th, not to the UTC date
- and: a base URL ending in a slash still gives a single slash
- and: with a blank base URL there is no link at all
- and: the conference "going" email links to the conference's first day and gives the name, the venue, city and country, dates such as "Mon 24 Aug – Thu 27 Aug 2026", the speaking line and the info URL

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/FamilyNotificationMessages.java`, `src/test/java/dev/ted/jittertravel/infrastructure/FamilyNotificationMessagesTest.java`._
