# Analyst note: new local administrator account

## Summary

- Alerts: rule 60109 (level 8, account created), then rule 60154 (level 12, Administrators Group Changed), plus related account and group alerts (60110, 60160, 60170).
- Host: win-victim.
- Times (UTC): account created about 12:03:13, added to Administrators about 12:03:27.

## What happened

The local account `labtest2` was created and, 14 seconds later, added to the local Administrators group. The actor in the alert is `labadmin` (`subjectUserName`). The group is `Administrators` (`targetUserName`). The new account appears in the alert as a member SID.

## How I confirmed it

- Read the alert list and the fields of the 60154 alert.
- Ran `net localgroup administrators` in the VM: the members were `Administrator`, `labadmin` and `labtest2`. Then deleted the test account with `net user labtest2 /delete`.

## Verdict

True positive, benign. I created the account myself with two commands as a test on my own lab VM. Without that knowledge this pattern (new account, then made administrator seconds later) is a classic persistence step, so it would need an answer, not a closed ticket.

## What I would do next if this were real

1. Find out whether `labadmin` was expected to do this: who was logged in, from where, and was there a change request.
2. Check logons for the new account (event 4624) and what it did afterwards.
3. Check other activity by `labadmin` around the same time.
4. If not authorised: disable the new account, remove it from Administrators, and escalate.

## Evidence

- `screenshots/12-new-admin-alert-list.png`
- `screenshots/13-alert-60154-rule.png` (blur any SIDs)
