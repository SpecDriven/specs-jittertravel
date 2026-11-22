# Conference list, confirming and declining attendance, email preview and the event log

_Extracted from `src/main/java/dev/ted/jittertravel/web`._

The /conferences list shows each conference's name, a day-only date range that links to the calendar, city, country and venue, and the actions that suit its state. Separate pages confirm or decline attendance. The admin email preview shows every family email and sends each group of emails only to Ted's own address. The admin event log is paged and can be filtered. Also covered: the labels on the grey context bands of the problem calendar, and the problem count shown at the top of a rejected form.

## S1: An empty conference list says why it is empty for the chosen filter @v1 [proposed]

- Given: there are no conferences to show
- When: /conferences is rendered with the All filter
- Then: the page says "No conferences yet." and has no table cells
- and: with the Upcoming filter it says "No upcoming conferences."
- and: the link for the active filter (/conferences?filter=all or /conferences?filter=future) is marked active

_Source: `src/test/java/dev/ted/jittertravel/web/ConferencesRendererTest.java`, `src/main/java/dev/ted/jittertravel/web/ConferencesRenderer.java`._

## S2: The date range is one day-only column that links to the calendar from the week before @v1 [proposed]

- Given: a conference that runs from 1:00 AM on Sun 6/7 to Wed 6/10 in Europe/Amsterdam
- When: the conference list is rendered
- Then: the Dates column shows "Sun 6/7 -" and "Wed 6/10" as two units that never break inside themselves
- and: no time of day appears in the column
- and: the range links to /calendar?day=2026-06-07#w-2026-05-31, using the day in the conference's own time zone and the Sunday of the week before

_Source: `src/test/java/dev/ted/jittertravel/web/ConferencesRendererTest.java`, `src/main/java/dev/ted/jittertravel/web/ConferencesRenderer.java`._

## S3: City, country and venue share one cell @v1 [proposed]

- Given: a conference in Ede, Netherlands with a venue name
- When: the list is rendered
- Then: the City cell reads "Ede, Netherlands" with the venue on a line underneath
- and: when the country is empty, the city appears alone with no trailing comma
- and: when the venue name is blank, there is no venue line at all

_Source: `src/test/java/dev/ted/jittertravel/web/ConferencesRendererTest.java`, `src/main/java/dev/ted/jittertravel/web/ConferencesRenderer.java`._

## S4: A watched conference with a CFP offers Submitted, Ticket Bought and Decline @v1 [proposed]

- Given: a conference Ted is only watching, with a call for papers and nothing submitted
- When: the list is rendered
- Then: its Going? column shows a "Maybe" chip
- and: its actions link to /conferences/{id}/talk?outcome=SUBMITTED ("Submitted"), to /conferences/{id}/confirm?basis=TICKET_PURCHASED ("Ticket Bought") and to /conferences/{id}/decline ("Decline")

_Source: `src/test/java/dev/ted/jittertravel/web/ConferencesRendererTest.java`, `src/main/java/dev/ted/jittertravel/web/ConferencesRenderer.java`._

## S5: The confirm page opens with the given basis already chosen @v1 [proposed]

- Given: a conference that exists
- When: GET /conferences/{id}/confirm?basis=TICKET_PURCHASED is requested
- Then: the confirm attendance page is shown with that basis already selected
- and: with no basis, or a value that is not a basis, the page opens with nothing selected

_Source: `src/main/java/dev/ted/jittertravel/web/ConfirmConferenceAttendanceController.java`._

## S6: Confirming without a basis shows an error on the confirm page @v1 [proposed]

- Given: a conference that exists
- When: POST /conferences/{id}/confirm is sent with no basis, or with an unknown one
- Then: the confirm page is shown again with the error "Choose why you are going."
- and: no confirmation is recorded

_Source: `src/main/java/dev/ted/jittertravel/web/ConfirmConferenceAttendanceController.java`._

## S7: Confirming or declining records the decision and returns to the list @v1 [proposed]

