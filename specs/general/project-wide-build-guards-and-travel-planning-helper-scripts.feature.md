# Project-wide build guards and travel-planning helper scripts

_Extracted from `.`._

The build enforces a set of project-wide rules and fails with an explanation when any one is broken. Production code may not read the system clock. A control may not rely on hover alone to look like a control. A read model is built from events alone. Every event that carries a location reaches the schedule-problem and places-used read models. Every projector bean is registered. Every Spring test is tagged. Code may not name project classes by their fully qualified name. Each guard also checks that it is actually scanning something, so it cannot pass while looking at nothing. Next to the build, a standalone script counts Schengen 90/180 days from a backup file.

## S1: Production code that reads the system clock fails the build @v1 [proposed]

- Given: a production class that calls `now()`, calls `now(zone)`, passes `Instant::now` as a reference, calls `System.currentTimeMillis()` or builds a ticking system clock
- When: the architecture guards run
- Then: the build reports a violation saying that time comes from the injected Clock
- and: a class that reads `now(clock)` from an injected Clock is not reported
- and: the single Clock bean factory is exempt, and tests may read the wall clock freely

_Source: `src/test/java/dev/ted/jittertravel/architecture/NoAmbientClockReadsTest.java`, `src/test/java/dev/ted/jittertravel/architecture/ProjectClasses.java`._

