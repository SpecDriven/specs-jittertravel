# Booking hotels and trains, schedule collisions, airport and region lookups, and talk submissions

_Extracted from `src/main/java/dev/ted/jittertravel/domain`._

This part covers booking a hotel stay and a train trip. Mistakes in what was typed are refused as the command is built. Rules that depend on the current time or on other bookings are refused when the command runs. A proposed journey is checked against the legs already booked, and a connection does not count as a collision. It also covers looking up the city and address for an airport code and the states, provinces and territories of the countries that pick a region from a list. Station entries are cleaned of stray whitespace. Talk submissions and withdrawals are refused when they would record something untrue.

## S1: A valid hotel booking records a HotelBooked event @v1 [proposed]

- Given: a hotel named "Grand Hotel" with a complete address, a check-in two weeks from now and a check-out on the next day
- When: the booking is executed with a TENTATIVE or FINAL booking intent
- Then: one HotelBooked event is produced, carrying the hotel booking id, the hotel name, the address, the check-in, the check-out and the booking intent
- and: a cancel-by deadline is carried onto the event when one was given, and stays absent when none was given

_Source: `src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java`._

[test: validTentativeCommandProducesHotelBookedEventWithAllFields : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L25 ]
[test: validFinalCommandProducesHotelBookedEventWithFinalIntent : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L48 ]
[test: cancelByIsCarriedOntoTheEventWhenItPrecedesCheckIn : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L104 ]
[test: absentCancelByStaysNullOnTheEvent : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L117 ]

## S2: A hotel check-in must be strictly in the future @v1 [proposed]

- Given: a hotel booking whose check-in is an hour ago, or exactly now
- When: the booking is executed
- Then: it is refused with CheckInNotInFuture

_Source: `src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java`._

[test: checkInInPastThrowsCheckInNotInFuture : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L58 ]
[test: checkInExactlyNowIsNotAcceptedMustBeStrictlyAfter : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L68 ]

## S3: The hotel's dates and cancel-by deadline are checked as the booking is built @v1 [proposed]

- Given: a check-out on the same calendar day as the check-in
- When: the hotel booking is built
- Then: it is refused with InvalidHotelDateRange
- and: a check-out on the next day is accepted
- and: a cancel-by deadline one minute after check-in is refused with InvalidCancelByDate
- and: a cancel-by deadline exactly at check-in is accepted

_Source: `src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java`._

[test: checkOutOnSameDayAsCheckInIsRefusedAsTheCommandIsBuilt : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L84 ]
[test: checkOutExactlyOneDayAfterCheckInIsValid : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L92 ]
[test: cancelByAfterCheckInIsRefusedAsTheCommandIsBuilt : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L137 ]
[test: cancelByExactlyAtCheckInIsAccepted : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L126 ]

## S4: Every location problem with a hotel stay is reported in one answer @v1 [proposed]

- Given: a hotel stay with no name and no city
- When: the hotel booking is built
- Then: it is refused with InvalidEnteredLocation for the STAY role, listing "Name is required" against VENUE_NAME and then "City is required" against CITY
- and: a US address with no state is refused against REGION with "State required for United States"
- and: a hotel name pasted into the city is refused against CITY
- and: a location problem is reported ahead of a bad date range or a past check-in

_Source: `src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java`._

