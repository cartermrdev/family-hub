# The Wall Display is the primary surface; phones are secondary

Every action the hub offers can be done from the Wall Display, including Household management (Members, Sign-in Codes, devices) and reconnecting iCloud, as if no Member had or used a phone. Phones are a convenience, mostly for pushes and doing things away from home. We chose this over a "wall for glancing and ticking, phone for typing and admin" split because a Member may never sign in on a phone, and the household shouldn't depend on one to run.

## Consequences

- This reverses two earlier decisions: "no Member/device management from the wall" ([#11](https://github.com/cartermrdev/family-hub/issues/11)) and "after a 401, an Adult reconnects from a phone" ([#12](https://github.com/cartermrdev/family-hub/issues/12)).
- `/wall` needs its own in-app touch keyboard and touch date/time/Member pickers (inputs are `inputmode="none"`), because Chromium kiosk on a Pi has no usable keyboard or pickers. Wall forms are full-screen panels with one bottom input tray.
- Because becoming the Acting Member is just an avatar tap, Household-management actions on the wall are gated by an **Adult PIN**. It lasts until the Acting Member clears, 5 wrong tries lock that Adult out for 5 minutes, Adults reset each other's PIN, and the last resort is a developer reset. Everyday actions, including Chore assignment, stay PIN-free.
- First-run Household setup can't assume a phone: the Wall Display must be able to bootstrap the Household on its own.
- A Member who never signs in on a phone gets no pushes; their Messages and Chores appear only on the Wall Display.
