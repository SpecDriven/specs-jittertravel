# Booking, changing and cancelling travel: flights, itineraries, transfers and addresses

_Extracted from `src/main/java/dev/ted/jittertravel/domain`._

This part covers the domain rules for booking flights and multi-leg flight itineraries, for cancelling flights and ground transfers, and for the values these use. Addresses are normalized, airport codes are checked, and each airport resolves to a time zone. A booking is refused if the arrival comes before the departure, if the departure is not in the future, or if it overlaps a leg already booked. A cancellation is refused only when there is nothing live to cancel.

## S1: Booking a future flight records FlightBooked @v1 [proposed]

- Given: no legs are booked and the departure is after now
- When: a flight is booked from SFO to JFK with airline "United" and flight number "UA100"
- Then: a single FlightBooked event is emitted
- and: it carries the airline, flight number, both airports and both date/times exactly as given

_Source: `src/test/java/dev/ted/jittertravel/domain/BookFlightCommandTest.java`, `src/main/java/dev/ted/jittertravel/domain/BookFlightCommand.java`._

[test: executeProducesFlightBookedEvent : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookFlightCommandTest.java#L20 ]

## S2: A flight arriving before it departs is refused as the command is built @v1 [proposed]

- Given: an arrival date/time that is not after the departure date/time
- When: a flight booking is built
- Then: it is refused with InvalidDateRange, "Arrival date/time must be after departure date/time"
- and: no command reaches the command log

_Source: `src/test/java/dev/ted/jittertravel/domain/BookFlightCommandTest.java`, `src/main/java/dev/ted/jittertravel/domain/BookFlightCommand.java`._

[test: arrivalBeforeDepartureIsRefusedAsTheCommandIsBuilt : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookFlightCommandTest.java#L60 ]

## S3: A flight departing in the past is refused @v1 [proposed]

- Given: the departure is not after now
- When: the flight booking is executed
- Then: it is refused with DepartureNotInFuture, "Departure date/time must be in the future"

_Source: `src/test/java/dev/ted/jittertravel/domain/BookFlightCommandTest.java`, `src/main/java/dev/ted/jittertravel/domain/BookFlightCommand.java`._

[test: departureInPastThrowsDepartureNotInFuture : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookFlightCommandTest.java#L42 ]

## S4: A flight overlapping an already booked leg is refused and the refusal names the blocking leg @v1 [proposed]

- Given: a train leg is booked from 09:00 to 11:00
- When: a flight departing at 10:00 is booked
- Then: it is refused with OverlappingLegRefused
- and: the refusal identifies the booked train leg as the leg that blocks it

_Source: `src/test/java/dev/ted/jittertravel/domain/BookFlightCommandTest.java`, `src/main/java/dev/ted/jittertravel/domain/BookFlightCommand.java`._

[test: aFlightOverlappingAnAlreadyBookedLegIsRefused : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookFlightCommandTest.java#L77 ]

## S5: Booking an itinerary books every leg and records which flights belong together @v1 [proposed]

- Given: an itinerary with airline "United Airlines", confirmation code "MD7LKB" and two valid future legs
- When: the itinerary is booked
- Then: one FlightBooked is emitted for each leg, in leg order
- and: these are followed by a FlightItineraryBooked listing the legs' flight ids in order
- and: a connection where one leg lands at the same time the next departs is not a conflict

_Source: `src/test/java/dev/ted/jittertravel/domain/BookFlightItineraryCommandTest.java`, `src/main/java/dev/ted/jittertravel/domain/BookFlightItineraryCommand.java`._

[test: everyLegIsBookedAndTheItineraryRecordsWhichFlightsBelongTogether : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookFlightItineraryCommandTest.java#L25 ]
[test: aConnectionIsNotAConflict : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookFlightItineraryCommandTest.java#L42 ]

## S6: Overlapping legs of the same itinerary are both refused, each naming the other @v1 [proposed]

- Given: leg 1 runs from 08:00 to 11:00 and leg 2 from 10:00 to 12:00
- When: the itinerary is booked
- Then: it is refused with FlightItineraryRefused
- and: leg 1 is refused as overlapping leg 2, and leg 2 as overlapping leg 1

_Source: `src/test/java/dev/ted/jittertravel/domain/BookFlightItineraryCommandTest.java`, `src/main/java/dev/ted/jittertravel/domain/BookFlightItineraryCommand.java`._

[test: twoLegsOfTheSameItineraryThatOverlapAreBothRefusedNamingEachOther : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookFlightItineraryCommandTest.java#L53 ]

## S7: An itinerary leg that collides with a flight already booked is refused @v1 [proposed]

- Given: a flight is already booked from 09:00 to 12:00
- When: an itinerary with a leg from 08:00 to 11:00 is booked
- Then: FlightItineraryRefused holds one refusal for that leg, with the reason OverlappingLegRefused

_Source: `src/test/java/dev/ted/jittertravel/domain/BookFlightItineraryCommandTest.java`, `src/main/java/dev/ted/jittertravel/domain/BookFlightItineraryCommand.java`._

[test: aLegCollidingWithAFlightAlreadyBookedIsRefusedWithTheBlockingLeg : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookFlightItineraryCommandTest.java#L68 ]

## S8: Every itinerary refusal is reported together, each against its own leg @v1 [proposed]

- Given: leg 1 has already departed, leg 2 is valid, and leg 3 arrives before it departs
- When: the itinerary is booked
- Then: one FlightItineraryRefused holds two refusals: leg 1 with DepartureNotInFuture and leg 3 with InvalidDateRange
- and: a leg whose own window is invalid is not also checked for overlaps
- and: building an itinerary with no legs is refused, because an itinerary has at least one leg

_Source: `src/test/java/dev/ted/jittertravel/domain/BookFlightItineraryCommandTest.java`, `src/main/java/dev/ted/jittertravel/domain/BookFlightItineraryCommand.java`._

[test: everyRefusalIsReportedTogetherEachAgainstItsOwnLeg : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookFlightItineraryCommandTest.java#L82 ]
[test: anItineraryHasAtLeastOneLeg : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookFlightItineraryCommandTest.java#L99 ]

## S9: Cancelling a live flight records FlightCancelled, and an unknown flight is refused @v1 [proposed]

- Given: a live flight
- When: it is cancelled with reason "Rebooked on UA58" and a cancellation time
- Then: FlightCancelled is emitted with that reason and that time
- and: an absent reason becomes the empty string
- and: cancelling a flight that does not exist, or was already cancelled, is refused with FlightNotFound, and the message names the flight id

_Source: `src/test/java/dev/ted/jittertravel/domain/CancelFlightCommandTest.java`, `src/main/java/dev/ted/jittertravel/domain/CancelFlightCommand.java`._

[test: cancellingALiveFlightEmitsFlightCancelledWithWhenItWasCancelled : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/CancelFlightCommandTest.java#L17 ]
[test: cancellingAFlightThatDoesNotExistIsRefused : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/CancelFlightCommandTest.java#L25 ]
[test: anAbsentReasonBecomesEmptyRatherThanNull : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/CancelFlightCommandTest.java#L34 ]

## S10: Ground transfers can be cancelled at any time, but only if they exist @v1 [proposed]

- Given: a live ground transfer, even one whose time has already passed
- When: it is cancelled
- Then: GroundTransferCancelled is emitted for that transfer
- and: cancelling a transfer that does not exist, or was already cancelled, is refused with GroundTransferNotFound

_Source: `src/test/java/dev/ted/jittertravel/domain/CancelGroundTransferCommandTest.java`, `src/main/java/dev/ted/jittertravel/domain/CancelGroundTransferCommand.java`._

[test: emitsCancelledEventForTheTransferBeingCancelled : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/CancelGroundTransferCommandTest.java#L13 ]
[test: unknownTransferIsRejected : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/CancelGroundTransferCommandTest.java#L25 ]
[test: aTransferThatHasAlreadyHappenedIsStillCancellable : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/CancelGroundTransferCommandTest.java#L33 ]

## S11: Address fields are trimmed, never null, and the matching location falls back to the city @v1 [proposed]

- Given: an address typed with stray spaces, such as "Hamburg ", or with fields left out
- When: the address is created
- Then: every field has its surrounding whitespace removed, and an absent field becomes the blank string
- and: a matching location that is absent or holds only spaces becomes the trimmed city
- and: a matching location that differs from the city, such as "Johannesberg" for a venue in "Rückersbach", is kept
- and: a city typed with a trailing space matches the same city typed without one

_Source: `src/test/java/dev/ted/jittertravel/domain/AddressTest.java`, `src/main/java/dev/ted/jittertravel/domain/Address.java`._

[test: surroundingWhitespaceIsRemovedFromEveryField : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/AddressTest.java#L19 ]
[test: aCityTypedWithATrailingSpaceIsTheSamePlaceAsOneWithout : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/AddressTest.java#L29 ]
[test: everyAbsentFieldBecomesTheBlankString : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/AddressTest.java#L39 ]
[test: absentMatchingLocationFallsBackToTheTrimmedCity : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/AddressTest.java#L47 ]
[test: whitespaceOnlyMatchingLocationFallsBackToTheCity : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/AddressTest.java#L60 ]
[test: aMatchingLocationThatDiffersFromTheCityIsKept : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/AddressTest.java#L68 ]

## S12: Airport codes are three letters, stored in upper case, and resolve to a known time zone @v1 [proposed]

- Given: an airport code
- When: it is created
- Then: lowercase or mixed case is turned into upper case, so "sfo" becomes "SFO"
- and: a missing code, a code that is not exactly 3 characters, or a code containing anything but letters is refused with InvalidAirportCode
- and: a known airport resolves to its zone: FRA to Europe/Berlin, SFO to America/Los_Angeles, JFK to America/New_York, YOW to America/Toronto
- and: a code missing from the airport table is refused with ZoneResolutionException; there is no default zone

_Source: `src/test/java/dev/ted/jittertravel/domain/AirportCodeTest.java`, `src/test/java/dev/ted/jittertravel/domain/AirportZoneResolverTest.java`, `src/main/java/dev/ted/jittertravel/domain/AirportCode.java`, `src/main/java/dev/ted/jittertravel/domain/AirportZoneResolver.java`._

[test: lowercaseIsNormalizedToUppercase : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/AirportCodeTest.java#L11 ]
[test: mixedCaseIsNormalizedToUppercase : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/AirportCodeTest.java#L16 ]
[test: nullCodeIsRejected : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/AirportCodeTest.java#L21 ]
[test: wrongLengthIsRejected : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/AirportCodeTest.java#L27 ]
[test: nonLettersAreRejected : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/AirportCodeTest.java#L35 ]
[test: resolvesKnownAirportToItsZone : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/AirportZoneResolverTest.java#L15 ]
