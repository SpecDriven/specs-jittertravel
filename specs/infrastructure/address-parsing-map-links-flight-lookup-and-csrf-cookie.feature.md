# Address parsing, map links, flight lookup and CSRF cookie

_Extracted from `src/main/java/dev/ted/jittertravel/infrastructure`._

Fills address fields from a Nominatim geocoding answer, using country codes and, for the US, Canada and Australia, state codes. Builds Google Maps search links that show a country's name instead of its code. Turns AeroDataBox flight-number results into one or more flight segments the user can pick from. Makes sure the CSRF cookie is written before any page starts rendering.

## S1: A US address fills the country and state codes @v1 [proposed]

- Given: Nominatim returns house number 10345, road Park Meadows Drive, town Lone Tree, state Colorado, ISO3166-2-lvl4 US-CO, postcode 80124 and country_code us
- When: the address is parsed
- Then: street is 10345 Park Meadows Drive, city is Lone Tree, region is CO, postal code is 80124 and country is US
- and: the location for matching is Lone Tree

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/AddressParseServiceTest.java`, `src/main/java/dev/ted/jittertravel/infrastructure/AddressParseService.java`._

[test: aUsAddressFillsTheCountryAndStateCodes : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AddressParseServiceTest.java#L21 ]

## S2: A three-letter Australian state code is kept @v1 [proposed]

- Given: Nominatim returns city Sydney, ISO3166-2-lvl4 AU-NSW and country_code au
- When: the address is parsed
- Then: the region is NSW

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/AddressParseServiceTest.java`, `src/main/java/dev/ted/jittertravel/infrastructure/AddressParseService.java`._

