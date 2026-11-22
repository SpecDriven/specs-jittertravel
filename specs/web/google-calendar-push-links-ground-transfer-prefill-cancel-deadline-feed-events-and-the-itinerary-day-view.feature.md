# Google Calendar push links, ground-transfer prefill, cancel-deadline feed events and the itinerary day view

_Extracted from `src/main/java/dev/ted/jittertravel/web`._

This part covers several helpers. A link opens Google Calendar's event editor already filled in, with times on the venue's own clock. The ground-transfer form is filled in from a missing-travel gap. Each hotel's free-cancellation deadline becomes a reminder event in the private calendar feed, which is written as iCalendar. List pages get a toolbar with their create action. The three-day itinerary page has date navigation. Links remember where they came from so the user returns there afterwards. Post-deploy tasks can be marked done.

## S1: The Google Calendar link carries the title, the local times and the zone @v1 [proposed]

- Given: an entry titled "SoCraTes" from 2026-09-15 09:00 to 2026-09-17 17:00 in Europe/Berlin, with location "Hotel Park, Soltau, DE" and details "https://socrates-conference.de"
- When: the Google Calendar link is built
- Then: it is https://calendar.google.com/calendar/render?action=TEMPLATE with text=SoCraTes and dates=20260915T090000/20260917T170000
- and: ctz=Europe%2FBerlin, with the location and details URL-encoded

_Source: `src/main/java/dev/ted/jittertravel/web/GoogleCalendarLink.java`, `src/test/java/dev/ted/jittertravel/web/GoogleCalendarLinkTest.java`._

## S2: Link times use the venue's wall clock, never UTC @v1 [proposed]

- Given: a dinner from 19:30 to 22:00 in Berlin
- When: the link is built
- Then: dates reads 20260915T193000/20260915T220000
- and: neither the UTC time 17:30 nor a trailing Z appears

_Source: `src/main/java/dev/ted/jittertravel/web/GoogleCalendarLink.java`, `src/test/java/dev/ted/jittertravel/web/GoogleCalendarLinkTest.java`._

## S3: An end recorded in another zone is shown in the start's zone @v1 [proposed]

- Given: a start at 09:00 in Berlin and an end at 03:00 in Denver on the same date
- When: the link is built
- Then: dates reads 20260915T090000/20260915T110000 and ctz is Europe/Berlin

_Source: `src/main/java/dev/ted/jittertravel/web/GoogleCalendarLink.java`, `src/test/java/dev/ted/jittertravel/web/GoogleCalendarLinkTest.java`._

## S4: A blank location and blank details are left out, and the title is URL-encoded @v1 [proposed]

- Given: an entry titled "Ted & Friends: dinner" with a blank location and blank details
- When: the link is built
- Then: it contains text=Ted+%26+Friends%3A+dinner
- and: it has no location or details parameter at all

_Source: `src/main/java/dev/ted/jittertravel/web/GoogleCalendarLink.java`, `src/test/java/dev/ted/jittertravel/web/GoogleCalendarLinkTest.java`._

## S5: The calendar icon link opens in a new tab with a tooltip @v1 [proposed]

- Given: a Google Calendar URL and the tooltip "Add Dinner to Google Calendar"
- When: the icon control is rendered
- Then: it is a link with class gcal-add, that href and that title, target _blank and rel noopener, holding the calendar-arrow icon
- and: the labelled version also shows its words in a span with class gcal-label

_Source: `src/main/java/dev/ted/jittertravel/web/GoogleCalendarLink.java`, `src/test/java/dev/ted/jittertravel/web/GoogleCalendarLinkTest.java`._

## S6: A missing-travel gap fills in both ends of the ground-transfer form @v1 [proposed]

- Given: a missing-travel gap from Johannesberg to Frankfurt, where each end resolves to exactly one stay: check-out on Sep 13 at 11:00 and check-in at 15:00
- When: the ground-transfer form is prefilled
- Then: origin is hotel:seminarzentrum and destination is hotel:holiday-inn
- and: the date is 2026-09-13, departure is 11:00 and arrival is 15:00

_Source: `src/main/java/dev/ted/jittertravel/web/GroundTransferPreselection.java`, `src/test/java/dev/ted/jittertravel/web/GroundTransferPreselectionTest.java`._

## S7: An end that cannot be settled is left for the user to choose @v1 [proposed]

- Given: a gap where only the origin resolves to one candidate
- When: the form is prefilled
- Then: the origin is set and the destination stays empty
- and: the arrival time keeps the form's default of 12:45
- and: when only the destination resolves, the origin stays empty and the date and arrival time come from the destination

_Source: `src/main/java/dev/ted/jittertravel/web/GroundTransferPreselection.java`, `src/test/java/dev/ted/jittertravel/web/GroundTransferPreselectionTest.java`._

## S8: A prefilled arrival before the departure is moved to 45 minutes after it @v1 [proposed]

- Given: an origin with check-out at 11:00 and a destination flight departing at 09:00
- When: the form is prefilled
- Then: departure is 11:00 and arrival is 11:45

_Source: `src/main/java/dev/ted/jittertravel/web/GroundTransferPreselection.java`, `src/test/java/dev/ted/jittertravel/web/GroundTransferPreselectionTest.java`._

## S9: A future free-cancel deadline becomes a feed event with 48h, 24h and 4h alarms @v1 [proposed]

- Given: a hotel booking "Grand Hotel" that is not cancelled and has a free-cancel deadline 10 days from now
- When: the calendar feed's events are gathered
- Then: there is one event with UID "<booking id>-cancelby@jittertravel" that starts at the deadline and lasts 15 minutes
- and: its summary is "Free-cancel deadline: Grand Hotel", its description names the city and the check-in and check-out dates, and its alarms are -PT48H, -PT24H and -PT4H

_Source: `src/main/java/dev/ted/jittertravel/web/HotelCancelDeadlineSource.java`, `src/test/java/dev/ted/jittertravel/web/HotelCancelDeadlineSourceTest.java`._

## S10: Bookings that are cancelled, have no deadline or have a past deadline give no feed event @v1 [proposed]

- Given: a booking with no deadline, a cancelled booking with a future deadline, or a booking whose deadline passed an hour ago
- When: the feed's events are gathered
- Then: none of them produces an event

_Source: `src/main/java/dev/ted/jittertravel/web/HotelCancelDeadlineSource.java`, `src/test/java/dev/ted/jittertravel/web/HotelCancelDeadlineSourceTest.java`._

## S11: The list toolbar puts the filters first and a green create link last @v1 [proposed]

- Given: a list page with two filters and the create action "Book another hotel" pointing to /book-hotel
- When: the toolbar is rendered
- Then: a list-toolbar row holds the filters in order, followed by a create-link anchor to /book-hotel
- and: the create link is filled #1e7a1e with white text, sits at the right edge and turns #1b6f1b on hover

_Source: `src/main/java/dev/ted/jittertravel/web/ListToolbar.java`, `src/test/java/dev/ted/jittertravel/web/ListToolbarTest.java`._

## S12: The itinerary page shows three days with navigation @v1 [proposed]

- Given: a signed-in viewer opens /itinerary, with or without a date
- When: the page renders
- Then: it shows the given date, or the viewer's today, and the two days after it, each with its entries, any ongoing stay, a missing hotel or being at home
- and: Previous and Next move by one day, Today is greyed out when today is on screen, and "Next entry" jumps to the first entry after the three days or is greyed out when there is none

_Source: `src/main/java/dev/ted/jittertravel/web/ItineraryController.java`, `src/main/java/dev/ted/jittertravel/web/ItineraryRenderer.java`._
