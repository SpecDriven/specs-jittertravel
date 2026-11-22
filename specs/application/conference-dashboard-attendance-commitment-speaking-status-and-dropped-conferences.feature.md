# Conference dashboard: attendance commitment, speaking status and dropped conferences

_Extracted from `src/main/java/dev/ted/jittertravel/application`._

The owner's /conferences dashboard lists planned conferences. Attendance commitment and speaking status are worked out from each conference's events. Talk acceptance, rejection and invitation change attendance according to fixed rules. Declined conferences stay in the list but are hidden by default, and conferences cancelled by the organizers disappear. The list can be filtered to upcoming conferences, and the time zone shown is chosen from a parameter or cookie that falls back to a default.

## S1: A planned conference appears on the list as watched @v1 [proposed]

- Given: a conference is planned with a name, venue, address and start and end dates
- When: the dashboard lists all conferences
- Then: the conference appears with its name and venue city
- and: its commitment is WATCHING, its speaking status is NOT_SPEAKING and it has no CFP deadline recorded

_Source: `src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/ConferenceProjector.java`._

[test: projectorCreatesViewFromEvents : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L47 ]
[test: aPlannedConferenceStartsOutMerelyWatched : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L178 ]
[test: aPlannedConferenceHasNoCfpDeadlineUntilOneIsRecorded : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L244 ]
[test: aMerelyPlannedConferenceRecordsNoSpeakingEvidence : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L463 ]
[test: aPlannedConferenceIsWatchedAndSaysNothingAboutSpeaking : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java#L26 ]

## S2: Conferences are listed by start date, earliest first @v1 [proposed]

- Given: a conference starting on 1 July and one starting on 28 June are planned, in that order
- When: the dashboard lists all conferences
- Then: the 28 June conference comes before the 1 July conference

_Source: `src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/ConferenceProjector.java`._

[test: projectedViewsAreSortedAscendingByStartDate : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L74 ]

## S3: The upcoming filter keeps a conference until its last day ends in the venue's zone @v1 [proposed]

- Given: one conference that started yesterday and ends tomorrow, and one that ended last week, both in a venue zone different from the server's
- When: the dashboard lists FUTURE conferences at the current moment
- Then: only the conference in progress is listed
- and: listing ALL conferences shows both

_Source: `src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/ConferenceView.java`._

[test: futureFilterKeepsInProgressConferenceButDropsFinishedOne : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L107 ]

## S4: Declining a conference keeps it, but hides it unless dropped conferences are shown @v1 [proposed]

- Given: a planned conference, whether watched or already confirmed
- When: attendance is declined
- Then: the conference is missing from the default list (dropped=hide)
- and: with dropped=show it is listed with commitment NOT_GOING
- and: the dropped count under the current time filter includes it

_Source: `src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/ConferenceProjector.java`, `src/main/java/dev/ted/jittertravel/application/DroppedView.java`._

[test: decliningAttendanceKeepsTheConferenceAsDropped : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L129 ]
[test: aConfirmedConferenceCanStillBeDeclined : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L229 ]
[test: droppedCountCountsTheConferencesTheDefaultViewLeavesOut : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L624 ]
[test: droppedCountObeysTheTimeFilter : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L646 ]
[test: decliningDropsTheConferenceWhateverTheTalkDid : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java#L140 ]

## S5: A conference cancelled by the organizers is gone @v1 [proposed]

- Given: a planned conference
- When: the organizers cancel it
- Then: it is missing from the list even with dropped=show
- and: its detail page finds nothing

_Source: `src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/ConferenceProjector.java`._

[test: aConferenceCancelledByOrganizersIsGoneEvenWhenDroppedOnesAreAskedFor : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L164 ]
[test: aCancelledConferenceHasNoDetailView : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L763 ]

## S6: Confirming attendance changes only the commitment @v1 [proposed]

- Given: a planned conference
- When: attendance is confirmed with basis SPEAKING_ACCEPTED
- Then: its commitment is GOING and it counts as speaking
- and: every other field of the row stays as it was
- and: confirming attendance for a conference that was never planned adds nothing to the list

_Source: `src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java`, `src/main/java/dev/ted/jittertravel/application/ConferenceProjector.java`._

[test: confirmingAttendanceLeavesTheRestOfTheViewAlone : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L203 ]
[test: confirmingAttendanceTurnsTheViewIntoGoing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L189 ]
[test: confirmingAttendanceForAnUnknownConferenceAddsNothing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L220 ]
[test: speakingIsDerivedFromTheBasisAndTheBasisItselfNeverReachesTheView : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L438 ]

## S7: An accepted talk commits attendance on its own @v1 [proposed]

- Given: a planned conference with a submitted talk
- When: the talk is accepted
- Then: the commitment is GOING and the conference counts as speaking, with no attendance confirmation needed
- and: while the talk is only submitted, it does not count as speaking

_Source: `src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java`, `src/main/java/dev/ted/jittertravel/application/ConferenceState.java`._

[test: anAcceptanceCommitsAttendanceAndMeansSpeaking : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java#L37 ]
[test: aSubmittedTalkIsNotYetSpeaking : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java#L46 ]
[test: anAcceptedTalkCommitsAttendanceWithNoConfirmationEvent : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L479 ]

## S8: A rejection drops a conference only when acceptance is required to attend @v1 [proposed]

- Given: a submitted talk at a conference
- When: the talk is rejected
- Then: an ACCEPTANCE_REQUIRED conference is dropped, at NOT_GOING
- and: a CALL_FOR_PAPERS conference stays WATCHING with speaking status REJECTED

_Source: `src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java`, `src/main/java/dev/ted/jittertravel/application/ConferenceState.java`._

[test: aRejectionDropsAConferenceWhereAcceptanceWasRequired : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java#L119 ]
[test: aRejectionLeavesACallForPapersConferenceWatched : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java#L131 ]
[test: aRejectionDropsAConferenceWhereAcceptanceWasRequired : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L542 ]
[test: aRejectionLeavesACallForPapersConferenceStillWatched : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L565 ]

## S9: An unanswered invitation commits nothing and does not count as speaking @v1 [proposed]

- Given: a planned conference
- When: Ted is invited to speak
- Then: the commitment stays WATCHING and the conference does not count as speaking
- and: confirming with basis SPEAKING_INVITED makes it GOING and speaking
- and: confirming with basis TICKET_PURCHASED makes it GOING but not speaking

_Source: `src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java`, `src/main/java/dev/ted/jittertravel/application/ConferenceState.java`._

[test: anUnansweredInvitationIsNotSpeaking : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java#L56 ]
[test: anInvitationAcceptedAsSpeakingIsSpeaking : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java#L68 ]
[test: anInvitationTakenUpAsAPlainTicketIsNotSpeaking : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java#L78 ]
[test: anInvitationCommitsNothingUntilItIsAccepted : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L501 ]
[test: anInvitationTakenUpAsAPlainTicketIsNotSpeaking : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L525 ]

## S10: Talk events decide speaking over the confirmation basis @v1 [proposed]

- Given: a conference confirmed with no talk events
- When: speaking is worked out
- Then: basis SPEAKING_ACCEPTED counts as speaking and TICKET_PURCHASED does not
- and: if a talk is later submitted and rejected, the conference no longer counts as speaking, whatever the basis said
- and: withdrawing an accepted talk leaves the conference GOING but not speaking, and not dropped

_Source: `src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java`, `src/main/java/dev/ted/jittertravel/application/ConferenceState.java`._

[test: withNoSubmissionEventsTheBasisDecides : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java#L91 ]
[test: theStreamOverridesTheBasisWhenTheyDisagree : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java#L98 ]
[test: withdrawingAnAcceptedTalkLeavesHimGoingButNotSpeaking : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceStateTest.java#L109 ]
[test: theSubmissionStreamOverridesTheBasisWhenTheyDisagree : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L603 ]
[test: withdrawingAnAcceptedTalkLeavesHimGoing : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L582 ]
[test: speakingIsDerivedFromTheBasisAndTheBasisItselfNeverReachesTheView : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ConferenceProjectorTest.java#L438 ]

## S11: Venue event subtitles show the venue, then the city, then the time range @v1 [proposed]

- Given: an event at venue "Skills Matter" in London, GB, from 18:00 to 21:00
- When: its subtitle lines are built
- Then: the lines are "Skills Matter", "London, United Kingdom", then the start–end range
- and: a blank venue leaves out the venue line
- and: a blank country shows only "London"

_Source: `src/test/java/dev/ted/jittertravel/application/EventCalendarSubtitleTest.java`, `src/main/java/dev/ted/jittertravel/application/EventCalendarSubtitle.java`._

[test: venueThenCityCountryThenTimeRange : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/EventCalendarSubtitleTest.java#L22 ]
[test: blankVenueOmitsTheVenueLine : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/EventCalendarSubtitleTest.java#L33 ]
[test: blankCountryShowsOnlyTheCity : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/EventCalendarSubtitleTest.java#L43 ]

## S12: Display zone and viewer zone fall back to safe defaults @v1 [proposed]

- Given: a ?tz= parameter, or a viewer's zone cookie
- When: the parameter is missing or not recognized, such as "utc"
- Then: the display zone is entry
- and: the parameter is matched without regard to case, so "BrOwSeR" means browser
- and: a valid IANA cookie such as "Europe/Berlin" is used as given, while a missing, blank or unknown cookie such as "Mars/Olympus_Mons" falls back to the configured zone

_Source: `src/test/java/dev/ted/jittertravel/application/DisplayZoneTest.java`, `src/main/java/dev/ted/jittertravel/application/DisplayZone.java`, `src/test/java/dev/ted/jittertravel/application/ViewerTodayZoneTest.java`._

[test: absentParamFallsBackToEntryLocal : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/DisplayZoneTest.java#L10 ]
[test: unrecognizedParamFallsBackToEntryLocal : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/DisplayZoneTest.java#L16 ]
[test: paramIsCaseInsensitive : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/DisplayZoneTest.java#L24 ]
[test: resolvesAValidIanaZoneFromTheCookie : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ViewerTodayZoneTest.java#L15 ]
[test: fallsBackWhenCookieIsAbsent : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ViewerTodayZoneTest.java#L21 ]
[test: fallsBackWhenCookieValueIsNotARecognizedZone : https://github.com/jitterted/jittertravel/blob/main/src/test/java/dev/ted/jittertravel/application/ViewerTodayZoneTest.java#L33 ]
