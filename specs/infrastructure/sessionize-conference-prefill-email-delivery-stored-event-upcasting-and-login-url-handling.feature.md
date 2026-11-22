# Sessionize conference prefill, email delivery, stored-event upcasting and login-URL handling

_Extracted from `src/main/java/dev/ted/jittertravel/infrastructure`._

This part covers several pieces of infrastructure. It turns a pasted Sessionize call-for-speakers URL into prefill values for the /plan-conference form, sends plain-text emails through Brevo, and builds subject and body from email template files. It also brings stored event payloads up to the current schema version and narrows the event log by type and identifier. Finally, it saves the browser's time zone in a cookie at login and redirects paths that end in a trailing slash to the path without it.

## S1: Only a Sessionize URL yields a slug, so no other address is ever fetched @v1 [proposed]

- Given: a pasted URL such as `https://sessionize.com/jfokus-2027/`, with or without `www`, http or https, a trailing slash or surrounding spaces
- When: the prefill reads the URL
- Then: the slug `jfokus-2027` is taken from it
- and: query strings and fragments such as `?utm_source=twitter` or `#cfs` are dropped rather than refused
- and: any other host, a lookalike host, a deeper path, a `..` path, a `file:` URL, text that is not a URL, an empty value or a null value yields no slug, and no request is made

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/SessionizePrefillService.java`, `src/test/java/dev/ted/jittertravel/infrastructure/SessionizePrefillServiceTest.java`._

## S2: A Sessionize page fills the conference fields it actually carries @v1 [proposed]

- Given: the Sessionize page and the calendar (.ics) file for `jfokus-2027`
- When: the prefill is put together
- Then: name is `Jfokus 2027`, without the `: Call for Speakers` suffix
- and: infoUrl is `https://www.jfokus.se/`, venueName is `Stockholm Waterfront Congress Centre`, venueCity is `Stockholm` and venueCountry is the ISO code `SE`
- and: venueState is empty because Sweden has no state list
- and: startDate and endDate carry guessed times of day, `2027-02-08T09:00` and `2027-02-10T17:00`

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/SessionizePrefillService.java`, `src/test/java/dev/ted/jittertravel/infrastructure/SessionizePrefillServiceTest.java`._

## S3: A US venue keeps its state as a code, and any other middle part is dropped @v1 [proposed]

- Given: the venue line reads `Centennial, Colorado, United States`
- When: the prefill is put together
- Then: venueCity is `Centennial`, venueState is `CO` and venueCountry is `US`
- and: the state resolves the deadline zone to `America/Denver`
- and: for `Steventon, Oxfordshire, United Kingdom`, venueState is empty and venueCountry is `GB`

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/SessionizePrefillService.java`, `src/test/java/dev/ted/jittertravel/infrastructure/SessionizePrefillServiceTest.java`._

## S4: The CFP deadline is given in the venue's zone, or left blank together with the submission URL @v1 [proposed]

- Given: the .ics DTSTART is `20261001T063000Z` and the venue is in Stockholm
- When: the prefill is put together
- Then: cfpClosesOn is `2026-10-01T08:30` and deadlineZone is `Europe/Stockholm`
- and: cfpSubmissionUrl is the normalized `https://sessionize.com/jfokus-2027/`
- and: when the venue location cannot be resolved, or the .ics is missing, has an empty DTSTART or an unreadable one, cfpClosesOn is blank and cfpSubmissionUrl is blank too, with no exception

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/SessionizePrefillService.java`, `src/test/java/dev/ted/jittertravel/infrastructure/SessionizePrefillServiceTest.java`._

## S5: Each prefill field degrades to blank on its own @v1 [proposed]

- Given: one of the two documents cannot be read
- When: the prefill is put together
- Then: fields from the other document are still filled, for example the venue city when the deadline is missing, or the page name when the .ics is missing
- and: when the page has been redesigned, the name falls back to the .ics SUMMARY with `: deadline to submit a session` removed, and folded .ics lines are unfolded first
- and: the SUMMARY inside the reminder alarm is never used as the conference name

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/SessionizePrefillService.java`, `src/test/java/dev/ted/jittertravel/infrastructure/SessionizePrefillServiceTest.java`._

## S6: An email is posted to Brevo with the API key and the exact message body @v1 [proposed]

