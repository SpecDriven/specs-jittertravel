# Change forms for hotels, trains, gatherings and private events, clearing conflicts, and the conference detail page

_Extracted from `src/main/java/dev/ted/jittertravel/web`._

The owner can reopen a booked hotel, booked train or planned gathering at its own path to change it, re-match a private event to a different location, clear a city conflict between a gathering and a conference, and view one conference in full. Which record a change applies to always comes from the path, never from the form. Refused changes re-render the form with each error shown under the input that fixes it. A saved change returns the owner to the calendar or itinerary day it was started from, or otherwise to the list. Unknown or malformed ids send the owner back to the list without a message.

## S1: Opening a known hotel booking shows its change form @v1 [proposed]

- Given: a booked hotel stay exists
- When: the owner opens /booked-hotels/{id}
- Then: the change form is shown, prefilled with the stay's details
- and: the form posts back to /booked-hotels/{id} and has no hidden booking id field
- and: the page has no cancel option and no HTML comments

_Source: `src/test/java/dev/ted/jittertravel/web/ChangeHotelControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/ChangeHotelController.java`, `src/main/java/dev/ted/jittertravel/web/ChangeHotelRequest.java`._

## S2: An unknown or malformed id on a change page returns silently to its list @v1 [proposed]

- Given: no hotel booking, train trip, gathering or private event has the id in the path, or the id is not a valid UUID
- When: the owner opens its change page
- Then: the owner is redirected to /booked-hotels, /booked-trains, /planned-gatherings or /planned-private-events, with no message

_Source: `src/test/java/dev/ted/jittertravel/web/ChangeHotelControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/ChangeHotelController.java`, `src/main/java/dev/ted/jittertravel/web/ChangeTrainController.java`, `src/main/java/dev/ted/jittertravel/web/ChangeGatheringController.java`, `src/main/java/dev/ted/jittertravel/web/ChangePrivateEventMatchingLocationController.java`._

## S3: A saved change applies to the record named in the path and returns to the list @v1 [proposed]

- Given: a valid hotel change is submitted to /booked-hotels/{id}
- When: the change is accepted
- Then: the booking named by the path is changed
- and: the owner is redirected to /booked-hotels

_Source: `src/test/java/dev/ted/jittertravel/web/ChangeHotelControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/ChangeHotelController.java`._

## S4: A change started from the calendar or itinerary returns to that day @v1 [proposed]

- Given: the owner opened a change form with from=itinerary or from=calendar and a day
- When: the change form is shown
- Then: it carries from and day as hidden inputs
- When: the change is saved
- Then: from=itinerary returns to /itinerary?date={day}, from=calendar to /calendar?day={day}
- and: with no origin the owner returns to the list, such as /planned-gatherings

_Source: `src/test/java/dev/ted/jittertravel/web/ChangeHotelControllerTest.java`, `src/test/java/dev/ted/jittertravel/web/ChangeGatheringControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/ChangeGatheringController.java`, `src/main/java/dev/ted/jittertravel/web/ChangeHotelController.java`._

## S5: A record that disappears before saving is reported on the form @v1 [proposed]

- Given: the hotel booking, train or gathering was removed after its form was opened
- When: the owner submits the change
- Then: the form re-renders with the not-found message as a page-wide error and no count of fields to fix
- and: the form still posts back to the same path

_Source: `src/test/java/dev/ted/jittertravel/web/ChangeHotelControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/ChangeHotelController.java`, `src/main/java/dev/ted/jittertravel/web/ChangeTrainController.java`, `src/main/java/dev/ted/jittertravel/web/ChangeGatheringController.java`._

## S6: Refused hotel dates and locations are shown under the input that fixes them @v1 [proposed]

- Given: a hotel change is submitted
- When: check-in is not in the future, check-out is less than one calendar day after check-in, the cancel-by date is invalid, the location is invalid, or the time zone cannot be found
- Then: the form re-renders with the error on checkIn, checkOut, cancelBy, the location fields, or zone ("Could not determine the time zone from the location — please choose one.")
- and: a hotel name pasted into the city gets its error on the city field
- and: an unparsable date re-renders the form without trying the change

