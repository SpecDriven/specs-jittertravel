# Hotel stays on the calendar and edit form, home cities, and recorded itinerary change outcomes

_Extracted from `src/main/java/dev/ted/jittertravel/application`._

This part covers how hotel bookings appear as lodging entries on the calendar and how the change-hotel edit form gets filled in. A hotel form's times are read in the hotel's own time zone, and the cancellation deadline is optional. Configured home cities count as one place for travel purposes. After an itinerary change is recorded, the app counts how many legs it added, moved and dropped. It also judges whether that change is news family should hear about.

## S1: A booked hotel appears on the calendar as a lodging entry @v1 [proposed]

- Given: a hotel booking for "Grand Hotel" at an address in Springfield, IL, checking in 2026-06-14 15:00 and out 2026-06-15 11:00
- When: the calendar is built from the events
- Then: it holds one lodging entry from check-in to check-out
- and: its main title is "Grand Hotel" and its subtitle is "Springfield, IL"
- and: the entry carries the owner's edit path "/booked-hotels/<bookingId>" and the cancel path "/booked-hotels/<bookingId>/cancel"

_Source: `src/test/java/dev/ted/jittertravel/application/HotelCalendarProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/HotelCalendarProjector.java`._

## S2: A changed hotel replaces its calendar entry instead of adding one @v1 [proposed]

- Given: a hotel booking that is later changed to "Seaside Resort" in Miami, FL, with new dates
- When: the calendar is built
- Then: there is still exactly one entry for that booking
- and: it shows the new hotel name, the new dates and "Miami, FL"
- and: when the booking is cancelled, its entry is removed

_Source: `src/test/java/dev/ted/jittertravel/application/HotelCalendarProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/HotelCalendarProjector.java`._

## S3: The hotel edit form is filled from the latest booking details @v1 [proposed]

- Given: a hotel booking with a name, an address, check-in and check-out times, a booking intent, a maps URL and a cancel-by deadline
- When: the details for that booking id are looked up
- Then: every value comes back as entered, with times shown as the hotel's local wall-clock time
- and: looking up a booking id that does not exist returns nothing

_Source: `src/test/java/dev/ted/jittertravel/application/HotelDetailsViewProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/HotelDetailsViewProjector.java`, `src/main/java/dev/ted/jittertravel/application/HotelDetailsView.java`._

## S4: A hotel change with no deadline clears the earlier cancel-by deadline @v1 [proposed]

- Given: a hotel booking that had a cancel-by deadline
- When: the booking is changed with no deadline given
- Then: the edit details show the changed values in place of the old ones
- and: the cancel-by deadline is empty

_Source: `src/test/java/dev/ted/jittertravel/application/HotelDetailsViewProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/HotelHandler.java`._

## S5: Hotel times are read in the hotel's own time zone @v1 [proposed]

- Given: a book-hotel or change-hotel form with an address and local check-in, check-out and cancel-by times
- When: the form is submitted
- Then: if a zone was picked, all three times use the picked zone; if not, they use the zone the address resolves to
- and: if no zone was picked and the address cannot be resolved, the command is rejected
- and: on a change, which booking is changed comes from the path, not from the submitted form

_Source: `src/main/java/dev/ted/jittertravel/application/HotelHandler.java`, `src/main/java/dev/ted/jittertravel/application/HotelBooking.java`._

## S6: Home city names match ignoring case and surrounding whitespace @v1 [proposed]

- Given: the home cities "San Francisco", "San Jose" and "Oakland"
- When: "san francisco" is checked
- Then: it is a home city, while "Amsterdam" is not
- and: configured names are trimmed, and blank entries are ignored

_Source: `src/test/java/dev/ted/jittertravel/application/HomeCitiesTest.java`, `src/main/java/dev/ted/jittertravel/application/HomeCities.java`._

## S7: Any two home cities count as the same location @v1 [proposed]

- Given: the Bay Area cities are configured as home
- When: "San Francisco" and "San Jose" are compared
- Then: they are the same location
- and: "San Francisco" and "Los Angeles" are different locations
- and: names that differ only in case or surrounding spaces, such as "Hamburg " and "Hamburg", are the same location

_Source: `src/test/java/dev/ted/jittertravel/application/HomeCitiesTest.java`, `src/main/java/dev/ted/jittertravel/application/HomeCities.java`._

## S8: The primary home city is the first non-blank city configured, with its original casing @v1 [proposed]

- Given: the home cities "  ", " San Jose " and "Oakland"
- When: the primary city is asked for
- Then: it is "San Jose"
- and: with no home cities configured, the primary city is empty, nothing counts as home, and two cities are the same location only when their names match

_Source: `src/test/java/dev/ted/jittertravel/application/HomeCitiesTest.java`, `src/main/java/dev/ted/jittertravel/application/HomeCities.java`._

## S9: A recorded rebooking counts the legs it added, moved and dropped @v1 [proposed]

- Given: one rebooking of itinerary HE80PP that books one leg, moves one leg and drops two legs as "Rebooked", all within the same command
- When: the itinerary's last change is asked for
- Then: the outcome is REBOOKING with 1 added, 1 moved and 2 dropped

_Source: `src/test/java/dev/ted/jittertravel/application/ItineraryChangeOutcomesProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/ItineraryChangeOutcomesProjector.java`._

## S10: Only the change's own legs count, and a cancellation made by hand is not a drop @v1 [proposed]

- Given: a leg booked by another command, plus a leg cancelled by hand and a leg moved within a schedule change of HE80PP
- When: the itinerary's last change is asked for
- Then: the outcome is SCHEDULE_CHANGE with 0 added, 1 moved and 0 dropped

_Source: `src/test/java/dev/ted/jittertravel/application/ItineraryChangeOutcomesProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/ItineraryChangeOutcomesProjector.java`._

## S11: Only an itinerary's latest change is kept @v1 [proposed]

- Given: a schedule change of HE80PP, followed by a rebooking that drops one leg
- When: the last change is asked for
- Then: the outcome is REBOOKING with 0 added, 0 moved and 1 dropped
- and: an itinerary that was never changed has no outcome

_Source: `src/test/java/dev/ted/jittertravel/application/ItineraryChangeOutcomesProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/ItineraryChangeOutcomesProjector.java`._

## S12: An itinerary change is news to family only when a journey's outer ends materially change @v1 [proposed]

- Given: the legs still flying before and after an itinerary change, grouped into journeys
- When: the two are compared
- Then: it is news if a journey now starts or ends in a different city, though two home cities count as the same place
- and: it is news if a journey's first departure or last arrival moves by more than 2 hours
- and: it is news if a journey is added that overlaps no earlier one, or an earlier journey is dropped; only the first difference found is reported
- and: changes to a connection's airports or times, or a departure moving 40 minutes, are not news

_Source: `src/main/java/dev/ted/jittertravel/application/ItineraryChangeNews.java`, `src/main/java/dev/ted/jittertravel/application/HomeCities.java`._
