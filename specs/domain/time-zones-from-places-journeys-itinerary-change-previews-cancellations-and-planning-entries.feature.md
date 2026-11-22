# Time zones from places, journeys, itinerary change previews, cancellations and planning entries

_Extracted from `src/main/java/dev/ted/jittertravel/domain`._

This part covers how a place's time zone is worked out from its city, state or province, and country, and how itinerary legs are grouped into journeys. It also covers how a pasted itinerary change is compared with what is booked and which legs it refuses, how hotel stays and train trips are cancelled, and the rules for planning conferences and private events, recording CFPs, speaking invitations and rejected talks.

## S1: A city in the curated table sets the zone, ahead of its state or country @v1 [proposed]

- Given: an address with city "Phoenix", region "AZ" and country "US"
- When: its time zone is resolved
- Then: the zone is America/Phoenix, taken from the city table before the state
- and: "  NEW YORK " in US resolves to America/New_York, ignoring case and surrounding whitespace
- and: a single-zone country such as DE resolves from its ISO code alone, so Soltau, DE is Europe/Berlin

_Source: `src/main/java/dev/ted/jittertravel/domain/LocationZoneResolver.java`, `src/test/java/dev/ted/jittertravel/domain/LocationZoneResolverTest.java`._

## S2: A town missing from the city table resolves from its state code, read only within its own country @v1 [proposed]

- Given: an address in a town no table lists, with region "WA"
- When: the country is US
- Then: the zone is America/Los_Angeles
- and: when the country is AU, the same region gives Australia/Perth
- and: Centennial, CO, US resolves to America/Denver and North Kawartha, ON, CA resolves to America/Toronto

_Source: `src/main/java/dev/ted/jittertravel/domain/LocationZoneResolver.java`, `src/test/java/dev/ted/jittertravel/domain/LocationZoneResolverTest.java`._

## S3: An unknown place fails instead of falling back to a default zone @v1 [proposed]

- Given: an address whose city, region and country match no table, such as Nowheresville, "XX", US
- When: its time zone is resolved
- Then: a zone resolution error is raised, because a multi-zone country has no default zone
- and: the error message contains "region='XX'"
- and: a missing address, a country stored as a name such as "Germany", and a state stored as a name such as "Colorado" also fail to resolve

_Source: `src/main/java/dev/ted/jittertravel/domain/LocationZoneResolver.java`, `src/test/java/dev/ted/jittertravel/domain/LocationZoneResolverTest.java`._

## S4: In a single-zone country the region is ignored @v1 [proposed]

- Given: an address "Steventon" with region "Abingdon" and country "GB"
- When: its time zone is resolved
- Then: the zone is Europe/London
- and: a train station given only as a city and country resolves the same way, so Frankfurt, DE is Europe/Berlin

_Source: `src/main/java/dev/ted/jittertravel/domain/LocationZoneResolver.java`, `src/test/java/dev/ted/jittertravel/domain/LocationZoneResolverTest.java`._

## S5: More than 24 hours on the ground ends a journey @v1 [proposed]

- Given: legs SFO→ORD and ORD→YOW on 18 October, then YOW→ORD and ORD→SFO on 31 October
- When: the legs are split into journeys
- Then: there are two journeys, SFO→YOW and YOW→SFO
- and: a gap of exactly 24 hours between landing and the next departure is still a connection, but 24 hours and one minute starts a new journey
- and: landing at 21:00 and leaving at 07:30 the next day is still a connection

_Source: `src/main/java/dev/ted/jittertravel/domain/Journey.java`, `src/test/java/dev/ted/jittertravel/domain/JourneyTest.java`._

## S6: Journeys are built in departure order and run from their first leg to their last @v1 [proposed]

- Given: two connecting legs supplied in reverse order
- When: they are split into journeys
- Then: they form one journey from the first leg's departure airport to the last leg's arrival airport
- and: the journey departs at the first leg's departure time and arrives at the last leg's arrival time
- and: no legs give no journeys

_Source: `src/main/java/dev/ted/jittertravel/domain/Journey.java`, `src/test/java/dev/ted/jittertravel/domain/JourneyTest.java`._