[test: anAustralianStateCodeOfThreeLettersIsKept : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AddressParseServiceTest.java#L36 ]

## S3: Outside the US, Canada and Australia the region is the state name @v1 [proposed]

- Given: Nominatim returns a German address with village Johannesberg, state Bavaria, ISO3166-2-lvl4 DE-BY and country_code de
- When: the address is parsed
- Then: the region is Bavaria, the city is Johannesberg and the country is DE

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/AddressParseServiceTest.java`, `src/main/java/dev/ted/jittertravel/infrastructure/AddressParseService.java`._

[test: elsewhereTheRegionIsTheStateNameAsBefore : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AddressParseServiceTest.java#L49 ]

## S4: A missing or mismatched state code leaves the region blank for the form to ask for @v1 [proposed]

- Given: Nominatim returns a US address with state Georgia but no ISO3166-2-lvl4 field, or a Canadian address whose ISO3166-2-lvl4 is US-CO
- When: the address is parsed
- Then: the region is blank

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/AddressParseServiceTest.java`, `src/main/java/dev/ted/jittertravel/infrastructure/AddressParseService.java`._

[test: aUsAddressWithoutAStateCodeLeavesTheStateForTheFormToAskFor : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AddressParseServiceTest.java#L64 ]
[test: aStateCodeFromAnotherCountryIsNotTaken : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AddressParseServiceTest.java#L77 ]

## S5: No country_code leaves the country blank @v1 [proposed]

- Given: Nominatim returns an address with a country name but no country_code
- When: the address is parsed
- Then: the country is blank

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/AddressParseServiceTest.java`, `src/main/java/dev/ted/jittertravel/infrastructure/AddressParseService.java`._

[test: noCountryCodeLeavesTheCountryBlank : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AddressParseServiceTest.java#L90 ]

## S6: Map search links use the country name and put any place name first @v1 [proposed]

- Given: an address at Kolpingstraße 1, Johannesberg, with the picked country DE
- When: its maps link is built
- Then: the link is https://www.google.com/maps/search/Kolpingstra%C3%9Fe+1+Johannesberg+Germany
- and: when the place name Milton Hill House is given for a GB address, the search starts with Milton+Hill+House and ends with United+Kingdom
- and: a country stored as typed text, such as Belgium, is searched as written

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/AddressRendererTest.java`, `src/main/java/dev/ted/jittertravel/infrastructure/AddressRenderer.java`._

[test: aPickedCountryIsSearchedByItsName : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AddressRendererTest.java#L11 ]
[test: aPlaceNameLeadsTheSearch : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AddressRendererTest.java#L19 ]
[test: aStoredNameIsSearchedAsItIs : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AddressRendererTest.java#L29 ]

## S7: A single-segment flight lookup gives one result with local times @v1 [proposed]

- Given: AeroDataBox returns one segment for UA 195 on United Airlines, from MUC at 2026-06-28 11:45+02:00 to SFO at 2026-06-28 14:20-07:00
- When: the response is parsed
- Then: the user does not have to choose a segment
- and: the result has flight number UA195, departure MUC at 2026-06-28 11:45 and arrival SFO at 2026-06-28 14:20, with each time kept as local wall-clock time

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java`, `src/main/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClient.java`._

[test: parsesSampleResponseIntoFlightLookupResult : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java#L58 ]

## S8: Multi-leg flights keep every leg in departure order and offer the whole trip @v1 [proposed]

- Given: AeroDataBox returns two legs for AA 1, SFO to DFW and then DFW to JFK, in any order
- When: the response is parsed
- Then: the user must choose a leg, and the legs are listed SFO first, then DFW
- and: a whole-trip option runs from SFO at 06:00 to JFK at 16:45

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java`, `src/main/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClient.java`._

[test: multiSegmentKeepsEveryLegSeparately : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java#L94 ]
[test: chainingSegmentsAlsoOfferTheWholeTripAsOneOption : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java#L113 ]
[test: segmentsAreOrderedByDepartureTimeRegardlessOfResponseOrder : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java#L149 ]

## S9: Unrelated flights sharing a number offer no whole-trip option @v1 [proposed]

- Given: two AA 1 segments where the second leaves from ORD, not DFW, where the first one landed
- When: the response is parsed
- Then: there is no whole-trip option

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java`, `src/main/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClient.java`._

[test: unrelatedFlightsSharingANumberOfferNoWholeTripOption : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java#L127 ]

## S10: Incomplete segments are dropped without losing the others @v1 [proposed]

- Given: a response where one segment has no departure scheduled time and another is complete
- When: the response is parsed
- Then: only the complete segment, departing DFW, is returned

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java`, `src/main/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClient.java`._

[test: segmentMissingRequiredFieldsIsDroppedWithoutLosingTheOthers : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java#L171 ]

## S11: Empty, blank or malformed responses and a missing API key give no candidates @v1 [proposed]

- Given: a flight lookup response that is an empty array, blank, absent or not JSON
- When: the response is parsed
- Then: there are no candidates
- and: when no API key is set, a lookup for UA195 on 2026-06-28 returns no candidates and does not call the API

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java`, `src/main/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClient.java`._

[test: emptyJsonArrayReturnsNoCandidates : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java#L76 ]
[test: blankResponseReturnsNoCandidates : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java#L83 ]
[test: malformedResponseReturnsNoCandidates : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java#L89 ]
[test: lookupWithBlankApiKeyReturnsNoCandidatesWithoutCallingApi : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/AeroDataBoxClientParseTest.java#L192 ]

## S12: The CSRF token is read on every request so its cookie exists before the view renders @v1 [proposed]

- Given: a request that carries a CSRF token
- When: the request passes through the filter
- Then: the token's value is read exactly once, which writes the cookie
- and: the request continues unchanged to the controller and the view
- and: a request with no CSRF token passes through untouched

_Source: `src/test/java/dev/ted/jittertravel/infrastructure/CsrfCookieFilterTest.java`, `src/main/java/dev/ted/jittertravel/infrastructure/CsrfCookieFilter.java`._

[test: readsTheTokenSoTheCookieIsWrittenBeforeTheViewStarts : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/CsrfCookieFilterTest.java#L50 ]
[test: thenLetsTheRequestContinue : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/CsrfCookieFilterTest.java#L62 ]
[test: aRequestWithNoTokenAttributeIsPassedThroughUntouched : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/infrastructure/CsrfCookieFilterTest.java#L73 ]
