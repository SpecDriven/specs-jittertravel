# Conference attendance decisions and checking entered countries and places

_Extracted from `src/main/java/dev/ted/jittertravel/domain`._

This part covers two things. First, how Ted records that he will attend a conference he has planned, or that he will not. Second, how a country, state or city typed into a booking is checked before the booking is written. Confirming and declining are refused only when the conference does not exist. Neither depends on the conference dates. The countries table maps ISO 3166-1 alpha-2 codes to English names. When an entered location fails its checks, every problem is reported together, each against the role and field at fault.

## S1: Confirming attendance records the basis and the moment of confirmation @v1 [proposed]

- Given: a conference that exists
- When: Ted confirms attendance with any attendance basis and a confirmation time
- Then: exactly one ConferenceAttendanceConfirmed event is emitted
- and: it carries the conference id, that basis and that confirmation time

_Source: `src/main/java/dev/ted/jittertravel/domain/ConfirmConferenceAttendanceCommand.java`, `src/main/java/dev/ted/jittertravel/domain/ConferenceAttendanceConfirmed.java`, `src/test/java/dev/ted/jittertravel/domain/ConfirmConferenceAttendanceCommandTest.java`._

## S2: Confirming attendance for an unknown conference is refused @v1 [proposed]

- Given: no live conference with the given id, because it was never planned, was cancelled or was declined
- When: Ted confirms attendance
- Then: the command fails with ConferenceNotFound
- and: no event is emitted

_Source: `src/main/java/dev/ted/jittertravel/domain/ConfirmConferenceAttendanceCommand.java`, `src/main/java/dev/ted/jittertravel/domain/ConfirmConferenceAttendanceContext.java`, `src/test/java/dev/ted/jittertravel/domain/ConfirmConferenceAttendanceCommandTest.java`._

## S3: A confirmed attendance without a basis is rejected @v1 [proposed]

- Given: a conference id and a confirmation time
- When: a ConferenceAttendanceConfirmed is created with no basis
- Then: it fails with a message containing "basis must not be null"

_Source: `src/main/java/dev/ted/jittertravel/domain/ConferenceAttendanceConfirmed.java`, `src/test/java/dev/ted/jittertravel/domain/ConfirmConferenceAttendanceCommandTest.java`._

## S4: Declining a conference records the reason and the moment of declining, whatever the dates @v1 [proposed]

- Given: a conference that exists
- When: Ted declines it with the reason "Schedule clash" and a decline time
- Then: exactly one ConferenceAttendanceDeclined event is emitted with the conference id, the reason and the decline time
- and: no check against the conference dates stops the decline

_Source: `src/main/java/dev/ted/jittertravel/domain/DeclineConferenceCommand.java`, `src/main/java/dev/ted/jittertravel/domain/ConferenceAttendanceDeclined.java`, `src/test/java/dev/ted/jittertravel/domain/DeclineConferenceCommandTest.java`._

## S5: Declining without a reason records an empty reason @v1 [proposed]

- Given: a conference that exists
- When: Ted declines it with no reason
- Then: the emitted ConferenceAttendanceDeclined has the empty string "" as its reason, not null

_Source: `src/main/java/dev/ted/jittertravel/domain/DeclineConferenceCommand.java`, `src/main/java/dev/ted/jittertravel/domain/ConferenceAttendanceDeclined.java`, `src/test/java/dev/ted/jittertravel/domain/DeclineConferenceCommandTest.java`._

## S6: Declining an unknown conference is refused @v1 [proposed]

- Given: no live conference with the given id
- When: Ted declines it
- Then: the command fails with ConferenceNotFound

_Source: `src/main/java/dev/ted/jittertravel/domain/DeclineConferenceCommand.java`, `src/main/java/dev/ted/jittertravel/domain/DeclineConferenceContext.java`, `src/test/java/dev/ted/jittertravel/domain/DeclineConferenceCommandTest.java`._

## S7: A country code returns the country's English name, and only codes are known @v1 [proposed]

