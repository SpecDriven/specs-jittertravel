# Booking flight itineraries from a pasted confirmation email, and gathering and ground-transfer views

_Extracted from `src/main/java/dev/ted/jittertravel/application`._

A whole flight itinerary is booked from a pasted airline confirmation email. Previewing and booking use the same evaluation: the paste is parsed, each airport gets a time zone, and the booking rules are checked without writing anything. All legs and the itinerary are recorded together under one command, or nothing is recorded. Repeat submissions, confirmation codes that are already booked and cancelled codes are refused with a clear reason. This part also covers the gathering details view used by the edit form, and how ground-transfer endpoints are offered.

## S1: A clean paste books every leg and the itinerary under one command @v1 [proposed]

- Given: a pasted confirmation email with code MD7LKB and two legs, UA2091 SFO→ORD and UA3509 ORD→YOW, that overlap no booked flight
- When: the itinerary is booked
- Then: the evaluation is bookable
- and: three events are recorded, a FlightBooked for each leg followed by FlightItineraryBooked naming both flight ids and the confirmation code MD7LKB
- and: exactly one command is logged, its id is the itinerary id, and every event is attributed to it
- and: both flight numbers appear on the booked flights list

_Source: `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingIntegrationTest.java`, `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingTest.java`, `src/main/java/dev/ted/jittertravel/application/FlightItineraryBooking.java`._

## S2: Leg times are zoned by the airport where they happen @v1 [proposed]

- Given: a paste whose second leg arrives at YOW at 05:16 PM on Oct 18, 2026
- When: the itinerary is booked
- Then: that leg's arrival is recorded as 17:16 in America/Toronto, the zone the curated airport table gives for YOW

_Source: `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingTest.java`._

## S3: An airport the table does not know waits for a zone and books nothing @v1 [proposed]

- Given: a paste with an airport the curated table does not know, such as YQB
- When: the itinerary is booked without a zone picked for that airport
- Then: the evaluation lists YQB as an unresolved airport and is not bookable
- and: no event is recorded

_Source: `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingTest.java`, `src/main/java/dev/ted/jittertravel/application/FlightItineraryBooking.java`._

## S4: A picked zone settles an unknown airport @v1 [proposed]

- Given: a paste with the unknown airport YQB
- When: it is evaluated with the zone CANADA_EASTERN picked for YQB
- Then: the evaluation is bookable
- and: the leg arriving at YQB has its arrival in America/Toronto

_Source: `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingTest.java`._

## S5: A leg that overlaps a booked flight refuses the whole paste and writes nothing @v1 [proposed]

- Given: a flight is already booked that overlaps the first leg's time window
- When: the itinerary is booked
- Then: the evaluation is not bookable
- and: the first leg shows an overlapping-leg refusal and the second leg shows no refusal
- and: no command is logged and no event is recorded, not even a partial booking

_Source: `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingTest.java`, `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingIntegrationTest.java`, `src/main/java/dev/ted/jittertravel/application/FlightItineraryBooking.java`._

## S6: A paste that cannot be read books nothing @v1 [proposed]

- Given: pasted text that is not an itinerary, such as "Thanks for choosing United!"
- When: the itinerary is booked
- Then: the evaluation reports parse problems
- and: no event is recorded

_Source: `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingTest.java`._

## S7: Submitting the same form twice books it once and reports the second as already booked @v1 [proposed]

- Given: an itinerary id has already been booked, and its flights may since have been cancelled
- When: the same form is submitted again with that itinerary id
- Then: the evaluation reports the itinerary as already booked and not bookable, rather than as overlaps with itself
- and: no command is logged and no event is recorded
- and: a booking under a different itinerary id does not count as already booked

_Source: `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingTest.java`, `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingIntegrationTest.java`, `src/main/java/dev/ted/jittertravel/application/FlightItineraryBooking.java`._

## S8: A confirmation code already on the books is refused rather than applied as a change @v1 [proposed]

- Given: a live itinerary with code MD7LKB is booked
- When: a new paste for MD7LKB with a changed time is booked
- Then: the evaluation reports the code as on the books, giving the booked itinerary, its leg count of 2, and its first departure and last arrival
- and: it is refused before any unknown airport's zone is asked for
- and: nothing is recorded

_Source: `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingTest.java`, `src/main/java/dev/ted/jittertravel/application/FlightItineraryBooking.java`._

## S9: A code whose itinerary was cancelled is refused with the reason @v1 [proposed]

- Given: the itinerary with code MD7LKB was booked and then cancelled
- When: a paste for MD7LKB is booked
- Then: the evaluation reports the problem "Itinerary MD7LKB was canceled; it cannot be changed"
- and: nothing is recorded

_Source: `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingTest.java`, `src/main/java/dev/ted/jittertravel/application/FlightItineraryBooking.java`._

## S10: The stored command carries none of the email's private lines @v1 [proposed]

- Given: a paste that contains an eTicket number, a frequent flyer number, traveler details and seats
- When: the itinerary is booked
- Then: the stored command includes the confirmation code MD7LKB
- and: it contains none of the eTicket number, the frequent flyer number, the traveler name or the seat

_Source: `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingTest.java`._

## S11: Cancelling an itinerary cancels every leg and the itinerary under one command, and only once @v1 [proposed]

- Given: a booked two-leg itinerary
- When: it is cancelled with the reason "Trip called off"
- Then: two leg cancellations and a FlightItineraryCancelled event are recorded under one command
- and: both legs leave the live flights list and the itinerary is no longer live
- and: cancelling it a second time is refused as itinerary not found and records nothing

_Source: `src/test/java/dev/ted/jittertravel/application/FlightItineraryBookingIntegrationTest.java`._

## S12: The gathering details view follows the latest planned or changed snapshot @v1 [proposed]

- Given: a gathering is planned with a title, venue, location, start and end, speaking flag and info URL
- When: the details for that gathering are looked up
- Then: every one of those values is returned as it was planned
- and: after a GatheringChanged event, every field is overwritten with the changed values and other gatherings stay unchanged
- and: an unknown gathering id finds nothing

_Source: `src/test/java/dev/ted/jittertravel/application/GatheringDetailsViewProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/GatheringDetailsViewProjector.java`._