- Given: a conference that exists
- When: POST /conferences/{id}/confirm is sent with a valid basis, or POST /conferences/{id}/decline is sent with an optional reason
- Then: the attendance or the decline is recorded, with an empty reason when none was given
- and: the browser is redirected to /conferences

_Source: `src/main/java/dev/ted/jittertravel/web/ConfirmConferenceAttendanceController.java`, `src/main/java/dev/ted/jittertravel/web/DeclineConferenceController.java`, `src/main/java/dev/ted/jittertravel/web/DeclineConferenceRequest.java`._

## S8: Stale or malformed conference links fall back quietly to the list, and read-only mode blocks the change @v1 [proposed]

- Given: a conference id that is malformed or names a conference that no longer exists
- When: the confirm or decline page is requested, or the form is submitted
- Then: the browser is redirected to /conferences with no message
- and: when the app is in read-only mode, submitting redirects to /read-only

_Source: `src/main/java/dev/ted/jittertravel/web/ConfirmConferenceAttendanceController.java`, `src/main/java/dev/ted/jittertravel/web/DeclineConferenceController.java`._

## S9: The email preview sends a group only to Ted's own address and reports how far it got @v1 [proposed]

- Given: a Brevo key and a TED_REPLY_EMAIL are configured
- When: Ted presses a group's send button (/admin/email-preview/send for flight emails, /admin/email-preview/send-conferences for conference emails)
- Then: that group's emails are sent one by one to the reply-to address only
- and: on success the page redirects and shows, for example, "5 emails sent to … at …" with a "Send them again" button
- and: on failure the page shows "Sending failed after 2 of 5" (or "Sending failed before any went out"), the reason, which emails went out, and a "Try again" button
- and: the other group's remembered result is left as it was

_Source: `src/main/java/dev/ted/jittertravel/web/EmailPreviewController.java`, `src/main/java/dev/ted/jittertravel/web/EmailPreviewPanels.java`, `src/main/java/dev/ted/jittertravel/web/EmailPreviewMemory.java`, `src/main/java/dev/ted/jittertravel/web/EmailGroup.java`._

## S10: The email preview cannot send without a key or a reply-to address @v1 [proposed]

- Given: JITTERTRAVEL_BREVO_API_KEY is missing, or TED_REPLY_EMAIL is blank
- When: the email preview page is shown
- Then: each group's panel says "Cannot send yet", explains which variable is missing, and asks for it to be set on the app service before a redeploy
- and: no remembered result is shown, and the previews still render

_Source: `src/main/java/dev/ted/jittertravel/web/EmailPreviewPanels.java`, `src/main/java/dev/ted/jittertravel/web/EmailPreviewController.java`._

## S11: The event log pages through filtered events and marks matched identifiers @v1 [proposed]

- Given: stored events, and a filter of event types and identifiers
- When: /admin/eventlog is requested with a page number
- Then: up to 50 matching events are shown, and a page number out of range is clamped to the first or last page
- and: the paging links carry the filter as type and id parameters
- and: the filter is summarised as, for example, "2 types and 1 identifier selected."
- and: each matched identifier in a payload is marked, with the longer identifier winning where one begins another

_Source: `src/main/java/dev/ted/jittertravel/web/EventLogController.java`, `src/main/java/dev/ted/jittertravel/web/EventLogFilterView.java`._

## S12: Context bands on the problem calendar name the thing and its exact dates @v1 [proposed]

- Given: a conference, gathering, hotel stay or trip on the schedule
- When: its context band label is built
- Then: a named item reads like "dev2next, Chicago · Sep 14–18", and a stay names its hotel, as in "Reichshof, Chicago · Sep 14–18"
- and: travel reads "London → Berlin · Sep 14"
- and: a range that crosses a month reads "Sep 30 – Oct 2", and a single day appears once

_Source: `src/test/java/dev/ted/jittertravel/web/ContextBandTest.java`, `src/main/java/dev/ted/jittertravel/web/ContextBand.java`._
