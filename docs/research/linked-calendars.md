# Reading Linked Calendars (Google and work Microsoft 365)

Research for issue #4. Researched 2026-10-01 against primary sources (linked inline and listed at the end).

## Answer

**Google:** use the Google Calendar API with a read-only scope. Leave the OAuth project unverified and add the four Members as test users. The catch is that while the project is in "Testing", Google expires refresh tokens after 7 days. Publishing it "In production" without verification fixes that, at the cost of an "unverified app" warning screen and a lifetime cap of 100 users.

**Work Microsoft 365:** assume Microsoft Graph is **blocked** unless the employer's IT admin grants consent. The Microsoft-managed default consent policy, which is also the default for new tenants, explicitly stops end users consenting to `Calendars.Read` and `Calendars.ReadBasic`. The dependable path that needs no admin is the Outlook on the web **"Publish a calendar" ICS link** at the "Can view when I'm busy" level. That link enforces Busy Only at the source. It only works if the tenant's sharing policy allows anonymous publishing.

## Google Calendar

### Read-only scopes
The Calendar API defines these read-only scopes ([Calendar API auth](https://developers.google.com/workspace/calendar/api/auth)):

| Scope | Meaning (Google's wording) |
|---|---|
| `calendar.readonly` | "See and download any calendar you can access using your Calendar" |
| `calendar.events.readonly` | "View events on all your calendars" |
| `calendar.calendarlist.readonly` | "See the list of Google calendars you're subscribed to" |
| `calendar.freebusy` | "View your availability in your calendars" |
| `calendar.events.freebusy` | "See the availability on Google calendars you have access to" |

- `freebusy.query` accepts `calendar.readonly`, `calendar`, `calendar.events.freebusy` or `calendar.freebusy` ([freebusy.query](https://developers.google.com/workspace/calendar/api/v3/reference/freebusy/query)). It "Returns free/busy information for a set of calendars": busy time ranges only, with no titles.
- Sensitivity: Google gives "reading events stored in Google Calendar" as an example of a **sensitive** scope ([sensitive scope verification](https://developers.google.com/identity/protocols/oauth2/production-readiness/sensitive-scope-verification)). **Not confirmed:** I couldn't find a primary source that classifies the two free/busy scopes. Treat them as sensitive until the Cloud Console says otherwise when they're added to the consent screen.
- None of the read-only scopes is "restricted", so a security assessment would never apply. That only matters if the app were ever verified.

### Verification for a personal app
- "If the app is for your personal use (fewer than 100 users), you and your limited number of users can continue using the app without going through verification". Users see the unverified-app screen but can continue ([when verification is not needed](https://support.google.com/cloud/answer/13464323)).
- **Testing** status: limited to 100 listed test users ([publishing status](https://support.google.com/cloud/answer/15549945)). However, a project that is External and in Testing "is issued a refresh token expiring in 7 days, unless the only OAuth scopes requested are a subset of name, email address, and user profile" ([Google OAuth 2.0](https://developers.google.com/identity/protocols/oauth2)). For a wall display that has to keep working unattended, a re-link every 7 days isn't workable.
- **In production, unverified:** Google shows the unverified-app warning. The cap is "100 new users in total", which "applies over the entire lifetime of the project, and it cannot be reset or changed" ([publishing status](https://support.google.com/cloud/answer/15549945)). Four Members is far below that, but every re-grant with a new account uses up the lifetime cap.
- **Not confirmed:** whether an In production, unverified project is free of the 7-day refresh-token expiry. The OAuth doc ties the 7-day rule only to "Testing" status ([Google OAuth 2.0](https://developers.google.com/identity/protocols/oauth2)), so by implication it is. Check this with a real token before relying on it.
- Other refresh-token limits: 100 tokens per account per client. The oldest is silently invalidated. Tokens also expire after 6 months unused, and the user can revoke them ([Google OAuth 2.0](https://developers.google.com/identity/protocols/oauth2)).

### Freshness
- Incremental sync: pass the last `syncToken` on list requests. A `410 Gone` response means do a full re-sync ([sync guide](https://developers.google.com/workspace/calendar/api/guides/sync)).
- Push: `watch` channels need an HTTPS webhook with a valid certificate. Channels expire and have to be replaced, and a notification only says that something changed, so you fetch the changes separately ([push guide](https://developers.google.com/workspace/calendar/api/guides/push)). With a backend we host, polling with a `syncToken` every few minutes is simpler and good enough.

### Busy Only on Google
- Strongest: request only `calendar.freebusy` for a Busy Only calendar. The hub then never receives titles. The cost is a different scope per Linked Calendar, which means a different consent per Visibility.
- Simpler: request `calendar.readonly` and strip titles and details in our backend for Busy Only. Full details then reach our server, so Busy Only becomes a display rule rather than a guarantee at the source.
- ICS fallback: the "Secret address in iCal format" exposes the full calendar. Google warns "Only you should know the Secret Address", it can be reset, and Workspace admins can hide it ([Google Calendar help](https://support.google.com/calendar/answer/37648)). It has no Busy Only mode, so we would have to filter it ourselves.

## Work Outlook / Exchange (Microsoft 365)

### Microsoft Graph permissions
- `Calendars.ReadBasic`: "read events in user calendars, except for properties such as body, attachments, and extensions". `Calendars.Read`: read events. Graph's *permission-level* flag says neither needs admin consent ([permissions reference](https://learn.microsoft.com/en-us/graph/permissions-reference)).
- `getSchedule` (free/busy) is least-privileged at `Calendars.ReadBasic` and is **not supported for personal Microsoft accounts** ([getSchedule](https://learn.microsoft.com/en-us/graph/api/calendar-getschedule?view=graph-rest-1.0)). Its response can still include `subject` and `location` for items the caller can see, so it doesn't strip details by itself.

### Will the work tenant block user consent? Very likely, yes.
- The tenant's consent *policy* overrides the permission's own flag. The Microsoft-managed setting "Let Microsoft manage your consent settings" "is also the default for a new tenant". Under it, end users can consent to delegated permissions **except** a list that includes `Calendars.Read`, `Calendars.ReadBasic`, `Calendars.ReadWrite`, `Calendars.Read.Shared` and the Mail, Contacts, Tasks and EWS/EAS/IMAP/POP permissions ([manage app consent policies](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/manage-app-consent-policies), updated 2026-08-28). This was rolled out to existing tenants on that policy in Oct–Nov 2025 ("Require admin consent for apps accessing Exchange and Teams content", Message Center MC1163922, [mirror](https://mc.merill.net/message/MC1163922)). Users who had already consented keep access.
- The alternative built-in policy, `microsoft-user-default-low`, only allows consent "for apps from verified publishers and apps that are registered in your tenant" ([configure user consent](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-user-consent)). Our app would be neither.
- Publisher verification needs a verified Microsoft AI Cloud Partner Program account, a Partner One ID and a non-`onmicrosoft.com` publisher domain ([publisher verification](https://learn.microsoft.com/en-us/entra/identity-platform/publisher-verification-overview)). That isn't realistic for a hobby app. Even when the tenant allows consent, unverified multitenant apps registered after Nov 2020 get a risky-app warning or are blocked under risk-based step-up consent (same source).
- The only tenants where user consent still works are ones still on the legacy policy, "Allow user consent for apps". Otherwise the Member has to use the admin consent workflow, if it's enabled, and wait for IT to approve ([configure user consent](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-user-consent)). We can't know which applies without trying it in each employer's tenant.

### If Graph is allowed
- Refresh tokens last 90 days and replace themselves on every use. They are revoked on admin action or user revocation, and a confidential-client token survives a password change ([refresh tokens](https://learn.microsoft.com/en-us/entra/identity-platform/refresh-tokens)). Conditional Access policies in the employer tenant can also force re-sign-in. That depends on the tenant and can't be known in advance.
- Change notifications for Outlook events last at most 10,080 minutes (under 7 days) and must be renewed. Microsoft documents event latency as "Unknown" ([subscription resource](https://learn.microsoft.com/en-us/graph/api/resources/subscription?view=graph-rest-1.0)). Polling is simpler.
- Busy Only: use `Calendars.ReadBasic`, which still returns subjects, and strip details in the backend. Busy Only is then our rule, not Microsoft's.

### ICS fallback: Outlook "Publish a calendar"
- How: Outlook on the web > Settings > Calendar > Shared calendars > Publish a calendar. Choose a calendar and detail level, then Publish ([publishing internet calendars](https://support.microsoft.com/en-us/office/introduction-to-publishing-internet-calendars-a25e68d6-695a-41c6-a701-103d44ba151d)).
- Detail levels: **"Can view when I'm busy"**, "Can view titles and locations", "Can view all details". Both HTML and ICS links are produced, and both are read-only ([share your calendar in Outlook on the web](https://support.microsoft.com/en-us/office/share-your-calendar-in-outlook-on-the-web-7ecef8ae-139c-40d9-bae2-a23977ee58d5)).
- Exposure: "Published calendars are viewable by anyone with the link". The URL is an unauthenticated secret ([publishing internet calendars](https://support.microsoft.com/en-us/office/introduction-to-publishing-internet-calendars-a25e68d6-695a-41c6-a701-103d44ba151d)). Store it as a credential.
- Admin control: anonymous publishing is governed by the Exchange sharing policy's `Anonymous` domain entry. Its levels are `CalendarSharingFreeBusySimple` (free/busy hours only), `...Detail` (adds subject and location) and `...Reviewer` (adds body). Admins can restrict or disable it ([Set-SharingPolicy](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-sharingpolicy); [sharing policies](https://learn.microsoft.com/en-us/exchange/sharing/sharing-policies/sharing-policies)). Microsoft's help also warns that publishing "may not be available for your account... depending on your organization settings" ([share your calendar](https://support.microsoft.com/en-us/office/share-your-calendar-in-outlook-on-the-web-7ecef8ae-139c-40d9-bae2-a23977ee58d5)). **Not confirmed in a primary source:** what the default policy sets for `Anonymous`. A third-party source says `CalendarSharingFreeBusyReviewer`.
- Refresh latency: **Microsoft doesn't document it.** Microsoft says only that changes "are synchronized to the web server" and that subscriber sync frequency "depends on the recipient's email provider". The hub is the subscriber, so we choose the poll interval, but the delay before Exchange regenerates the feed is unknown. Measure it during the prototype. RFC 7986 `REFRESH-INTERVAL` ("a suggested minimum interval for polling") is advisory only, if the feed includes it at all ([RFC 7986 §5.7](https://www.rfc-editor.org/rfc/rfc7986#section-5.7)).
- Busy Only: **"Can view when I'm busy" enforces Busy Only at the source.** Titles never leave Microsoft. This is the best privacy property of any path here, and it matches the "work calendars default to Busy Only" rule exactly.

## ICS in general (RFC 5545)
- Feeds are VCALENDAR documents of VEVENTs ([RFC 5545](https://www.rfc-editor.org/rfc/rfc5545)). Two properties matter for us. `TRANSP` (OPAQUE or TRANSPARENT, §3.8.2.7) says whether an event blocks time, so we should skip TRANSPARENT events when showing busy blocks. `CLASS` (PUBLIC, PRIVATE or CONFIDENTIAL, §3.8.1.3) is only advisory. RFC 5545 itself defines no polling or refresh mechanism. The parser must handle RRULE recurrences and VTIMEZONE.
- ICS has no change token: every poll downloads the whole feed. That's fine at household scale.

## Recommendation
1. **Google: Calendar API.** One Google Cloud project, External, **In production but unverified** (to avoid the 7-day token expiry in Testing). Use `calendar.readonly` plus `syncToken` polling. Busy Only is applied in the backend. Accept the warning screen. Fall back to the Secret iCal address only if OAuth is a problem.
2. **Work Microsoft 365: ICS first.** Ask the Member to publish their work calendar at "Can view when I'm busy" and paste the ICS link. Treat Graph `Calendars.ReadBasic` as an optional upgrade, only if a Member's IT approves admin consent.
3. Build one Linked Calendar abstraction with two adapters, **OAuth API** and **ICS URL**, both read-only. Apply Busy Only as a final backend filter on both, even where the source already enforces it.

## Risks
- Google's unverified screen and 100-user lifetime cap. Google could also change its personal-use policy.
- Not confirmed: whether an unverified In production project definitely avoids the 7-day token expiry.
- The employer may forbid or disable anonymous publishing. Publishing a work calendar may also break workplace policy even when it's technically allowed, so each Member should check with their employer.
- An ICS URL is a bearer secret. If it leaks, it exposes whatever level was published until it's reset.
- ICS freshness after an edit is undocumented. Graph event-notification latency is "Unknown".
- Conditional Access or token revocation in a work tenant can break Graph without warning.

## Sources
- https://developers.google.com/workspace/calendar/api/auth
- https://developers.google.com/workspace/calendar/api/v3/reference/freebusy/query
- https://developers.google.com/workspace/calendar/api/guides/sync
- https://developers.google.com/workspace/calendar/api/guides/push
- https://developers.google.com/identity/protocols/oauth2
- https://developers.google.com/identity/protocols/oauth2/production-readiness/sensitive-scope-verification
- https://support.google.com/cloud/answer/13464323
- https://support.google.com/cloud/answer/15549945
- https://support.google.com/calendar/answer/37648
- https://learn.microsoft.com/en-us/graph/permissions-reference
- https://learn.microsoft.com/en-us/graph/api/calendar-getschedule?view=graph-rest-1.0
- https://learn.microsoft.com/en-us/graph/api/resources/subscription?view=graph-rest-1.0
- https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/configure-user-consent
- https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/manage-app-consent-policies
- https://learn.microsoft.com/en-us/entra/identity-platform/publisher-verification-overview
- https://learn.microsoft.com/en-us/entra/identity-platform/refresh-tokens
- https://mc.merill.net/message/MC1163922 (mirror of Microsoft 365 Message Center MC1163922)
- https://support.microsoft.com/en-us/office/introduction-to-publishing-internet-calendars-a25e68d6-695a-41c6-a701-103d44ba151d
- https://support.microsoft.com/en-us/office/share-your-calendar-in-outlook-on-the-web-7ecef8ae-139c-40d9-bae2-a23977ee58d5
- https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-sharingpolicy
- https://learn.microsoft.com/en-us/exchange/sharing/sharing-policies/sharing-policies
- https://www.rfc-editor.org/rfc/rfc5545
- https://www.rfc-editor.org/rfc/rfc7986#section-5.7