- Given: the countries table
- When: a name is looked up for "US", "GB", "CZ", "TR" or " de "
- Then: the result is "United States", "United Kingdom", "Czechia", "Türkiye" or "Germany", ignoring case and surrounding spaces
- and: "Germany", "UK", "USA", "XX", the empty string and null are not known codes and return no name

_Source: `src/main/java/dev/ted/jittertravel/domain/Countries.java`, `src/test/java/dev/ted/jittertravel/domain/CountriesTest.java`._

## S8: A country name gets a code only when it matches the table's spelling exactly @v1 [proposed]

- Given: the countries table
- When: a code is looked up for "United States", " sweden " or "Türkiye"
- Then: the result is "US", "SE" or "TR", ignoring case and surrounding spaces
- and: "USA", "UK", "Turkey" and the empty string return no code

_Source: `src/main/java/dev/ted/jittertravel/domain/Countries.java`, `src/test/java/dev/ted/jittertravel/domain/CountriesTest.java`._

## S9: The full country list holds 198 countries, sorted by name @v1 [proposed]

- Given: the countries table
- When: every country is listed
- Then: there are 198 entries: the UN members plus Hong Kong, Taiwan, Puerto Rico, Palestine and Vatican City
- and: the list is sorted by name, not by code, from Afghanistan (AF) to Zimbabwe (ZW), so Germany (DE) comes before Ghana (GH)

_Source: `src/main/java/dev/ted/jittertravel/domain/Countries.java`, `src/main/java/dev/ted/jittertravel/domain/Country.java`, `src/test/java/dev/ted/jittertravel/domain/CountriesTest.java`._

## S10: An entered country must be a known code, and the US, Canada and Australia need a valid state or province @v1 [proposed]

- Given: an address entered for a booking
- When: its country is "Germany" instead of a code
- Then: the address gets one COUNTRY problem, "Unknown country", against the given role
- and: in the US or Australia a blank region gives "State required for <country name>", and in Canada a blank region gives "Province required for Canada"
- and: a region that is not one of that country's codes, such as "Colorado" in the US, gives "Not a state of United States", or "Not a province of Canada" in Canada
- and: a region such as "CO" in the US, free text in other countries, and a blank country all pass
- and: running the check throws every problem together, and passes quietly when there are none

_Source: `src/main/java/dev/ted/jittertravel/domain/EnteredCountry.java`, `src/main/java/dev/ted/jittertravel/domain/InvalidEnteredLocation.java`, `src/main/java/dev/ted/jittertravel/domain/InvalidLocationEntry.java`, `src/test/java/dev/ted/jittertravel/domain/EnteredCountryTest.java`._

## S11: A train station's country is checked but no region is required @v1 [proposed]

- Given: a train station in country "US" with no region
- When: its country is checked
- Then: there is no problem, because a station has no region field
- and: a station whose country is "UK" gets a COUNTRY problem under its departure or arrival role
- and: an absent address has nothing to check and gets no problems

_Source: `src/main/java/dev/ted/jittertravel/domain/EnteredCountry.java`, `src/test/java/dev/ted/jittertravel/domain/EnteredCountryTest.java`._

## S12: An entered building and city must both be filled in, and the city must not look like a building @v1 [proposed]

- Given: a station or hotel entered with its name and city
- When: the location is checked
- Then: a blank name gives "Name is required" on the venue name field
- and: a blank city gives "City is required" on the city field
- and: a city with brackets, a digit, or a whole word such as "Hbf", "Centraal", "Airport" or "Hotel" gives "Venue name, not a city" on the city field
- and: country problems are added on their own fields, there is at most one problem per field, and all the problems are reported together

_Source: `src/main/java/dev/ted/jittertravel/domain/EnteredLocation.java`, `src/main/java/dev/ted/jittertravel/domain/InvalidEnteredLocation.java`, `src/main/java/dev/ted/jittertravel/domain/InvalidLocationEntry.java`._
