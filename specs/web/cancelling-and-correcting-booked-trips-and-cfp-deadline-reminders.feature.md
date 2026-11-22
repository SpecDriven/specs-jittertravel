# Cancelling and correcting booked trips, and CFP deadline reminders

_Extracted from `src/main/java/dev/ted/jittertravel/web`._

The owner gets dedicated confirmation pages for cancelling booked flights, flight itineraries, trains, hotels, ground transfers and private events. Each page returns to where its link was clicked, and a stale or malformed link goes back to a list without saving anything. There are also forms for changing a booked flight, a flight itinerary or a conference's dates. Separately, the private calendar feed includes a reminder event, with three alarms, for every upcoming call-for-papers (CFP) deadline.

## S1: The flight cancel confirmation page identifies the flight @v1 [proposed]

- Given: a booked United Airlines flight UA2091 from SFO to ORD, departing Sun, Oct 18, 2026, 6:10 AM
- When: the owner opens /booked-flights/{id}/cancel
- Then: the page shows SFO, ORD, United Airlines, UA2091 and the departure time
- and: its form posts back to /booked-flights/{id}/cancel
- and: the submit button reads "Cancel this flight" in red, and no typed confirmation word is asked for

_Source: `src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/CancelFlightController.java`._

[test: confirmationPageIdentifiesTheFlight : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L52 ]
[test: confirmationPagePostsBackToItsOwnCancelPath : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L68 ]
[test: theConfirmationIsRedAndTakesNoTypedWord : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L166 ]

## S2: Confirming cancels the flight and returns to the list @v1 [proposed]

- Given: a booked flight
- When: the owner posts to /booked-flights/{id}/cancel with the reason "Rebooked on UA58"
- Then: the flight is cancelled with that reason at the current time
- and: the owner is redirected to /booked-flights
- and: if no reason is given, the cancellation records an empty reason rather than none

_Source: `src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/CancelFlightController.java`, `src/main/java/dev/ted/jittertravel/web/CancelFlightRequest.java`._

[test: confirmingCancelsTheFlightAndReturnsToTheList : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L80 ]
[test: anOmittedReasonIsRecordedAsEmptyRatherThanNull : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L98 ]

## S3: Stale or malformed cancel links go to the list without writing anything @v1 [proposed]

- Given: a flight id that is already cancelled, or a path id that is not a UUID
- When: the owner opens or posts the cancel page
- Then: the owner is redirected to /booked-flights
- and: no cancellation is recorded
- and: any from origin is ignored, because nothing was fixed

_Source: `src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/CancelFlightController.java`._

[test: aStaleLinkForAnAlreadyCancelledFlightNavigatesToTheListWritingNothing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L111 ]
[test: aMalformedIdNavigatesToTheListRatherThanFailing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L127 ]
[test: aStaleLinkIgnoresTheOriginBecauseNothingWasFixed : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L210 ]

## S4: A flight cancelled in another tab, or a write in read-only mode, is handled without an error page @v1 [proposed]

- Given: the flight is found, but by the time of the write it has already been cancelled
- When: the owner confirms the cancel
- Then: the owner is redirected to /booked-flights
- and: if the app is in read-only mode, the owner is redirected to /read-only instead

_Source: `src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/CancelFlightController.java`._

[test: cancelledInAnotherTabBetweenLookupAndWriteIsNotAnError : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L136 ]
[test: readOnlyModeSendsTheReaderToThePageThatExplainsIt : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L149 ]

## S5: A cancel started from the problem report returns to that report @v1 [proposed]

- Given: the cancel page was opened with from=problems-list
- When: the page renders
- Then: the form carries from=problems-list as a hidden input
- and: a successful cancel redirects to /schedule-problems?view=list
- and: a hand-edited from such as https://evil.example.com redirects to /schedule-problems?view=calendar, never off the app
- and: an ordinary visit renders no from input

_Source: `src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/CancelFlightController.java`._

[test: theConfirmationFormCarriesTheOriginThroughThePost : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L222 ]
[test: cancellingFromTheProblemListReturnsToTheProblemList : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L184 ]
[test: aHandEditedOriginCannotRedirectOffTheApp : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L197 ]
[test: anOrdinaryVisitRendersNoOriginInput : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L272 ]

## S6: A cancel started from an itinerary day returns to that day @v1 [proposed]