_Source: `src/test/java/dev/ted/jittertravel/web/ChangeHotelControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/ChangeHotelController.java`._

## S7: Refused gathering changes are shown on the matching input @v1 [proposed]

- Given: a gathering change is submitted
- When: the venue is refused for its country or region, the date is not in the future, the end time is not after the start time, or the time zone cannot be found
- Then: the change-gathering form re-renders with exactly one error on country, region, date, endTime or zone
- and: a blank or unparsable date or time re-renders the form without trying the change

_Source: `src/test/java/dev/ted/jittertravel/web/ChangeGatheringControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/ChangeGatheringController.java`, `src/main/java/dev/ted/jittertravel/web/ChangeGatheringRequest.java`._

## S8: Refused train changes, including an overlapping leg, are shown on the times @v1 [proposed]

- Given: a train change is submitted
- When: departure is not in the future, arrival is not after departure, or the trip overlaps a leg already booked
- Then: the change-train form re-renders with the error on departureDateTime or arrivalDateTime
- and: for an overlap, a notice about the leg already booked is shown beside it

_Source: `src/main/java/dev/ted/jittertravel/web/ChangeTrainController.java`, `src/main/java/dev/ted/jittertravel/web/ChangeTrainRequest.java`._

## S9: Re-matching a private event's location @v1 [proposed]

- Given: a planned private event exists
- When: the owner opens /planned-private-events/{id}/matching-location
- Then: the form is prefilled with the matching location currently in force
- When: the owner submits a new location
- Then: it is recorded and the owner returns to /planned-private-events
- and: a blank location re-renders the form with the error on locationForMatching
- and: if the event was cancelled meanwhile, the owner returns to the list silently

_Source: `src/main/java/dev/ted/jittertravel/web/ChangePrivateEventMatchingLocationController.java`, `src/main/java/dev/ted/jittertravel/web/ChangePrivateEventMatchingLocationRequest.java`._

## S10: Clearing a city conflict between a gathering and a conference @v1 [proposed]

- Given: the owner opens /clear-conflict with the gathering and conference ids, names, cities and date
- When: the owner submits it, with an optional reason
- Then: the conflict is cleared and the owner returns to /schedule-problems
- and: unparsable ids re-render the form with "This conflict could not be identified — please reopen it from Schedule Problems."
- and: in read-only mode the owner is redirected to /read-only
- and: any other failure re-renders the form with "Could not clear this conflict: " and the cause

_Source: `src/main/java/dev/ted/jittertravel/web/ClearConflictController.java`, `src/main/java/dev/ted/jittertravel/web/ClearConflictRequest.java`._

## S11: Viewing one conference in full @v1 [proposed]

- Given: a conference exists
- When: the owner opens /conferences/{id}
- Then: an HTML page titled with the conference name, including its venue, is returned as text/html;charset=UTF-8
- and: an unknown or malformed id redirects to /conferences
- and: a failure while building the page is not turned into a redirect

_Source: `src/test/java/dev/ted/jittertravel/web/ConferenceDetailControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/ConferenceDetailController.java`, `src/main/java/dev/ted/jittertravel/web/ConferenceDetailRenderer.java`._

## S12: Conference moves offered depend on talk and attendance state @v1 [proposed]

- Given: a conference the owner has declined
- Then: no moves are offered
- Given: a talk submitted and waiting
- Then: Accepted, Rejected and Withdrawn are offered, and Decline is not
- Given: an invitation to speak
- Then: Invitation Accepted and Decline are offered
- Given: nothing submitted or withdrawn
- Then: Submitted is offered unless the format is open space, Ticket Bought only if not already going, and Decline always
- Given: an accepted talk
- Then: Withdrawn and Decline are offered
- and: no state offers more than three moves, each a link to a page that confirms it

_Source: `src/main/java/dev/ted/jittertravel/web/ConferenceActions.java`._