[test: aStayWithNoNameAndNoCityReportsBothInOneAnswer : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L181 ]
[test: aUsStayWithNoStateIsRejectedAgainstTheRegionField : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L145 ]
[test: hotelNamePastedIntoTheCityIsRejectedAgainstTheCityField : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L165 ]
[test: hotelWithNoNameIsRejectedAgainstTheNameField : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L156 ]
[test: locationIsCheckedBeforeTheDates : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L194 ]
[test: aDataEntryMistakeIsRefusedBeforeThePastCheckInCouldBe : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookHotelCommandTest.java#L210 ]

## S5: A valid train booking records TrainBooked, and its departure must be strictly in the future @v1 [proposed]

- Given: a trip from London Euston to Manchester Piccadilly departing next week, with service "DB - ICE 610", and nothing else booked
- When: the booking is executed
- Then: one TrainBooked event is produced with the trip id, both stations, both times and the service id
- and: an arrival on the same day as the departure is accepted
- and: a departure an hour ago, or exactly now, is refused with DepartureNotInFuture
- and: an arrival before the departure is refused with InvalidDateRange as the booking is built

_Source: `src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java`, `src/main/java/dev/ted/jittertravel/domain/TrainBooked.java`._

[test: validCommandProducesTrainBookedEventWithAllFields : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L34 ]
[test: arrivalSameDayAsDepartureIsValid : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L82 ]
[test: departureInPastThrowsDepartureNotInFuture : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L57 ]
[test: departureExactlyNowIsNotAcceptedMustBeStrictlyAfter : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L66 ]
[test: arrivalBeforeDepartureIsRefusedAsTheCommandIsBuilt : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L75 ]

## S6: Station problems at both ends of a train trip are reported together @v1 [proposed]

- Given: a departure station with no name and an arrival station whose name was pasted into the city
- When: the train booking is built
- Then: it is refused with InvalidTrainEntry, listing DEPARTURE with VENUE_NAME and then ARRIVAL with CITY
- and: one station with neither a name nor a city adds two problems to the list, one for VENUE_NAME and one for CITY
- and: a station problem is reported ahead of an inverted time range or a past departure

_Source: `src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java`, `src/main/java/dev/ted/jittertravel/domain/TrainStations.java`._

[test: bothEndsAtFaultAreReportedTogether : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L125 ]
[test: oneEndWithNoNameAndNoCityContributesBothProblems : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L151 ]
[test: locationIsCheckedBeforeTheTimes : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L165 ]
[test: aDataEntryMistakeIsRefusedBeforeThePastDepartureCouldBe : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L180 ]
[test: stationWithNoNameIsRefusedForThatEndAsTheCommandIsBuilt : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L92 ]
[test: stationNamePastedIntoTheCityIsRefusedForThatEndAsTheCommandIsBuilt : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L108 ]

## S7: A train trip that overlaps a booked leg is refused, but a connection is accepted @v1 [proposed]

- Given: a booked train leg from 09:00 to 11:00
- When: a train trip from 10:00 to 12:00 is booked
- Then: it is refused with OverlappingLegRefused, which names the blocking leg
- and: a trip departing at 11:00, when the booked leg arrives, is accepted
- and: the time range is checked before the overlap, so an inverted time range is refused as InvalidDateRange

_Source: `src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java`, `src/test/java/dev/ted/jittertravel/domain/ScheduledLegsTest.java`, `src/main/java/dev/ted/jittertravel/domain/ScheduledLeg.java`._

[test: aTripOverlappingAnAlreadyBookedLegIsRefused : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L200 ]
[test: aTripConnectingOutOfABookedLegIsAccepted : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L213 ]
[test: theDateRulesAreCheckedBeforeTheOverlap : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/BookTrainCommandTest.java#L227 ]
[test: aJourneyOverlappingABookedLegIsReported : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/ScheduledLegsTest.java#L22 ]
[test: aConnectionIsNotACollision : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/ScheduledLegsTest.java#L30 ]

## S8: Collision checks skip the leg being changed and report only the first collision @v1 [proposed]

- Given: a schedule of booked legs
- When: a proposed window is checked against it
- Then: a leg being changed does not collide with itself, but it still collides with a different leg it overlaps
- and: a flight and a train collide, whatever their kinds
- and: when several legs overlap, only the first is reported
- and: windows entirely before or after a leg, or an empty schedule, give no collision

_Source: `src/test/java/dev/ted/jittertravel/domain/ScheduledLegsTest.java`, `src/main/java/dev/ted/jittertravel/domain/ScheduledLegs.java`._

[test: aLegDoesNotCollideWithItself : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/ScheduledLegsTest.java#L49 ]
[test: changingALegStillCollidesWithADifferentOne : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/ScheduledLegsTest.java#L59 ]
[test: aFlightCollidesWithATrainBecauseTheKindIsNotTheQuestion : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/ScheduledLegsTest.java#L70 ]
[test: theFirstCollidingLegIsReported : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/ScheduledLegsTest.java#L78 ]
[test: aJourneyEntirelyBeforeOrAfterIsNotACollision : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/ScheduledLegsTest.java#L39 ]
[test: anEmptyScheduleCollidesWithNothing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/ScheduledLegsTest.java#L92 ]

## S9: Airport codes resolve to their city and address, and a city resolves back only to its one airport @v1 [proposed]

- Given: the table of known airports
- When: a code or a city is looked up
- Then: "DEN" gives Denver, and an unknown code such as "ZZZ" is returned unchanged
- and: the address for "den" is city Denver, region CO, country US, and "FRA" has no region with country DE
- and: an unknown code has no address
- and: Frankfurt gives FRA and "  denver " gives DEN
- and: London, New York, Paris, Chicago, Tokyo and Washington DC have several airports and give nothing, as do an unknown city and a missing city

_Source: `src/test/java/dev/ted/jittertravel/domain/StaticAirportCityResolverTest.java`, `src/main/java/dev/ted/jittertravel/domain/StaticAirportCityResolver.java`._

[test: aKnownCodeResolvesToItsCity : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/StaticAirportCityResolverTest.java#L16 ]
[test: anUnknownCodeIsReturnedAsIsRatherThanGuessed : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/StaticAirportCityResolverTest.java#L29 ]
[test: anAirportsAddressIsItsCityStateAndCountryCode : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/StaticAirportCityResolverTest.java#L75 ]
[test: anAirportOutsideTheStateListCountriesHasNoRegion : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/StaticAirportCityResolverTest.java#L82 ]
[test: anUnknownAirportHasNoAddress : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/StaticAirportCityResolverTest.java#L89 ]
[test: aCityWithExactlyOneAirportResolvesBackToIt : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/StaticAirportCityResolverTest.java#L34 ]

## S10: The United States, Canada and Australia pick their region from a list @v1 [proposed]

- Given: the subdivision lists
- When: a country's region rules are asked for
- Then: US, CA, AU and " us " require a region, with 51, 13 and 8 subdivisions
- and: DE, GB, "United States" and a blank country do not require one and have no list
- and: a code is found only in its own country, ignoring case, so WA is Washington in the US and Western Australia in AU
- and: a name such as " georgia " is found only in its own country, and a code given as a name, or a name given as a code, is not found

_Source: `src/test/java/dev/ted/jittertravel/domain/SubdivisionsTest.java`, `src/main/java/dev/ted/jittertravel/domain/Subdivisions.java`._

[test: theUnitedStatesCanadaAndAustraliaRequireARegion : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/SubdivisionsTest.java#L15 ]
[test: everywhereElseTheRegionIsFreeText : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/SubdivisionsTest.java#L23 ]
[test: eachListIsComplete : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/SubdivisionsTest.java#L33 ]
[test: aCodeIsFoundInItsOwnCountry : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/SubdivisionsTest.java#L46 ]
[test: aNameIsFoundInItsOwnCountryOnly : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/SubdivisionsTest.java#L59 ]
[test: aCodeFromAnotherCountryOrANameIsNotFound : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/SubdivisionsTest.java#L72 ]

## S11: Train station entries are cleaned of surrounding whitespace @v1 [proposed]

- Given: a station entered with leading or trailing spaces, or with fields left empty
- When: the station is recorded
- Then: every field is trimmed, and every empty field becomes the blank string
- and: "Hamburg " and "Hamburg" count as the same city

_Source: `src/test/java/dev/ted/jittertravel/domain/TrainStationAddressTest.java`, `src/main/java/dev/ted/jittertravel/domain/TrainStationAddress.java`._

[test: surroundingWhitespaceIsRemovedFromEveryField : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/TrainStationAddressTest.java#L16 ]
[test: everyAbsentFieldBecomesTheBlankString : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/TrainStationAddressTest.java#L37 ]
[test: aCityTypedWithATrailingSpaceIsTheSamePlaceAsOneWithout : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/TrainStationAddressTest.java#L27 ]

## S12: Submitting or withdrawing a talk is refused when it would record something untrue @v1 [proposed]

- Given: a conference and where its talk stands
- When: a talk is submitted
- Then: it is refused with ConferenceNotFound when no live conference exists, with ConferenceHasNoCfp for an OPEN_SPACE conference, and with TalkAlreadyAccepted when a talk is already accepted
- and: otherwise a TalkSubmitted event is produced, including for a repeat submission
- When: a talk is withdrawn
- Then: a TalkWithdrawn event is produced only when the talk is SUBMITTED or ACCEPTED, and NoTalkToWithdraw is refused in every other state

_Source: `src/main/java/dev/ted/jittertravel/domain/SubmitTalkCommand.java`, `src/main/java/dev/ted/jittertravel/domain/WithdrawTalkCommand.java`, `src/main/java/dev/ted/jittertravel/domain/TalkPipelineContext.java`._

[test: emitsTalkSubmitted : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/TalkPipelineCommandTest.java#L38 ]
[test: unknownConferenceIsRejected : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/TalkPipelineCommandTest.java#L51 ]
[test: anOpenSpaceConferenceIsRejected : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/TalkPipelineCommandTest.java#L60 ]
[test: submittingAfterAnAcceptanceIsRejected : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/TalkPipelineCommandTest.java#L77 ]
[test: submittingAgainIsAllowedFromEveryOtherState : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/TalkPipelineCommandTest.java#L86 ]
[test: withdrawingWithNothingOutstandingIsRejected : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/domain/TalkPipelineCommandTest.java#L252 ]
