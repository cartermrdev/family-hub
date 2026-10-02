# Can a server read and write the iCloud Family Calendar?

Research for issue [#3](https://github.com/cartermrdev/family-hub/issues/3). Researched 2026-10-01.

## Answer

**Yes.** A backend we host can read, create, edit and delete events on the Household's Family Calendar over CalDAV (`https://caldav.icloud.com/`). It signs in with one Adult's Apple Account and an app-specific password. To that account, the Family Calendar is just another calendar collection in its calendar home. The catch is that Apple has never documented iCloud's CalDAV server: rate limits, sync-token behaviour and how shared calendars look are all known only from community testing and the Apple CalendarServer specs iCloud seems to be based on. Also, the credential stops working whenever that Adult changes their Apple Account password.

## Details

### 1. Authentication: app-specific passwords

- Apple documents app-specific passwords as the supported way to "sign in to your Apple Account in apps made by developers other than Apple" ([Apple 102654](https://support.apple.com/en-us/102654)). CalDAV clients use them as the HTTP Basic password, with the Apple Account email as the username (see the tsdav quickstart, [intro.md](https://github.com/natelindev/tsdav/blob/main/docs/docs/intro.md)).
- They require two-factor authentication on the account. An account can have up to 25 active at once ([Apple 102654](https://support.apple.com/en-us/102654)).
- **Fragility:** "Any time you change or reset your primary Apple Account password, all of your app-specific passwords are revoked automatically." They can also be revoked one at a time, or all together, at account.apple.com, under Sign-In and Security → App-Specific Passwords ([Apple 102654](https://support.apple.com/en-us/102654)).
- 2FA prompts do **not** apply to CalDAV sign-ins that use an app-specific password. Bypassing those prompts is the reason app-specific passwords exist.
- Apple does not document an expiry for app-specific passwords. Community reports describe them lasting until revoked, but that is not guaranteed.
- Advanced Data Protection does not block this. Apple states "Contacts and calendars are built on industry standards (CalDAV and CardDAV) that do not provide built-in support for end-to-end encryption," so Calendars stay outside ADP's end-to-end encryption ([Apple 102651](https://support.apple.com/en-us/102651)).
- One Home Assistant user fixed a failing setup by "checking security key settings" ([HA community thread](https://community.home-assistant.io/t/icloud-calendar-integration/112221?page=3)). Apple's security-keys page does not mention app-specific passwords ([Apple 102637](https://support.apple.com/en-us/102637)), so if the account owner has hardware security keys enabled, we should test that this combination works.

### 2. How the Family Calendar appears over CalDAV

- **Apple-documented:** When someone joins a Family Sharing group, "a shared calendar called *Family* is automatically added to your Calendar app," and members can add events that show up on the other members' devices ([Apple Mac Help](https://support.apple.com/en-ae/guide/mac-help/add-events-to-your-family-sharing-calendar-mhd8f2037623/11.0/mac/11.0); [Apple Personal Safety guide](https://support.apple.com/guide/personal-safety/manage-family-sharing-ips75b3b794f/web)). Family Sharing supports up to 5 additional family members, which covers our 4 Members.
- **Not documented by Apple for iCloud:** what the calendar looks like over CalDAV. Apple's open-source CalendarServer sharing spec, which iCloud appears to be derived from, says that when a sharee accepts a shared calendar, the server creates "a new calendar collection resource in the sharee's calendar home," making it "just another calendar in their calendar home." The sharee's copy has `CS:shared` in its resourcetype, while the owner's copy has `CS:shared-owner`. `CS:invite` and `CS:organizer` identify the owner, and access is either `CS:read` or `CS:read-write` ([caldav-sharing.txt §3, §5.2, §5.5](https://github.com/apple/ccs-calendarserver/blob/master/doc/Extensions/caldav-sharing.txt)). The python-caldav maintainer suspects iCloud "has inherited some code" from CalendarServer ([CHANGELOG](https://github.com/python-caldav/caldav/blob/master/CHANGELOG.md)).
- **Community-observed:** Home Assistant users list the "Family" calendar through `https://caldav.icloud.com` and normal discovery. Name matching is case-sensitive, and users hit problems from that ([HA community thread](https://community.home-assistant.io/t/icloud-calendar-integration/112221?page=3)).
- **Practical implications:**
  - Don't hard-code a URL. Discover the principal, then the `calendar-home-set`, then pick the collection whose display name is "Family" (or whose resourcetype is `CS:shared` / `CS:shared-owner`), and store that collection URL. iCloud URLs contain numeric account IDs and load-balanced hostnames such as `https://p12-caldav.icloud.com/12345/calendars/…`. There is no URL template for them ([python-caldav about.rst](https://github.com/python-caldav/caldav/blob/master/docs/source/about.rst)). The principal can redirect to a different host, and python-caldav rewrites its base URL when that happens ([collection.py `calendar_home_set`](https://github.com/python-caldav/caldav/blob/master/caldav/collection.py)).
  - The Family Sharing organizer owns the Family Calendar. Any other Adult sees it as a shared calendar. Members can add events in Apple's apps, so we expect read-write access over CalDAV, but this is unverified. Writes would come back as `403` if the calendar were read-only.
  - **Attribution:** iCloud records every event the hub writes as written by the account the server signs in as. It has no record of which Member made the change. If Family Hub needs to show who created an event, it must store that itself, for example in an `X-` property or in its own database.
- **Spike to confirm (≈1 hour):** with a real app-specific password, run PROPFIND on the calendar home, confirm the Family collection and its `CS:shared`/`CS:shared-owner` resourcetype and privileges, then PUT, edit and DELETE a test event and watch it appear on an iPhone.

### 3. Reading and writing (standard CalDAV)

- **Create:** `PUT` a new `.ics` resource with `If-None-Match: *`. **Edit:** `PUT` with `If-Match: <etag>` so you don't overwrite someone else's change ([RFC 4791 §5.3.2](https://www.rfc-editor.org/rfc/rfc4791.html#section-5.3.2)). Every calendar object has a strong ETag ([RFC 4791 §5.3.4](https://www.rfc-editor.org/rfc/rfc4791.html#section-5.3.4)). Each resource holds one component type, and UIDs must be unique within the collection ([RFC 4791 §4.1](https://www.rfc-editor.org/rfc/rfc4791.html#section-4.1)). **Delete:** `DELETE`.
- tsdav ships live-iCloud integration tests for `createCalendarObject`, `updateCalendarObject`, `deleteCalendarObject`, `fetchCalendarObjects` with a time range, `syncCalendars` and `smartCollectionSync` ([apple/calendar.test.ts](https://github.com/natelindev/tsdav/blob/main/src/__tests__/integration/apple/calendar.test.ts)). Note that its CI sets `MOCK_FETCH: 'true'` ([ci.yml](https://github.com/natelindev/tsdav/blob/main/.github/workflows/ci.yml)), so a green CI run does not show that these operations still work against live iCloud.
- **Known iCloud quirks** (all community-observed):
  - python-caldav's 2022 iCloud test run had to skip "all tests involving VTODO, VJOURNAL, VFREEBUSY, events with RRULE" ([python-caldav#3](https://github.com/python-caldav/caldav/issues/3)). The commented-out iCloud hint list adds `no_recurring`, `sticky_events`, `get_object_by_uid_is_broken` and `propfind_allprop_failure` ([compatibility_hints.py](https://github.com/python-caldav/caldav/blob/master/caldav/compatibility_hints.py)). **Implication:** don't rely on server-side recurrence expansion (`CALDAV:expand`, [RFC 4791 §9.6.5](https://www.rfc-editor.org/rfc/rfc4791.html#section-9.6.5)) or on time-range searches matching recurring events. Fetch the raw VEVENTs and expand RRULEs on our side.
  - iCloud sometimes duplicates DTSTAMP ([vcal.py](https://github.com/python-caldav/caldav/blob/master/caldav/lib/vcal.py)), returns the wrong principal key in PROPFIND ([davobject.py](https://github.com/python-caldav/caldav/blob/master/caldav/davobject.py)), and sends unreliable content-types ([response.py](https://github.com/python-caldav/caldav/blob/master/caldav/response.py)).
  - python-caldav's docs suggest switching between IPv4 and IPv6 if connections to iCloud fail, and link [issue #393](https://github.com/python-caldav/caldav/issues/393) ([about.rst](https://github.com/python-caldav/caldav/blob/master/docs/source/about.rst)).
  - iCloud-created events carry Apple extensions such as `X-APPLE-STRUCTURED-*` properties, sometimes with trailing whitespace that trips up parsers ([python-caldav code review §2.3](https://github.com/python-caldav/caldav/blob/master/docs/design/FULL_CODE_REVIEW_2026-06.md)). When editing, change the parsed iCalendar and keep properties we don't understand, rather than regenerating the event from scratch.

### 4. Sync tokens and change detection

- **The standard:** RFC 6578 adds the `DAV:sync-collection` REPORT and the `DAV:sync-token` property. A client sends its last token and gets back only the members that changed; deleted members come back as `404 Not Found`. If the token is invalid or expired, the `DAV:valid-sync-token` precondition fails and the client must start over with an empty token. The server may truncate results, signalled with a `507` ([RFC 6578 §3.2, §3.6, §4](https://www.rfc-editor.org/rfc/rfc6578.html)). Apple's `CS:getctag` collection tag is a cheaper "has anything changed?" check ([CalendarServer caldav-ctag.txt](https://github.com/apple/ccs-calendarserver/blob/master/doc/Extensions/caldav-ctag.txt)).
- **iCloud (community-observed):** iCloud returns both `getctag` and `sync-token`. tsdav's documentation uses an iCloud-shaped token as its example (`HwoQEgw…`) ([syncCalendars.md](https://github.com/natelindev/tsdav/blob/main/docs/docs/caldav/syncCalendars.md)), and its Apple integration tests run `smartCollectionSync` with `method: 'webdav'`, which is RFC 6578 ([calendar.test.ts](https://github.com/natelindev/tsdav/blob/main/src/__tests__/integration/apple/calendar.test.ts)). Apple has no documentation on how long tokens stay valid.
- **No push to third parties:** Apple's devices are told about changes over APNs, but there is no documented push channel for third-party CalDAV clients. **Plan on polling:** check the ctag or sync-token every 1–5 minutes, run an incremental `sync-collection` when it changes, and fall back to a full resync on any `valid-sync-token` error. Each poll is a single PROPFIND, which is cheap. The Wall Display can be refreshed straight after the hub's own writes, so it doesn't have to wait for the next poll.

### 5. Rate limits

- **Not documented by Apple.** Developers report `503` / "Rate Limit Exceeded" from iCloud CardDAV at high volume (hundreds of contacts a day), and say Apple publishes no limits and gave no official response ([Apple Developer Forums 722170](https://developer.apple.com/forums/thread/722170)). Apple's own Calendar app has also hit `503` on CalDAV refreshes (2021, [Apple Community 252842966](https://discussions.apple.com/thread/252842966)).
- **Our load:** one Household, one calendar, polling every few minutes, a handful of writes a day. That is far below anything reported as throttled. Respect `Retry-After`, back off exponentially on `503`, and never poll faster than once a minute.

### 6. Terms of service

- The iCloud Terms (last revised September 14, 2026), §V.B(10), forbid using the service to "interfere with or disrupt the Service (including accessing the Service through any automated means, like scripts or web crawlers)" ([iCloud Terms](https://www.apple.com/legal/internet-services/icloud/)). The same document lists calendars among Family Sharing content, and §V.E forbids reselling the service.
- **Reading (not legal advice):** the clause targets automation that interferes with or disrupts the service. A personal server that signs in to the Household's own account, using the credential type Apple provides for "apps made by developers other than Apple" ([Apple 102654](https://support.apple.com/en-us/102654)), and polls gently is the same use pattern as Thunderbird, DAVx⁵ or Home Assistant. Risk is low for a single-household hobby project. It would rise if Family Hub were offered to other households as a hosted service, so revisit this if the project's scope ever changes.

## Risks

| Risk | Likelihood | Mitigation |
| --- | --- | --- |
| Adult changes or resets their Apple Account password → app-specific password revoked → sync stops silently | Medium (happens eventually) | Detect `401`, show a "Calendar disconnected" status on the Wall Display and the phone UI, and give an Adult a simple re-enter-password flow. Store the credential encrypted. |
| Apple changes or tightens iCloud CalDAV (it was never officially documented) | Low–medium | Keep calendar access behind one module boundary. Treat the Family Calendar as the source of truth and keep a local cache so the Wall Display degrades to read-only gracefully. |
| Recurring-event handling on iCloud | High (known quirk) | Expand RRULEs on our side. Edit recurring series by changing the master VEVENT, and handle overridden instances with `RECURRENCE-ID`. Test this thoroughly. |
| Family Calendar not writable through the sharee account | Low (unverified) | Run the spike in §2. If needed, use the Family Sharing organizer's account, since the organizer owns the calendar. |
| Rate limiting (`503`) | Low at our volume | Back off, respect `Retry-After`, poll no faster than every minute. |
| Undocumented sync-token expiry | Low | Always handle `valid-sync-token` failures with a full resync. Fall back to ctag + ETag comparison if needed. |

## Recommended library

- **[tsdav](https://github.com/natelindev/tsdav)** (TypeScript, MIT) is the primary recommendation if the backend is JS/TS. Latest release is v2.3.5 (2026-10-01, [npm](https://www.npmjs.com/package/tsdav)) and it is actively maintained. It runs on Node 18+, Bun, Deno and Cloudflare Workers ([README](https://github.com/natelindev/tsdav)), which suits low-cost hosting. Its docs list Apple CalDAV as supported ([intro.md](https://github.com/natelindev/tsdav/blob/main/docs/docs/intro.md)), it has iCloud integration tests for CRUD and sync, and it implements RFC 6578 sync (`syncCalendars`, `smartCollectionSync`). Pair it with an iCalendar parser and RRULE expander, such as `ical.js`.
- **[python-caldav](https://github.com/python-caldav/caldav)** (Python, GPLv3/Apache-2.0) is the alternative if the backend is Python. It is the most mature CalDAV client. Latest release is v3.3.1 (2026-09-16, [PyPI](https://pypi.org/project/caldav/)), and it supports `objects_by_sync_token` for RFC 6578. It has a lot of iCloud workarounds built into the code. However, its own docs say "Google and iCloud haven't been tested for a long time" and that iCloud "supports CalDAV partly, but there exists no official information about it" ([about.rst](https://github.com/python-caldav/caldav/blob/master/docs/source/about.rst)). The docs also show `features="icloud"`, but the iCloud profile in `compatibility_hints.py` is currently commented out (checked at commit `eb1cf35`).

## Sources

- Apple — App-specific passwords: https://support.apple.com/en-us/102654
- Apple — iCloud data security overview (ADP, CalDAV not E2EE): https://support.apple.com/en-us/102651
- Apple — Security keys for Apple Account: https://support.apple.com/en-us/102637
- Apple — Add events to your Family Sharing calendar (Mac Help): https://support.apple.com/en-ae/guide/mac-help/add-events-to-your-family-sharing-calendar-mhd8f2037623/11.0/mac/11.0
- Apple — Manage Family Sharing (Personal Safety guide): https://support.apple.com/guide/personal-safety/manage-family-sharing-ips75b3b794f/web
- Apple — iCloud Terms and Conditions (rev. 2026-09-14): https://www.apple.com/legal/internet-services/icloud/
- Apple CalendarServer — CalDAV sharing extension: https://github.com/apple/ccs-calendarserver/blob/master/doc/Extensions/caldav-sharing.txt
- Apple CalendarServer — CTag extension: https://github.com/apple/ccs-calendarserver/blob/master/doc/Extensions/caldav-ctag.txt
- RFC 4791 (CalDAV): https://www.rfc-editor.org/rfc/rfc4791.html
- RFC 6578 (WebDAV Collection Synchronization): https://www.rfc-editor.org/rfc/rfc6578.html
- tsdav repo, docs, Apple integration tests, CI: https://github.com/natelindev/tsdav (commit `76bb05f`)
- tsdav on npm: https://www.npmjs.com/package/tsdav
- python-caldav repo, docs, compatibility hints: https://github.com/python-caldav/caldav (commit `eb1cf35`)
- python-caldav on PyPI: https://pypi.org/project/caldav/
- python-caldav iCloud issue #3: https://github.com/python-caldav/caldav/issues/3
- python-caldav issue #393 (iCloud sign-in): https://github.com/python-caldav/caldav/issues/393
- Apple Developer Forums — CardDAV rate limit: https://developer.apple.com/forums/thread/722170
- Apple Community — CalDAV 503: https://discussions.apple.com/thread/252842966
- Home Assistant community — iCloud calendar via CalDAV: https://community.home-assistant.io/t/icloud-calendar-integration/112221?page=3