- Given: the cancel page was opened with from=itinerary and day=2026-07-03
- When: the page renders
- Then: the keep link reads "Keep it — back to the itinerary" and points to /itinerary?date=2026-07-03
- and: a successful cancel redirects to /itinerary?date=2026-07-03
- and: with no origin, the keep link reads "Keep it — back to the list" and points to /booked-flights

_Source: `src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/CancelFlightController.java`._

[test: keepItFromAnItineraryBinLeadsBackToThatDay : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L248 ]
[test: cancellingFromAnItineraryBinReturnsToTheDayItWasOn : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L234 ]
[test: keepItWithNoOriginLeadsToTheList : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightControllerTest.java#L260 ]

## S7: Other cancel pages fall back to their own list or to the itinerary @v1 [proposed]

- Given: a cancel link for a train, hotel, private event or ground transfer that is stale or malformed, or whose item was already cancelled
- When: the owner opens or posts it
- Then: train misses go to /booked-trains and hotel misses go to /booked-hotels
- and: private event and ground transfer misses go to /itinerary
- and: when there is no origin, a successful ground transfer cancel returns to /itinerary?date= the day the transfer departs, and asks for no reason

_Source: `src/main/java/dev/ted/jittertravel/web/CancelTrainController.java`, `src/main/java/dev/ted/jittertravel/web/CancelHotelController.java`, `src/main/java/dev/ted/jittertravel/web/CancelPrivateEventController.java`, `src/main/java/dev/ted/jittertravel/web/CancelGroundTransferController.java`, `src/main/java/dev/ted/jittertravel/web/CancelGroundTransferRequest.java`._

[test: aStaleLinkForAnAlreadyCancelledTripNavigatesToTheListWritingNothing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelTrainControllerTest.java#L118 ]
[test: aMalformedIdNavigatesToTheListRatherThanFailing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelTrainControllerTest.java#L134 ]
[test: getOnUnknownBookingRedirectsToList : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelHotelControllerTest.java#L69 ]
[test: malformedBookingIdRedirectsInsteadOfThrowing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelHotelControllerTest.java#L162 ]
[test: getOnAnAlreadyCancelledEventRedirectsToTheItinerary : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelPrivateEventControllerTest.java#L129 ]
[test: cancellingLandsBackOnTheItineraryDayTheTransferLeft : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelGroundTransferControllerTest.java#L109 ]

## S8: Cancelling an itinerary with a departed leg is refused on the page @v1 [proposed]

- Given: a live flight itinerary, one of whose legs has already departed
- When: the owner posts to /booked-itineraries/{id}/cancel
- Then: the confirmation page is shown again, marked as departed and keeping the typed reason
- and: the page lists the legs not yet cancelled in departure order, plus the stays that would be left booked
- and: an unknown, malformed or already-cancelled itinerary redirects to /booked-flights

_Source: `src/main/java/dev/ted/jittertravel/web/CancelFlightItineraryController.java`, `src/main/java/dev/ted/jittertravel/web/CancelFlightItineraryRequest.java`._

[test: aDepartedLegIsAnsweredOnThePageWithALinkToCancelEachRemainingFlight : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightItineraryControllerTest.java#L278 ]
[test: aDepartedRefusalKeepsTheWayBackItArrivedWith : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightItineraryControllerTest.java#L262 ]
[test: legsAreListedInDepartureOrderWhateverOrderTheItineraryNamesThem : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightItineraryControllerTest.java#L84 ]
[test: aStaleLinkForAnItineraryThatIsNotLiveNavigatesToTheListWritingNothing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightItineraryControllerTest.java#L203 ]
[test: aMalformedIdNavigatesToTheListRatherThanFailing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CancelFlightItineraryControllerTest.java#L218 ]

## S9: A future CFP deadline becomes a calendar event with 72h, 24h and 4h alarms @v1 [proposed]

- Given: a conference named J-Fall whose CFP closes 30 days from now
- When: the private feed collects its events
- Then: one event starts at the deadline, with uid "{conferenceId}-cfp@jittertravel"
- and: its summary is "CFP closes: J-Fall"
- and: its alarm triggers are -PT72H, -PT24H and -PT4H
- and: its description reads "Ede, Netherlands — conference runs 2026-11-05 to 2026-11-06"

_Source: `src/test/java/dev/ted/jittertravel/web/CfpDeadlineSourceTest.java`, `src/main/java/dev/ted/jittertravel/web/CfpDeadlineSource.java`._

[test: aFutureCfpDeadlineBecomesOneVeventCarrying72h24hAnd4hAlarms : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CfpDeadlineSourceTest.java#L36 ]
[test: theDescriptionSaysWhereAndWhenTheConferenceItselfIs : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CfpDeadlineSourceTest.java#L81 ]

## S10: Only open CFPs of conferences that have not been dropped produce reminders @v1 [proposed]

- Given: conferences of any date, past or future, excluding the ones Ted has dropped
- When: the feed collects CFP events
- Then: a conference with no recorded CFP produces nothing
- and: a CFP that has already closed produces nothing
- and: a CFP still open produces an event even when the conference itself is months away, because only the deadline is filtered on

_Source: `src/test/java/dev/ted/jittertravel/web/CfpDeadlineSourceTest.java`, `src/main/java/dev/ted/jittertravel/web/CfpDeadlineSource.java`._

[test: aConferenceWithNoRecordedCfpProducesNothing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CfpDeadlineSourceTest.java#L52 ]
[test: aCfpThatHasAlreadyClosedIsExcluded : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CfpDeadlineSourceTest.java#L59 ]
[test: theDeadlineIsWhatIsFilteredOnNotWhetherTheConferenceHasHappened : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/CfpDeadlineSourceTest.java#L73 ]

## S11: Changing a conference's dates is guarded and reports problems on the form @v1 [proposed]

- Given: the change-dates form at /conferences/{id}/dates
- When: the owner opens or submits it
- Then: in read-only mode the owner is redirected to /read-only before the form is shown
- and: an unknown id goes to /conferences, and a conference Ted declined (NOT_GOING) goes to its detail page
- and: the form is prefilled with the dates currently in force
- and: an end date before the start date is rejected on the end date field
- and: a conference canceled or declined meanwhile shows "Not saved: this conference was canceled or declined"
- and: success redirects to the conference detail page

_Source: `src/main/java/dev/ted/jittertravel/web/ChangeConferenceDatesController.java`, `src/main/java/dev/ted/jittertravel/web/ChangeConferenceDatesRequest.java`._

[test: getWhileReadOnlyRedirectsBeforeTheFormIsShown : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/ChangeConferenceDatesControllerTest.java#L188 ]
[test: getForAnUnknownConferenceRedirectsToTheList : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/ChangeConferenceDatesControllerTest.java#L94 ]
[test: getForADroppedConferenceRedirectsToItsDetailPage : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/ChangeConferenceDatesControllerTest.java#L213 ]
[test: getRendersTheFormPrefilledWithTheDatesInForce : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/ChangeConferenceDatesControllerTest.java#L79 ]
[test: anEndBeforeTheStartRendersUnderTheEndInput : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/ChangeConferenceDatesControllerTest.java#L151 ]
[test: aConferenceGoneInAnotherTabIsSaidOnTheFormWithTheTypedDatesKept : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/ChangeConferenceDatesControllerTest.java#L169 ]

## S12: Changing a booked flight or itinerary validates input and previews before applying @v1 [proposed]

- Given: the change-flight form at /booked-flights/{id}, or the change-itinerary form at /booked-itineraries/{id}/change
- When: the owner submits it
- Then: a departure not in the future, an arrival before departure, an overlapping leg, an invalid airport code or a time zone that cannot be resolved is reported on the flight form
- and: when the flight lookup finds nothing, the form shows "No flight found for {number} on {date} (or the API key is not configured). Edit the fields manually."; when one flight number has several legs that day, the owner picks one
- and: on the itinerary form, action=preview evaluates the change and writes nothing; action=apply writes only a clean change, then redirects to /booked-flights?changed={id}&how=schedule-change or how=rebooked

_Source: `src/main/java/dev/ted/jittertravel/web/ChangeFlightController.java`, `src/main/java/dev/ted/jittertravel/web/ChangeFlightRequest.java`, `src/main/java/dev/ted/jittertravel/web/ChangeFlightItineraryController.java`, `src/main/java/dev/ted/jittertravel/web/ChangeFlightItineraryRequest.java`._

[test: eachRefusalLandsWhereTheFormShowsIt : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/ChangeFlightControllerTest.java#L52 ]
[test: applyingGoesToTheFlightsListCarryingTheOutcomeKeyNotTheSentence : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/web/ChangeFlightItineraryControllerTest.java#L183 ]
