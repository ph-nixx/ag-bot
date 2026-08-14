# Feature Specification: ALGO GYM LeetCode Progress Tracking & Leaderboard

**Feature Branch**: `001-leetcode-submission-tracking`

**Created**: 2026-08-13

**Status**: Draft

**Input**: User description: "ALGO GYM is a discord server for entry level devs trying to excel at technical interviews with an emphasis on verbalizing reasoning efficiently; the server also acts a place for the devs to network; the server requires that members of the serve stay consistent with thier commitment to learning and problem solving; to do this we will use daily leetcode problem submissions and solutions to keep track of the efforts of each memeber in the community so they be judged and ranked; we want access to the submission solution so other members of the community can review there solution for learning purposes and check for any LLM assistance (we dont want LLM assistance)" — refined through follow-up discussion: members register only their LeetCode username (no manual posting or solution collection); the system automatically detects solved problems and computes daily and weekly difficulty-weighted rankings on a leaderboard that refreshes approximately every 5 minutes; LLM-assistance checking is handled separately via live mock interviews and is out of scope here.

## Clarifications

### Session 2026-08-13

- Q: Should the system require any proof that a member actually owns the LeetCode username they register, or is self-reported username entry sufficient? → A: Self-reported entry only for this version (no ownership proof required, beyond the existing "must be a real public profile" check). Ownership verification via a bot-issued token the member temporarily displays on their public LeetCode profile, through a dedicated registration channel, is a planned future enhancement and is out of scope for this feature.
- Q: What should happen when a member tries to register a LeetCode username that another member has already registered? → A: Reject the registration attempt and inform the member the username is already claimed by another member.
- Q: What should the leaderboard and a member's tracked data show once that member leaves the Discord server? → A: A job MUST be run to wipe that member's registration and solve history data from the datastore entirely (not merely hidden from the leaderboard).
- Q: When the external LeetCode data source is unavailable for an extended period, should the leaderboard keep showing the last-known data, or should it indicate something is stale/degraded? → A: The bot MUST display an outage/error indicator on the leaderboard when a refresh fails, rather than silently showing stale data.
- Q: If a registered member's LeetCode profile becomes private after registration, how should the system handle that member going forward? → A: Treat the profile as no longer valid — hide the member from the leaderboard and freeze their score updates until the profile is public again. Additionally, the bot MUST post about this event in a separate, public activity feed channel (distinct from the leaderboard channel).
- Q: Which external libraries/services will this feature depend on for the Discord bot interface and for retrieving LeetCode solve data? → A: discord.py (https://discordpy.readthedocs.io/en/stable/) for the Discord bot framework, and the LeetCode public GraphQL endpoint (`https://leetcode.com/graphql`) queried directly — per the reverse-engineered, partial schema documented in `docs/leetcode-graphql-schema.graphql` — rather than a third-party wrapper API, for retrieving public profile and submission data.
- Q: How should the system handle the fact that `https://leetcode.com/graphql` is an undocumented, unofficial endpoint (no auth, no published rate limits, no stability guarantee) when polling many members every ~5-minute refresh cycle? → A: During development, test against the live endpoint to confirm the polling design can comfortably support up to 250 registered members within a refresh cycle. To reduce brittleness against this undocumented endpoint, the ingestion implementation MUST also minimize both the number of distinct GraphQL query operations and the complexity (field selection depth) of each query used against it.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Register LeetCode Username (Priority: P1)

A new member provides their LeetCode username once so the system can begin tracking their solved problems automatically going forward.

**Why this priority**: Registration is the single enabling action for the entire feature — without it, no data can be collected for a member at all.

**Independent Test**: Can be fully tested by having a member register a valid LeetCode username and verifying the system stores the association and confirms successful registration, independent of polling or leaderboard behavior.

**Acceptance Scenarios**:

1. **Given** a member has not yet registered, **When** they provide a LeetCode username, **Then** the system associates that username with their Discord identity and confirms registration.
2. **Given** a member provides a username that does not correspond to an existing public LeetCode profile, **When** registration is attempted, **Then** the system rejects it and informs the member the username could not be found.
3. **Given** a member is already registered, **When** they provide a new LeetCode username, **Then** the system updates their registered username to the new value.

---

### User Story 2 - Automatic Ingestion of Solved Problems (Priority: P2)

Once registered, a member's publicly solved LeetCode problems are automatically detected and recorded by difficulty, without the member posting anything in the server.

**Why this priority**: This is the core data pipeline that makes tracking and ranking possible. It depends on registration (Story 1) but delivers the "no manual posting" value entirely on its own.

**Independent Test**: Can be fully tested by registering a member, allowing a polling cycle to run, and confirming newly solved problems appear in their history classified as Easy/Medium/Hard with no manual action from the member.

**Acceptance Scenarios**:

1. **Given** a registered member solves a new LeetCode problem, **When** the system's next polling cycle runs, **Then** a record of that solve, including its difficulty, is added to the member's history.
2. **Given** a member already has a specific problem recorded, **When** the system polls again and that problem still appears in their public activity, **Then** it is not recorded a second time or double-counted.
3. **Given** a member's LeetCode profile is temporarily unreachable during a polling cycle, **When** the system retries on a later cycle, **Then** previously recorded solves are unaffected and new solves are captured once the profile becomes reachable again.

---

### User Story 3 - View Daily and Weekly Leaderboards (Priority: P3)

Community members can see how they and others rank, both for the current day and the current week, based on a difficulty-weighted score, on a shared leaderboard.

**Why this priority**: Leaderboard visibility is the payoff that drives engagement and accountability. It depends on ingestion (Story 2) already producing data.

**Independent Test**: Can be fully tested by seeding solve records of varying difficulty across multiple members and verifying the leaderboard displays members ordered correctly by computed score for both the daily and weekly windows.

**Acceptance Scenarios**:

1. **Given** multiple members have solved problems of varying difficulty today, **When** the leaderboard is viewed, **Then** members are ranked by their daily score, computed as (Easy count × 1) + (Medium count × 3) + (Hard count × 9).
2. **Given** multiple members have solved problems of varying difficulty during the current week, **When** the leaderboard is viewed, **Then** members are ranked by their weekly score using the same weighting.
3. **Given** new solves are recorded, **When** the next leaderboard refresh cycle occurs (approximately every 5 minutes), **Then** the displayed daily and weekly rankings update together to reflect the new data.

---

### Edge Cases

- What happens when a member solves the same problem more than once (e.g., resubmits an improved solution) — does it still count only once toward their score?
- How does the system handle the boundary between one day and the next, and between one week and the next, for scoring purposes?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST allow a member to register a single LeetCode username associated with their Discord identity.
- **FR-002**: System MUST verify that a submitted LeetCode username corresponds to an existing public profile before completing registration, and MUST inform the member if it cannot be found. System MUST NOT require proof of ownership of that username beyond this existence check (registration is self-reported).
- **FR-003**: System MUST allow a registered member to update their LeetCode username to a different value.
- **FR-004**: After registration, system MUST automatically detect newly solved problems from the member's public LeetCode activity on a recurring basis, without requiring the member to post anything in the server.
- **FR-005**: System MUST classify each detected solve by difficulty (Easy, Medium, or Hard).
- **FR-006**: System MUST NOT count the same problem more than once toward a member's score, regardless of how many times the member resubmits or re-solves it.
- **FR-007**: System MUST maintain, for each member, separate counts of Easy, Medium, and Hard problems solved within the current day and within the current week.
- **FR-008**: System MUST compute a member's score for a given window as (Easy count × 1) + (Medium count × 3) + (Hard count × 9).
- **FR-009**: System MUST rank members against one another using their computed score, separately for the daily window and the weekly window.
- **FR-010**: System MUST display the daily and weekly rankings as a leaderboard in a designated, community-visible location.
- **FR-011**: System MUST refresh the leaderboard's underlying data on a recurring, automatic basis approximately every 5 minutes, updating both the daily and weekly views together.
- **FR-012**: System MUST preserve historical per-difficulty solve records for active members so that daily and weekly scores can be recomputed or audited at any time.
- **FR-013**: System MUST NOT require members to submit, post, or share their solution content as part of this feature.
- **FR-014**: System MUST reject a registration attempt for a LeetCode username that is already registered to a different member, and MUST inform the member that the username is already claimed.
- **FR-015**: System MUST run a data-wipe job that permanently removes a member's registration and solve history from the datastore after they leave the Discord server, removing them from all leaderboards.
- **FR-016**: System MUST visibly indicate on the leaderboard when a data refresh has failed due to the external data source being unavailable, rather than silently continuing to display stale data as current.
- **FR-017**: System MUST treat a registered member's LeetCode profile becoming inaccessible (e.g., made private) the same as an invalid profile: the member MUST be hidden from the leaderboard and their score updates frozen until the profile becomes accessible again.
- **FR-018**: System MUST post a notification in a public activity feed channel, separate from the leaderboard channel, whenever a registered member's profile becomes inaccessible in this way.

### Key Entities

- **Member**: A registered community participant. Attributes include a Discord identity and a linked LeetCode username.
- **Solve Record**: A single detected instance of a member solving a specific LeetCode problem for the first time. Attributes include the member, the problem identifier, its difficulty, and the timestamp it was detected.
- **Daily/Weekly Score** *(derived)*: A member's Easy/Medium/Hard counts and computed weighted score for the current day or current week.
- **Leaderboard**: The ranked ordering of members by computed score, maintained separately for the daily and weekly windows and displayed to the community.
- **Activity Feed**: A public channel, separate from the leaderboard, where the system posts notable status-change events (e.g., a member's LeetCode profile becoming inaccessible).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new member can complete registration by providing their LeetCode username in under 1 minute.
- **SC-002**: A newly solved problem is reflected in the member's daily and weekly score within one leaderboard refresh cycle (approximately 5 minutes) of the system detecting it.
- **SC-003**: 100% of scores displayed on the leaderboard are accurate relative to the defined weighting formula and each member's recorded solve history, verifiable by an organizer at any time.
- **SC-004**: No solved problem is ever counted more than once toward a member's score.
- **SC-005**: The daily and weekly leaderboards are visible to all community members at all times without any member needing to run a command or take action to view them.
- **SC-006**: The polling/ingestion design is validated during development to comfortably support at least 250 registered members completing a full refresh cycle against the live LeetCode GraphQL endpoint without triggering rate-limiting or blocking.

## Assumptions

- The system retrieves publicly available LeetCode profile and submission data by querying LeetCode's own GraphQL endpoint (`https://leetcode.com/graphql`) directly, using the reverse-engineered, partial schema in `docs/leetcode-graphql-schema.graphql`; this endpoint is undocumented and outside the community's direct control, so extended outages, undocumented rate limits, or unannounced schema changes may delay leaderboard updates or require ingestion logic changes.
- The Discord bot interface is built with discord.py (https://discordpy.readthedocs.io/en/stable/).
- Because `https://leetcode.com/graphql` is undocumented and unofficial, the ingestion implementation is expected to minimize the number of distinct GraphQL query operations and the field-selection complexity of each query, to reduce exposure to unannounced schema changes or rate-limiting; the 250-member scale target (SC-006) is confirmed by testing against the live endpoint during development rather than by a published capacity guarantee from LeetCode.
- A "day" and a "week" are each defined relative to a single, community-wide reference timezone rather than each member's local timezone.
- A "week" begins on a fixed weekday (e.g., Monday) common to all members, forming a calendar-week window rather than a rolling 7-day window.
- Only problems marked as solved (accepted) on a member's public LeetCode profile are eligible to be recorded; attempted-but-unsolved problems are not tracked.
- LLM-assistance verification is explicitly out of scope for this feature; it is addressed separately through live mock interviews, not through anything this feature collects or displays.
- Each LeetCode username may only be registered to one Discord member at a time.
- Community organizers (existing Discord moderators/admins) are responsible for resolving data disputes not otherwise covered by this feature's automated rules; this role is not created by this feature.
- Token-based ownership verification (member displays a bot-issued token on their public LeetCode profile via a dedicated registration channel) is a planned future enhancement beyond this feature's scope; this version relies on self-reported usernames only.