## S7: A pasted itinerary change is matched to booked legs and listed as unchanged, changed, added or removed @v1 [proposed]

- Given: a booked itinerary and a pasted set of legs
- When: the change is planned
- Then: each pasted leg is paired with a booked leg first by flight number and departure day, then by route, and the earliest unpaired booked leg wins a tie
- and: a pasted leg with no partner is added, and a booked leg with no partner is removed
- and: a booked leg that has already departed and is missing from the paste counts as unchanged, not removed
- and: the rows are listed in departure order, and every refusal is collected so all of them are reported together

_Source: `src/main/java/dev/ted/jittertravel/domain/ItineraryChangePlan.java`._

## S8: An itinerary change refuses legs that rewrite history, return cancelled legs, or collide @v1 [proposed]

- Given: a pasted change to an itinerary
- When: a leg that has already departed is pasted with different times
- Then: that leg is refused
- and: a pasted leg that matches only a leg Ted cancelled earlier is refused with "Canceled earlier; it cannot be added back here"
- and: a changed or added leg must depart in the future and arrive after it departs
- and: it must not overlap any other booked leg outside this itinerary
- and: it must not overlap another leg of the same paste, which is refused with "Overlaps leg N of this itinerary"

_Source: `src/main/java/dev/ted/jittertravel/domain/ItineraryChangePlan.java`, `src/main/java/dev/ted/jittertravel/domain/LegCancelledEarlier.java`, `src/main/java/dev/ted/jittertravel/domain/OverlapsAnotherItineraryLeg.java`, `src/main/java/dev/ted/jittertravel/domain/OverlappingLegRefused.java`._

## S9: An existing hotel booking can be cancelled at any time, with its reason kept @v1 [proposed]

- Given: a hotel booking that exists
- When: it is cancelled with the reason "Trip called off"
- Then: a hotel booking cancelled event carrying that reason is emitted
- and: no check-in date or cancel-by deadline is checked
- and: a missing reason is recorded as an empty string
- and: cancelling a booking that does not exist is refused as hotel booking not found

_Source: `src/test/java/dev/ted/jittertravel/domain/CancelHotelCommandTest.java`._

## S10: Cancelling a train trip that does not exist is refused @v1 [proposed]

- Given: a train trip id with no trip behind it
- When: the trip is cancelled
- Then: it is refused as train not found, and the message names the trip id
- and: cancelling an existing trip emits a train cancelled event with the reason, or with an empty string when no reason is given

_Source: `src/test/java/dev/ted/jittertravel/domain/CancelTrainCommandTest.java`._

## S11: Planning a conference or private event checks its dates @v1 [proposed]

- Given: a conference whose end date is before its start date
- When: it is planned
- Then: it is refused with "End date must be on or after start date"
- and: a conference that has already ended, judged by calendar day in the venue's zone, is refused with "Conference has already ended", but one on its last day is accepted
- and: a private event must end after it starts and must start today or later in its own zone, or it is refused with "Private event date must be today or later"

_Source: `src/main/java/dev/ted/jittertravel/domain/PlanConferenceCommand.java`, `src/main/java/dev/ted/jittertravel/domain/PlanPrivateEventCommand.java`, `src/main/java/dev/ted/jittertravel/domain/PrivateEventDateNotInFuture.java`._

## S12: Recording a CFP, a speaking invitation or a rejected talk needs a conference, and some need a submitted talk @v1 [proposed]

- Given: a conference
- When: a CFP is recorded for it
- Then: it is refused if the conference does not exist or is open-space, and accepted even after its closing date has passed
- and: a speaking invitation is refused only when the conference does not exist, whatever its format, and it does not commit attendance
- and: a talk rejection is refused when no talk was submitted, including when Ted was only invited to speak

_Source: `src/main/java/dev/ted/jittertravel/domain/OpenCfpCommand.java`, `src/main/java/dev/ted/jittertravel/domain/InviteToSpeakCommand.java`, `src/main/java/dev/ted/jittertravel/domain/RejectTalkCommand.java`, `src/main/java/dev/ted/jittertravel/domain/NoTalkToDecide.java`._