[test: productionCodeNeverReadsTheAmbientSystemClock : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/NoAmbientClockReadsTest.java#L72 ]
[test: readingTheInjectedClockIsAllowed : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/NoAmbientClockReadsTest.java#L93 ]

## S2: A link whose only affordance is a hover underline is reported @v1 [proposed]

- Given: a style rule where a selector's base rule is `color: inherit` with no underline, and its `:hover` rule adds `text-decoration: underline`
- When: the stylesheet and every web renderer are scanned
- Then: a violation is reported in the form "<file>: .conf-cfp is `color: inherit` and only underlines on :hover"
- and: a base rule that declares its own colour is not reported
- and: a base rule that is permanently underlined is not reported, whatever its hover does
- and: the scan fails if it reaches 20 or fewer sources or misses `site.css`

_Source: `src/test/java/dev/ted/jittertravel/architecture/HoverIsNeverTheAffordanceTest.java`._

[test: aHoverOnlyUnderlineOnAnInheritedColourIsReported : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/HoverIsNeverTheAffordanceTest.java#L92 ]
[test: aLinkThatDeclaresItsOwnColourIsFine : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/HoverIsNeverTheAffordanceTest.java#L104 ]
[test: aPermanentUnderlineIsFineEvenOnAnInheritedColour : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/HoverIsNeverTheAffordanceTest.java#L115 ]
[test: theScanReachesEveryRendererAndTheStylesheet : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/HoverIsNeverTheAffordanceTest.java#L125 ]
[test: noControlDependsOnHoverToLookLikeOne : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/HoverIsNeverTheAffordanceTest.java#L70 ]

## S3: A projector that names another read model is reported @v1 [proposed]

- Given: a projector source that names another `*Projector` or `*Aggregator`, as a type, as a decapitalized field or through a static call
- When: the projectors in the application package are scanned
- Then: each name is reported with its line, for example "SampleProjector names ScheduleGapProjector (line 2)"
- and: a projector that names only itself, in either spelling, is not reported
- and: names that appear only in comments or string literals are ignored
- and: the scan fails if it finds 20 or fewer projectors

_Source: `src/test/java/dev/ted/jittertravel/architecture/ProjectorsDependOnEventsAloneTest.java`._

[test: aProjectorNamingAnotherReadModelIsReported : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/ProjectorsDependOnEventsAloneTest.java#L71 ]
[test: aProjectorNamingOnlyItselfIsNotAViolation : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/ProjectorsDependOnEventsAloneTest.java#L87 ]
[test: theScanReadsCodeAndIgnoresCommentsAndStringLiterals : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/ProjectorsDependOnEventsAloneTest.java#L99 ]
[test: theScanCoversEveryProjectorInTheApplicationPackage : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/ProjectorsDependOnEventsAloneTest.java#L118 ]
[test: noProjectorNamesAnotherReadModel : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/ProjectorsDependOnEventsAloneTest.java#L50 ]

## S4: Every event that carries a location reaches schedule-problem detection @v1 [proposed]

- Given: a domain event record with an address, airport code or train station component
- When: the schedule gap projector's source has no `case` for that event
- Then: the build fails and names the unhandled events, asking for a presence fact to be added to the location trace
- and: the scan must find at least FlightBooked, TrainBooked, HotelBooked, ConferencePlanned, GatheringPlanned and PrivateEventPlanned

_Source: `src/test/java/dev/ted/jittertravel/architecture/LocatedEventsReachScheduleProblemsTest.java`, `src/test/java/dev/ted/jittertravel/architecture/LocatedEvents.java`._

[test: everyEventCarryingALocationIsHandledByTheProjector : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/LocatedEventsReachScheduleProblemsTest.java#L43 ]
[test: theScanFindsTheLocatedEventsItIsMeantToCover : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/LocatedEventsReachScheduleProblemsTest.java#L55 ]

## S5: Every event that carries an address or station reaches the places-used list @v1 [proposed]

- Given: a domain event record with an address or train station component
- When: the places-used projector's source has no `case` for that event
- Then: the build fails and names the events whose country would never join the short list that the address forms offer first
- and: airport codes alone do not count as a location for this rule
- and: the scan must find at least HotelBooked, TrainBooked, GroundTransferPlanned, ConferencePlanned, GatheringChanged and PrivateEventPlanned

_Source: `src/test/java/dev/ted/jittertravel/architecture/LocatedEventsReachPlacesUsedTest.java`, `src/test/java/dev/ted/jittertravel/architecture/LocatedEvents.java`._

[test: everyEventCarryingAnAddressOrAStationIsHandledByTheProjector : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/LocatedEventsReachPlacesUsedTest.java#L29 ]
[test: theScanFindsTheEventsItIsMeantToCover : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/LocatedEventsReachPlacesUsedTest.java#L37 ]

## S6: Every projector bean must be registered with the bootstrapper @v1 [proposed]

- Given: a bean in the event-sourcing configuration that consumes the event stream
- When: that bean never calls the bootstrapper's register
- Then: the build fails, explaining that the projector would answer every query with an empty read model, for example a blank `/calendar` for every anonymous visitor
- and: the guard fails if fewer than 20 such beans are found, or if PublicCalendarProjector is not among them

_Source: `src/test/java/dev/ted/jittertravel/architecture/EveryProjectorBeanIsRegisteredTest.java`._

[test: everyBeanThatConsumesTheEventStreamIsRegisteredWithTheBootstrapper : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/EveryProjectorBeanIsRegisteredTest.java#L42 ]
[test: theGuardIsLookingAtRealProjectorBeans : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/EveryProjectorBeanIsRegisteredTest.java#L59 ]

## S7: A test that starts a Spring context must be tagged spring @v1 [proposed]

- Given: a test class meta-annotated to extend with the Spring extension
- When: the class does not carry `@Tag("spring")` itself, either alone or among several tags
- Then: the build fails, naming the class, because mutation testing excludes only the `spring` group

_Source: `src/test/java/dev/ted/jittertravel/architecture/SpringTestsAreTaggedTest.java`, `src/test/java/dev/ted/jittertravel/architecture/ProjectClasses.java`._

[test: everySpringTestIsTaggedSpring : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/SpringTestsAreTaggedTest.java#L42 ]

## S8: Code may not refer to a project class by its fully qualified name @v1 [proposed]

- Given: a main or test Java source line that contains `dev.ted.jittertravel.<package>.<Class>` in code
- When: the sources are scanned
- Then: the build fails, listing the file, the line and its text, and asks for an import instead
- and: package, import, comment and Javadoc lines are skipped, as is a name directly preceded by a quote

_Source: `src/test/java/dev/ted/jittertravel/architecture/NoFullyQualifiedClassReferencesTest.java`._

[test: noFullyQualifiedProjectClassReferencesInCodeBodies : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/architecture/NoFullyQualifiedClassReferencesTest.java#L32 ]

## S9: Schengen days are counted from entry-to-exit envelopes @v1 [proposed]

- Given: a backup whose flights or trains that are not cancelled cross the Schengen border
- When: the Schengen day script runs on it
- Then: each entry is paired with the next exit to form an envelope, and every local day inside an envelope counts
- and: an entry is dated by its local arrival date and an exit by its local departure date
- and: two entries in a row, and an exit with no entry before it, are both reported

_Source: `scripts/schengen_days.py`._

## S10: An open envelope and unknown airports are reported rather than guessed @v1 [proposed]

- Given: an entry into Schengen with no exit booked, or a flight using an airport that is missing from the airport table
- When: the Schengen day script runs
- Then: the open envelope is reported and no day after its entry is counted
- and: the unknown airports are listed and treated as outside Schengen

_Source: `scripts/schengen_days.py`._

## S11: The Schengen count reports a floor and a ceiling over a rolling 180 days @v1 [proposed]

- Given: hotels, conferences and gatherings in Schengen countries
- When: the Schengen day script runs, optionally with `--today YYYY-MM-DD`
- Then: the floor counts envelope days plus the days of hotels, gatherings and GOING conferences
- and: the ceiling also counts the days of WATCHING conferences, while OUT conferences are left out
- and: the report shows the count for today, the peak from today to the last day counted, a floor/ceiling count for each month, and every day on which the ceiling exceeds 90

_Source: `scripts/schengen_days.py`._
