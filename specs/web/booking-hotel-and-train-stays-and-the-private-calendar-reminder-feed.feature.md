# Booking hotel and train stays, and the private calendar reminder feed

_Extracted from `src/main/java/dev/ted/jittertravel/web`._

The book-hotel and book-train forms fill in sensible starting values and accept prefill parameters from the calendar day menu and from schedule-problem fix links. When the service refuses a booking, the error is shown on the field that fixes it. A booking that goes through redirects back to where it was started. A private iCalendar feed, gated by a token, serves deadline reminders, a weekly heartbeat and a probe event. Subscribe links are built from the base URL and the token. Pages for family and anonymous viewers ship a browser-zone script; owner pages do not.

## S1: Hotel form defaults to a stay two weeks out @v1 [proposed]

- Given: the current date is 2026-05-31
- When: the owner opens /book-hotel with no parameters
- Then: check-in is 2026-06-14 at 15:00
- and: check-out is 2026-06-15 at 11:00
- and: the form carries a newly minted hotel booking id that is different on every request

_Source: `src/main/java/dev/ted/jittertravel/web/BookHotelController.java`, `src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java`._

[test: getBookHotelFormSetsCheckInTwoWeeksFromNowAtThreePmAndCheckOutNextMorningAtElevenAm : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java#L45 ]
[test: getBookHotelFormAssignsDistinctHotelBookingIdOnEachRequest : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java#L116 ]

## S2: Hotel form opens on the stay a fix link or the calendar day asks for @v1 [proposed]

- Given: a "Book hotel" fix link with city Johannesberg, country DE, checkIn 2026-09-10 and checkOut 2026-09-14
- When: /book-hotel is opened with those parameters
- Then: city and country are prefilled
- and: check-in is 2026-09-10 at 15:00 and check-out is 2026-09-14 at 11:00
- and: a ?date= from the calendar day menu instead seeds check-in on that day at 15:00 with check-out the next morning at 11:00
- and: when checkIn and date are both given, checkIn wins

_Source: `src/main/java/dev/ted/jittertravel/web/BookHotelController.java`, `src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java`._

[test: getBookHotelFormWithAFixLinksCityCountryAndNightsOpensOnThatStay : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java#L75 ]
[test: getBookHotelFormWithDateSeedsCheckInOnThatDayAtThreePmAndCheckOutNextMorning : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java#L57 ]

## S3: Bad or blank hotel prefill values fall back @v1 [proposed]

- Given: a checkOut that is not after the check-in day, or a city and country that are blank
- When: /book-hotel is opened
- Then: check-out falls back to one night after check-in, at 11:00
- and: the blank city and country are left empty rather than filled in

_Source: `src/main/java/dev/ted/jittertravel/web/BookHotelController.java`, `src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java`._

[test: aCheckOutThatIsNotAfterCheckInFallsBackToOneNight : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java#L92 ]
[test: aBlankCityAndCountryAreLeftAloneRatherThanWrittenIn : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java#L104 ]

## S4: Each hotel booking refusal lands on the input that fixes it @v1 [proposed]

- Given: a submitted hotel form that the booking service refuses
- When: the form is posted to /book-hotel
- Then: the book-hotel form is shown again
- and: "check-in not in the future" is reported on checkIn, an invalid date range on checkOut, and an invalid cancel-by date on cancelBy
- and: every location problem is reported in the same response on its own field (hotelName, city, country, region)
- and: a time zone that cannot be resolved is reported on zone with "Could not determine the time zone from the location — please choose one."

_Source: `src/main/java/dev/ted/jittertravel/web/BookHotelController.java`, `src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java`._

[test: eachRefusalLandsOnTheInputsThatFixIt : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java#L138 ]
[test: anUnresolvableZoneErrorRendersUnderTheZonePicker : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelWebIntegrationTest.java#L217 ]
[test: aBlankNameAndABlankCityAreBothReportedInOneResponse : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelWebIntegrationTest.java#L344 ]
[test: aCancelByAfterCheckInErrorRendersUnderCancelBy : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelWebIntegrationTest.java#L197 ]
[test: postWithCheckInInPastRendersFormAgain : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelWebIntegrationTest.java#L80 ]

## S5: A refused booking form comes back under a fresh id @v1 [proposed]

- Given: a hotel or train booking was refused
- When: the form is shown again
- Then: it carries a new booking id, different from the refused one
- and: everything else that was typed is kept
- and: the errors still show under their inputs

_Source: `src/main/java/dev/ted/jittertravel/web/BookHotelController.java`, `src/main/java/dev/ted/jittertravel/web/BookTrainController.java`, `src/main/java/dev/ted/jittertravel/web/BookHotelRequest.java`, `src/main/java/dev/ted/jittertravel/web/BookTrainRequest.java`, `src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java`._

[test: aRefusedFormIsReRenderedUnderAFreshId : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java#L192 ]
[test: aRefusedFormIsReRenderedUnderAFreshId : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainControllerTest.java#L193 ]
[test: aRefusedFormComesBackUnderAFreshId : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelWebIntegrationTest.java#L108 ]
[test: aRefusedFormComesBackUnderAFreshId : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainWebIntegrationTest.java#L124 ]

## S6: A successful booking returns to where it started @v1 [proposed]

- Given: a valid hotel booking submitted with from=calendar and day=2026-12-03
- When: the booking service accepts it
- Then: the browser is redirected to /calendar?day=2026-12-03
- and: without from and day it is redirected to /booked-hotels, or to /booked-trains for a train

_Source: `src/main/java/dev/ted/jittertravel/web/BookHotelController.java`, `src/main/java/dev/ted/jittertravel/web/BookTrainController.java`, `src/test/java/dev/ted/jittertravel/web/BookHotelControllerTest.java`._

[test: postValidHotelRedirectsToBookedHotels : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookHotelWebIntegrationTest.java#L62 ]
[test: postValidTrainRedirectsToBookedTrains : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainWebIntegrationTest.java#L77 ]

## S7: Train form defaults and prefill @v1 [proposed]

- Given: no ?date= parameter
- When: /book-train is opened
- Then: departure is at 09:00 one week from today and arrival is four hours later
- and: a ?date= puts departure on that day at 09:00
- and: fromCity, fromCountry, toCity and toCountry from a fix link prefill the departure and arrival fields, and blank values are ignored

_Source: `src/main/java/dev/ted/jittertravel/web/BookTrainController.java`._

[test: getBookTrainFormSetsDepartureOneWeekFromNowAtNineAm : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainControllerTest.java#L48 ]
[test: getBookTrainFormWithDateSeedsDepartureOnThatDayAtNineAm : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainControllerTest.java#L60 ]
[test: getBookTrainFormSetsArrivalSameDayAsDeparture : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainControllerTest.java#L74 ]
[test: aFixLinksCitiesAndCountriesSeedTheStationFields : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainControllerTest.java#L107 ]
[test: blankCitiesAndCountriesAreLeftAloneRatherThanWrittenIn : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainControllerTest.java#L122 ]
[test: getBookTrainFormAssignsDistinctTripIdOnEachRequest : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainControllerTest.java#L86 ]

## S8: Train booking refusals are reported on the relevant fields @v1 [proposed]

- Given: a submitted train form that the booking service refuses
- When: it is posted to /book-train
- Then: a departure that is not in the future is reported on departureDateTime
- and: an invalid date range is reported on arrivalDateTime
- and: a refused overlapping leg is reported on departureDateTime, together with a notice about the leg already booked
- and: a form whose dates were blank or did not parse is shown again without calling the service

_Source: `src/main/java/dev/ted/jittertravel/web/BookTrainController.java`._

[test: eachRefusalLandsOnTheInputsThatFixIt : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainControllerTest.java#L143 ]
[test: aBlankDepartureRerendersTheFormWithRequiredUnderItAndBooksNothing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainWebIntegrationTest.java#L153 ]
[test: anOverlappingTripIsRefusedAndNamesTheBlockingLegAsALink : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainWebIntegrationTest.java#L441 ]
[test: postWithPastDepartureRendersFormAgain : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainWebIntegrationTest.java#L96 ]
[test: postWithArrivalBeforeDepartureRendersFormAgain : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/BookTrainWebIntegrationTest.java#L182 ]

## S9: Calendar feed is gated by its token @v1 [proposed]

- Given: a calendar feed token is configured
- When: /calendar/feed/{token}.ics or /calendar/feed/{token}/probe.ics is requested with the matching token
- Then: a text/calendar body is returned
- and: a wrong or missing token, or no configured token at all, gets a 404 with an empty body

_Source: `src/main/java/dev/ted/jittertravel/web/CalendarFeedController.java`._

[test: correctTokenReturns200TextCalendarWithTheVevent : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedControllerTest.java#L54 ]
[test: wrongTokenReturns404WithEmptyBody : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedControllerTest.java#L69 ]
[test: probeWithCorrectTokenReturns200WithASingleSyntheticVevent : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedControllerTest.java#L76 ]
[test: probeWithWrongTokenReturns404 : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedControllerTest.java#L90 ]
[test: feedRequestWithAnyTokenReturns404WhenDisabled : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedDisabledTest.java#L48 ]
[test: probeRequestWithAnyTokenReturns404WhenDisabled : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedDisabledTest.java#L55 ]

## S10: Feed holds every source's deadlines plus a weekly heartbeat @v1 [proposed]

- Given: hotel cancel-deadline events and CFP deadline events exist
- When: the feed is assembled
- Then: the events from both sources are included
- and: there is always a heartbeat event whose uid starts with "heartbeat-" and ends with "@jittertravel", with a -PT5M alarm
- and: the heartbeat starts strictly in the future, a whole number of weeks after Monday 2024-01-01 17:00 UTC
- and: with no deadlines at all, the feed holds only the heartbeat

_Source: `src/main/java/dev/ted/jittertravel/web/CalendarFeedAssembler.java`, `src/test/java/dev/ted/jittertravel/web/CalendarFeedAssemblerTest.java`._

[test: everySourcesEventsReachTheFeed : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedAssemblerTest.java#L36 ]
[test: feedAlwaysContainsTheLivenessHeartbeatEvenWithNothingToRemind : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedAssemblerTest.java#L46 ]
[test: heartbeatIsStrictlyInTheFutureAndAlignedToTheWeeklyCadence : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedAssemblerTest.java#L59 ]

## S11: Probe event is ten minutes out @v1 [proposed]

- Given: the current instant
- When: the probe feed is requested
- Then: it holds one event with uid "probe@jittertravel" that starts ten minutes from now
- and: the event has a -PT5M alarm

_Source: `src/main/java/dev/ted/jittertravel/web/CalendarFeedAssembler.java`, `src/test/java/dev/ted/jittertravel/web/CalendarFeedAssemblerTest.java`._

[test: probeEventIsTenMinutesOutWithAShortAlarmAndAStableUid : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedAssemblerTest.java#L70 ]

## S12: Feed links use webcal for subscribing and https for adding the probe @v1 [proposed]

- Given: base URL https://travel.example.com/ and token TOKEN123
- When: the feed links are built
- Then: the subscribe URL is webcal://travel.example.com/calendar/feed/TOKEN123.ics, with no doubled slash
- and: the probe add URL is https://travel.example.com/calendar/feed/TOKEN123/probe.ics
- and: the probe subscribe URL is webcal://travel.example.com/calendar/feed/TOKEN123/probe.ics
- and: an http base URL is also swapped to webcal for subscribing

_Source: `src/main/java/dev/ted/jittertravel/web/CalendarFeedLinks.java`, `src/test/java/dev/ted/jittertravel/web/CalendarFeedLinksTest.java`._

[test: subscribeUrlUsesWebcalSchemeSoItOpensTheIosSubscribeSheet : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedLinksTest.java#L10 ]
[test: probeAddUrlStaysHttpsForOneOffAddAllImport : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedLinksTest.java#L18 ]
[test: probeSubscribeUrlUsesWebcalSchemeToTestTheSubscriptionPath : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedLinksTest.java#L26 ]
[test: trailingSlashOnTheBaseUrlIsNotDoubledIntoThePath : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedLinksTest.java#L34 ]
[test: httpBaseUrlAlsoSwapsToWebcalForSubscription : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CalendarFeedLinksTest.java#L42 ]
