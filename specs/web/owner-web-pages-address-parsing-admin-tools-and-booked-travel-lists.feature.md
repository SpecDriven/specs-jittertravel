# Owner web pages: address parsing, admin tools and booked-travel lists

_Extracted from `src/main/java/dev/ted/jittertravel/web`._

This part covers the address-parse endpoint, the owner's admin pages, the lists of booked hotels and trains, and the flight booking forms. The admin pages are home, settings, backup, restore, database and legacy migration. The flight forms are a single flight and a pasted itinerary. List pages take a time filter that falls back to future stays when the value is missing or unknown. The hotel list also has a switch for cancelled stays. Booking forms send the user to the read-only page when the app is in read-only mode.

## S1: An address query is parsed into JSON fields @v1 [proposed]

- Given: the address service can resolve "Kolpingstr. 1, 63867 Johannesberg"
- When: a GET is sent to /api/parse-address with that text as q
- Then: the response is 200 with JSON
- and: the JSON includes "city":"Johannesberg" and "postalCode":"63867"

_Source: `src/test/java/dev/ted/jittertravel/web/AddressParseControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/AddressParseController.java`._

## S2: An unresolvable address query is refused as unprocessable @v1 [proposed]

- Given: the address service cannot resolve "nowhere"
- When: a GET is sent to /api/parse-address?q=nowhere
- Then: the response status is 422 Unprocessable Entity

_Source: `src/test/java/dev/ted/jittertravel/web/AddressParseControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/AddressParseController.java`._

## S3: The backup downloads as a JSON attachment @v1 [proposed]

- Given: the backup service produces "jittertravel-backup-local-2026-05-31T100000Z.json"
- When: the owner requests /admin/backup
- Then: the response is JSON
- and: Content-Disposition is attachment with that filename

_Source: `src/test/java/dev/ted/jittertravel/web/AdminControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/AdminController.java`._

## S4: The settings page reports whether cookies are marked Secure @v1 [proposed]

- Given: a request over plain http with no X-Forwarded-Proto header
- When: the owner opens /admin/settings
- Then: the page says "Cookies on this request are NOT marked Secure." and "No usable X-Forwarded-Proto reached the app"
- and: over a secure request it says "Cookies on this request are marked Secure, as desired."
- and: on a secure request, the missing forwarded header is explained as consumed after being applied
- and: /admin itself no longer shows the cookie readout

_Source: `src/test/java/dev/ted/jittertravel/web/AdminControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/AdminController.java`._

## S5: The database page lists row counts per table @v1 [proposed]

- Given: event_log and command_log tables with row counts
- When: the owner opens /admin/database
- Then: an HTML page lists event_log and command_log
- and: right after a truncation, it shows an amber "⚠" warning to "Restart the application before importing"

_Source: `src/test/java/dev/ted/jittertravel/web/AdminControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/AdminController.java`._

## S6: The legacy event migration previews counts and runs when confirmed @v1 [proposed]

- Given: a migration preview with types to rename
- When: the owner opens /admin/migrate-legacy-events
- Then: the page shows "Events scanned", "Types to rename", "Rows to write", "Renaming types is one-way." and "Run migration"
- and: it links to both the site home (/) and /admin
- and: when there is nothing to rename, the one-way warning is left out
- and: posting confirm=MIGRATE runs the migration and shows "✓ Migration complete and verified."

_Source: `src/test/java/dev/ted/jittertravel/web/AdminControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/AdminController.java`._

## S7: Restore applies a valid backup or shows its errors @v1 [proposed]

- Given: backup content posted to /admin/restore
- When: the content has no errors
- Then: the success page shows how many commands and events were restored and how many were skipped
- and: when the content has errors, the restore form shows the errors with the content kept
- and: posting to /admin/restore/validate shows valid command and event counts, or the errors, without restoring

_Source: `src/main/java/dev/ted/jittertravel/web/AdminController.java`._

## S8: The family-notify probe sends a test email and remembers the result @v1 [proposed]

- Given: the email client is configured
- When: the owner posts to /admin/family-notify/probe
- Then: a fixed test email is sent, its success is remembered, and the owner is redirected to /admin/settings, where the result flashes once
- and: if sending fails, the settings page is shown with the failure reason
- and: if the email client is not configured, the settings page is shown and nothing is sent

_Source: `src/main/java/dev/ted/jittertravel/web/AdminController.java`._

## S9: Booked hotels hide cancelled stays unless the switch shows them @v1 [proposed]

- Given: one live stay at Hotel Moxy and one cancelled stay at Hotel Adlon
- When: the owner opens /booked-hotels
- Then: Hotel Moxy is listed and Hotel Adlon is not
- and: the switch reads "Show Canceled Hotels" with a count of 1
- When: the owner opens /booked-hotels?filter=all&cancelled=show
- Then: Hotel Adlon appears as plain text, with no link and no actions
- and: the All filter link keeps cancelled=show

_Source: `src/test/java/dev/ted/jittertravel/web/BookedHotelsControllerTest.java`, `src/main/java/dev/ted/jittertravel/web/BookedHotelsController.java`, `src/main/java/dev/ted/jittertravel/web/BookedHotelsRenderer.java`._

## S10: Booked trains default to the future view @v1 [proposed]

- Given: the booked trains page
- When: it is opened with no filter, or with an unknown filter such as filter=bogus
- Then: future trains are listed
- and: filter=all lists all trains
- and: an empty future list says "No upcoming trains."

_Source: `src/test/java/dev/ted/jittertravel/web/BookedTrainsWebIntegrationTest.java`, `src/main/java/dev/ted/jittertravel/web/BookedTrainsController.java`, `src/main/java/dev/ted/jittertravel/web/BookedTrainsRenderer.java`._

## S11: The flight booking form is prefilled from date and city links @v1 [proposed]

- Given: the app is not read-only
- When: the owner opens /book-flight?date=…&fromCity=…&toCity=…
- Then: departure is 09:00 on that date and arrival is three hours later
- and: without a date, the day defaults to one week from today
- and: an airport is filled in only for a city that has exactly one airport
- and: in read-only mode, the owner is redirected to /read-only

_Source: `src/main/java/dev/ted/jittertravel/web/BookFlightController.java`._

## S12: A rejected flight booking shows field errors and gets a fresh id @v1 [proposed]

- Given: a submitted flight booking
- When: the departure is not in the future, the arrival is not after the departure, the flight overlaps a booked leg, an airport code is invalid, or a zone cannot be resolved
- Then: the form is shown again with the error
- and: the flight id is replaced with a new one
- and: a successful booking redirects to the return page, by default /booked-flights

_Source: `src/main/java/dev/ted/jittertravel/web/BookFlightController.java`._