- Given: an API key, a recipient `family@example.com` and a reply-to `ted@example.com` are configured
- When: a message is sent
- Then: a POST goes to `https://api.brevo.com/v3/smtp/email` with an `api-key` header
- and: the body holds sender `JitterTravel <notifications@jittertravel.com>`, replyTo, to, subject and textContent
- and: a blank reply-to leaves out the replyTo object entirely
- and: sending to a given address puts that address in `to` instead of the configured recipient
- and: the reported recipient is the configured recipient with surrounding spaces trimmed

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/BrevoEmailClientTest.java`._

## S7: An unconfigured or rejected email send fails @v1 [proposed]

- Given: the API key or the recipient is missing or blank
- When: the client is asked whether it is configured
- Then: it says it is not configured, and sending throws without making a request
- and: a reply-to is optional for being configured
- and: sending to a given address needs only a key and a non-blank address, and fails otherwise
- and: a non-2xx response from Brevo throws, so the command can be marked failed

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/BrevoEmailClientTest.java`._

## S8: An email template's first line is the subject and the rest is the body @v1 [proposed]

- Given: an email template file that starts with `Subject: `
- When: it is rendered
- Then: the subject is the text of that first line and the body is the rest, without the `Subject:` line
- and: an optional link line appears only when its value, such as calendarUrl, is given
- and: the body never has more than one blank line in a row
- and: a file with no subject line is refused with `email/<name>.txt must start with "Subject: "`
- and: in the trip-changed email, the `What changed:` section, the calendar link and the old-flights section are left out when there is nothing to put in them

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/EmailTemplatesTest.java`._

## S9: A stored event climbs every upcast step above its stored schema version @v1 [proposed]

- Given: a ConferencePlanned payload stored at schema version 2
- When: it is upcast
- Then: an absent format is set to `CALL_FOR_PAPERS`, an explicit format is kept, and the venue country becomes a code such as `US`
- and: the same payload stored at version 3 gets only the location-codes step, with no format added
- and: a payload already at the current version, or one that is not an object, is returned unchanged
- and: legacy full class names and the retired name `ConferenceTentativelyPlanned` are upcast the same way as the current logical name

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/EventPayloadUpcasterTest.java`, `src/test/java/dev/ted/jittertravel/infrastructure/ConferenceFormatUpcasterTest.java`._

## S10: A broken upcast path fails loudly instead of reading a stale shape @v1 [proposed]

- Given: a stored event whose version has no step to advance it, for example a HotelBooked row from before time zones were stored
- When: it is upcast
- Then: it fails with `No upcaster advances HotelBooked from schema version 1` and `was a rung retired before its rows were migrated?`
- and: two steps claiming the same move fail with `Two upcasters`
- and: an unregistered type name fails, naming that type
- and: when stored events are read, an unknown property is ignored, and the retired `state` field no longer fills region

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/EventPayloadUpcasterTest.java`, `src/test/java/dev/ted/jittertravel/infrastructure/EventJsonMapperFactoryTest.java`._

## S11: The event log filter keeps known type names and splits identifiers @v1 [proposed]

- Given: type names and identifier text are submitted to filter the event log
- When: the filter is built
- Then: no types and no identifiers give the empty filter
- and: known logical type names are kept in the order submitted, and duplicates or copies with extra spaces count once
- and: unknown names, full class names and retired names are dropped
- and: identifiers are split on spaces, commas, tabs and line breaks, with no blank entries and no duplicates

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/EventLogFilterTest.java`._

## S12: Login saves the browser zone in a cookie, and a trailing-slash path redirects @v1 [proposed]

- Given: a login form submits a valid IANA zone in its `browserZone` field
- When: the login succeeds
- Then: an HttpOnly cookie on path `/` holding the zone id is set on the same redirect to the saved request, and lasts 400 days
- and: the cookie is Secure only when the request is secure
- and: an absent, blank or unrecognized zone sets no cookie
- and: separately, any path ending in a slash such as `/admin/` gets a 308 redirect to `/admin` with the query string kept, while `/` is left alone

_Source: `src/main/java/dev/ted/jittertravel/infrastructure/ZoneCapturingAuthenticationSuccessHandler.java`, `src/main/java/dev/ted/jittertravel/infrastructure/TrailingSlashConfig.java`._
