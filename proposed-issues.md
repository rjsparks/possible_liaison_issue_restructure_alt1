# Proposed Issues for the New Liaison Tool Repository

This document proposes issues for a new repository that consolidates and restructures
work tracked in `ietf-tools/liaison-tooling-2025a` (GitHub issues #1–#45) and open issues
carrying the `component: liaison/` label in `ietf/datatracker`. It also introduces new
issues from analysis of the current codebase and RFC 4053bis compliance gaps.

Each issue is attributed to its upstream origins. All decisions have been made and all
issues are ready to implement.

---

## Proposed Labels

| Label | Description |
|-------|-------------|
| `workflow` | State machine and approval workflow changes |
| `roles` | Authorization and permission changes |
| `data-model` | Database schema / model changes |
| `email` | Email sending and notification changes |
| `usability` | UI and UX improvements |
| `attachments` | Attachment upload, deletion, storage |
| `incoming` | Specific to incoming liaison statements |
| `outgoing` | Specific to outgoing liaison statements |
| `rfc-compliance` | Required or implied by RFC 4053bis |
| `proposed-resolution` | Issue has a proposed resolution that needs sign-off before coding |
| `bug` | Bug in current implementation |
| `blocked` | Cannot be started until another issue is resolved |

---

## Proposed Milestones

- **M1 – Workflow redesign**: State machine, approval gates, auto-posting
- **M2 – Roles and permissions**: Authorization model changes
- **M3 – Data model**: Schema changes, contact fields, purpose values
- **M4 – Email and notifications**: All email-related changes
- **M5 – Usability**: UI improvements, drafts, action-taken, attachments

---

## Issues

---

### ISSUE-1: Remove "Obtained prior approval" bypass; always route outgoing statements through the approval workflow

**Labels:** `workflow`, `outgoing`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a #1; RFC 4053bis §4.1

#### Background

`OutgoingLiaisonForm` (forms.py:505) includes an `approved = forms.BooleanField` field
labelled "Obtained prior approval." When checked, `is_approved()` returns `True` and the
form's `save()` method immediately calls `change_state(state_id='posted')`, bypassing the
entire approval workflow. RFC 4053bis §4.1 requires AD approval for all outgoing statements
with no exceptions. The proposed workflow in `liaison-tooling-2025a/README.md` likewise
shows no bypass path.

#### Requirements

- Remove the `approved` BooleanField from `OutgoingLiaisonForm`.
- Remove or replace the `is_approved()` override in `OutgoingLiaisonForm` (forms.py:511–512).
- All outgoing statements must enter `state=pending` after submission, regardless of submitter.
- Update `STATE_EVENT_MAPPING` and related tests accordingly.

#### Acceptance Criteria

- [ ] `OutgoingLiaisonForm` has no `approved` field.
- [ ] Submitting any outgoing statement results in `state=pending`.
- [ ] No remaining code path allows an outgoing statement to skip the pending state.
- [ ] Existing tests updated; regression test added for the removed bypass.

---

### ISSUE-2: Redesign state machine to support new workflow

**Labels:** `workflow`, `data-model`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a README; liaison-tooling-2025a #34

#### Background

The current state machine is `pending → approved → posted`, with a `dead` state (reversible
via "resurrect"). The new workflow introduces three sequential gates before posting for
outgoing statements. The `dead` state is retained and enhanced (see ISSUE-26).

No new states are introduced. The existing `LiaisonStatementState` rows are reused;
`STATE_EVENT_MAPPING` in models.py needs updating to cover the new transitions.

#### Requirements

- Retain the existing `dead` state; do not introduce a `not-approved` state.
- Update `STATE_EVENT_MAPPING` in models.py to cover all valid transitions in the new
  workflow, including `pending → dead` (with mandatory reason comment) and
  `dead → pending` (resurrection).
- The `approved` event represents AD/Chair approval only (gate 1). Per-group liaison
  manager reviews are recorded separately as `manager-reviewed` events (one per `to_group`,
  gate 2) and the coordinator final review as a `coordinator-reviewed` event (gate 3).
  See ISSUE-4. Auto-posting (ISSUE-3) requires all three gates; the `approved` event alone
  must not trigger posting.
- No new `LiaisonStatementState` database rows are required.

#### Acceptance Criteria

- [ ] State transitions match the agreed workflow.
- [ ] `STATE_EVENT_MAPPING` is complete and has no missing transition keys.
- [ ] The Django admin correctly reflects available states.

---

### ISSUE-3: Auto-post when all required sign-offs are obtained; attribute posting to (System) user

**Labels:** `workflow`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a README

#### Background

The README states: "Once approval and review are acquired (whichever occurs last), the
liaison statement will be immediately posted, with the posting attributed to the `(System)`
user." Currently, posting requires an explicit human action and is attributed to the
approving person.

The "(System)" person is the datatracker's existing system `Person` record used for
automated events.

#### Requirements

- When all required gates are satisfied, automatically transition the statement to `posted`:
  - **Outgoing**: AD/Chair approval (`approved` event) **and** one `manager-reviewed` event
    per `to_group` **and** final coordinator review (`coordinator-reviewed` event).
  - **Incoming**: a single `reviewed` event from the Liaison Manager (or Coordinator proxy).
- The `LiaisonStatementEvent` for the `posted` transition must have `by=system_person`.
- If the statement has the send-by-email flag set, trigger the email at this moment.

#### Acceptance Criteria

- [ ] Recording the final required sign-off immediately posts the statement.
- [ ] Posted event's `by` field is the system `Person` record.
- [ ] Email is dispatched on auto-posting when the send-by-email flag is set.
- [ ] No separate "post" button action is required after sign-offs are complete.

---

### ISSUE-4: Implement sequential review gates for outgoing statements: per-group liaison manager review followed by coordinator final review

**Labels:** `workflow`, `roles`, `outgoing`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a README; liaison-tooling-2025a #41

#### Background

Outgoing statements require three sequential gates before auto-posting (ISSUE-3):

1. **AD/Chair approval** (ISSUE-7) — first gate; required per RFC 4053bis §4.1.
2. **Per-group liaison manager review** — second gate, unlocked only after AD approval.
   One review is required per `to_group`; reviews across groups proceed in parallel.
3. **IAB Liaison Coordinator final review** — third gate, required after all per-group
   manager reviews are complete. A single coordinator review covers all `to_groups`.

Currently there are no liaison manager or coordinator review steps.

#### Requirements

**Notifications at "mark ready for review":**
- Notify the liaison manager assigned to each `to_group`. If no liaison manager is assigned
  to a `to_group`, notify IAB Liaison Coordinators instead (they will proxy for that group).
- Notify the responsible ADs / Chair as applicable (ISSUE-7, ISSUE-18).

**Gate 2 — per-group liaison manager review:**
- Record a `manager-reviewed` event in `LiaisonStatementEvent` for each `to_group`.
- The `manager-reviewed` action is only available after the `approved` event exists on
  the statement; the UI must not offer it until gate 1 is satisfied.
- A Liaison Manager may only record the `manager-reviewed` event for `to_groups` to which
  they are assigned.
- IAB Liaison Coordinators and Secretariat may record a `manager-reviewed` event as proxy
  for a `to_group` with no assigned liaison manager; a mandatory comment identifying the
  group being covered is required.
- When AD approval is obtained, notify all liaison managers (and coordinators standing in
  for groups with no LM) that the gate 2 review window is now open.

**Gate 3 — coordinator final review:**
- Record a `coordinator-reviewed` event in `LiaisonStatementEvent`.
- The `coordinator-reviewed` action is only available after all required `manager-reviewed`
  events exist; the UI must not offer it until all gate 2 reviews are complete.
- Any single IAB Liaison Coordinator completing the final review is sufficient.
- When all per-group manager reviews are complete, notify IAB Liaison Coordinators that
  the gate 3 review window is now open.
- A coordinator who performed proxy manager reviews for gate 2 must still separately
  complete the gate 3 final review; the two actions are distinct.

**Access views:**
- Coordinators can access a view of all statements awaiting their action (gate 2 proxy
  reviews or gate 3 final review).
- Liaison Managers can access a view of all statements awaiting their review.

#### Acceptance Criteria

- [ ] `manager-reviewed` event type exists; one event per `to_group` is recorded for gate 2.
- [ ] `coordinator-reviewed` event type exists; one event per statement is recorded for gate 3.
- [ ] Gate 2 actions are unavailable until the `approved` event exists (gate 1 complete).
- [ ] Gate 3 action is unavailable until all required `manager-reviewed` events are present.
- [ ] Notification emails sent to LMs (or Coordinators for groups with no LM) at "ready for
      review" time.
- [ ] Notification emails sent to LMs (and proxy Coordinators) when AD approval is obtained.
- [ ] Notification email sent to IAB Liaison Coordinators when all per-group LM reviews
      are complete.
- [ ] Auto-post logic checks all three gates before posting (ISSUE-3).
- [ ] Pending-review lists available in the UI for both Liaison Managers and Coordinators.

---

### ISSUE-5: Post incoming statements after Liaison Manager entry, review, and approval

**Labels:** `workflow`, `roles`, `incoming`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a README
**Decision:** Incoming statements post immediately when a Liaison Manager performs the
review action. If no Liaison Manager is assigned to the sending SDO, an IAB Liaison
Coordinator enters and reviews the statement (also triggering immediate post).

#### Background

`IncomingLiaisonForm.is_approved()` (forms.py:471) always returns `True`, so every incoming
statement is auto-posted on submission with no human review. This is wrong in the opposite
direction: the old auto-post required zero human involvement, but the new model requires
exactly one — either a Liaison Manager or, when none is assigned to the SDO, an IAB
Liaison Coordinator.

When a Liaison Manager is assigned to the sending SDO, that manager enters and reviews the
statement. When no Liaison Manager is assigned, an IAB Liaison Coordinator enters and
reviews instead. Both roles trigger immediate posting. This is a process decision, not a
tool decision: the tool permits both Liaison Managers and IAB Liaison Coordinators to
perform the review/post action on incoming statements.

#### Requirements

- Remove the `is_approved()` override so incoming statements land in `state=pending` after
  submission.
- When a **Liaison Manager** performs the review action on a pending incoming statement,
  the statement is **immediately posted**.
- When an **IAB Liaison Coordinator** performs the review action on a pending incoming
  statement, the statement is also **immediately posted**.
- No AD approval step is required or solicited for incoming statements.
- The authorized entry roles are: Liaison Manager, IAB Liaison Coordinator, Secretariat.
  Authorized Individual is explicitly excluded (see ISSUE-29).
- The post event is attributed to `(System)` (per ISSUE-3).

#### Acceptance Criteria

- [ ] `IncomingLiaisonForm.is_approved()` no longer unconditionally returns `True`.
- [ ] Submitted incoming statements have `state=pending`.
- [ ] Liaison Manager review triggers immediate post, attributed to `(System)`.
- [ ] IAB Liaison Coordinator review also triggers immediate post, attributed to `(System)`.
- [ ] No AD approval is required or solicited.
- [ ] Authorized Individuals cannot access the incoming liaison creation form (ISSUE-29).

---

### ISSUE-6: Capture the submitter's role when creating a liaison statement

**Labels:** `usability`, `roles`
**Priority:** Medium
**Milestone:** M2
**Origins:** liaison-tooling-2025a #37

#### Background

ietf-tools/liaison-tooling-2025a#37 requests that "the role the person is acting under when
entering the statement will be captured." A person may simultaneously be an AD, a WG Chair,
and a Liaison Manager; recording which capacity they are acting in is needed for the
statement record and for determining applicable approval requirements.

#### Requirements

- During statement creation, the submitter selects the role they are acting in from a
  dropdown.
- Choices are limited to roles the user actually holds relative to the selected `from_groups`
  (derived from `internal_groups_for_person()`).
- The selected role is stored on the statement or as a `submitted` event attribute.
- The stored role is shown in the statement detail and history views.

#### Acceptance Criteria

- [ ] Submission form includes a role selector constrained to the user's actual roles.
- [ ] The selected role is persisted and visible in the statement history.
- [ ] If only one role is applicable, it is pre-selected and the dropdown is hidden.

---

### ISSUE-7: Implement the approval authority model for outgoing statements

**Labels:** `roles`, `outgoing`
**Priority:** High
**Milestone:** M2
**Origins:** liaison-tooling-2025a #2, #41; RFC 4053bis §4.1; ietf/datatracker #9457

#### Background

RFC 4053bis §4.1 defines approval authority for outgoing liaison statements:

> "All liaison statements sent by any group in the IETF need AD approval… This does not
> include statements sent by the IAB, which require IAB approval. Statements sent from an
> area… need approval by at least one of the responsible ADs. Statements sent by the IETF
> or IESG require IETF Chair approval."

**Approver by from_group:**

| From group | Required approver |
|------------|------------------|
| WG or Area | At least one responsible AD |
| IETF / IESG | IETF Chair |
| IAB | IAB Chair (acting for the IAB as a body) |

IAB Liaison Coordinators and Liaison Managers are not approvers. Coordinators have a
separate parallel review gate (ISSUE-4); liaison managers are consulted but do not approve.

IAB Liaison Coordinators and Secretariat may record approval in the tool on behalf of an
AD or Chair when that approval was obtained out of band; a mandatory comment naming the
actual approver is required.

Current relevant code: `Group.liaison_approvers()` (group models),
`approvable_liaison_statements()` (utils.py:26–42), and the approval view.

Note: ietf/datatracker#9457 documents a bug in the current `approver_emails()` method
where set intersection is performed on `Role` objects rather than `Person` objects. Because
each group has a distinct `Role` instance for the same person, the intersection returns
empty and approval notification emails are never sent. This issue resolves that bug.

#### Requirements

- Update `approvable_liaison_statements()` in utils.py to return statements accessible to:
  the actual approvers per from_group (per the table above), plus IAB Liaison Coordinators
  and Secretariat.
- Update the approval view/action to support two paths:
  - **Direct approval**: performed by the actual AD/IETF Chair/IAB Chair. No comment
    required (though one may be added).
  - **Proxy approval**: performed by an IAB Liaison Coordinator or Secretariat recording
    an out-of-band approval. A comment is required; it must name the person who actually
    approved. Store the comment as the `desc` field on the resulting `LiaisonStatementEvent`.
- Update notification emails (ISSUE-18) to target the right approvers.

#### Acceptance Criteria

- [ ] Actual approvers (AD/Chair per from_group) can perform the approval action directly.
- [ ] IAB Liaison Coordinators and Secretariat can perform the approval action as a proxy.
- [ ] Proxy approval is rejected if the comment field is empty.
- [ ] The approval event's `by` field records who performed the tool action; the `desc`
      field records the mandatory comment (naming the actual approver for proxy cases).
- [ ] All other roles cannot perform the approval action.
- [ ] `approvable_liaison_statements()` returns only statements accessible to the current user.
- [ ] Tests cover: direct approval by each approver role; proxy approval with comment;
      proxy rejection when comment is missing; unauthorized role attempt.

---


### ISSUE-8: Grant IAB Chair authority to mark "action taken"

**Labels:** `roles`, `usability`
**Priority:** Medium
**Milestone:** M2
**Origins:** liaison-tooling-2025a #9

#### Background

ietf-tools/liaison-tooling-2025a#9 requests that Secretariat, IAB Liaison Coordinators, and
IAB Chair can mark a liaison statement as "action taken." The current `_can_take_care()` (views.py:45–55) allows
Secretariat and Liaison Coordinator but not IAB Chair. This is a small targeted change.

#### Requirements

- Add "IAB Chair" to the role check in `_can_take_care()`.
- Confirm the complete list is: Secretariat, Liaison Coordinator, IAB Chair (plus any
  person whose address appears in the statement contacts).

#### Acceptance Criteria

- [ ] IAB Chair can mark liaison statements as "action taken".
- [ ] Other existing behavior in `_can_take_care()` is unchanged.

---

### ISSUE-9: Allow Liaison Managers and Coordinators to edit posted statements and repost

**Labels:** `roles`, `workflow`, `usability`
**Priority:** Medium
**Milestone:** M2
**Origins:** liaison-tooling-2025a #45

#### Background

ietf-tools/liaison-tooling-2025a#45 requests that liaison managers (for groups they are
responsible for) and liaison coordinators can edit posted statements and repost. This flow
corrects errors without a full resubmission cycle and is the primary correction path
described in ISSUE-28.

#### Requirements

- Liaison Coordinators can edit any posted statement.
- Liaison Managers can edit posted statements where the SDO group matches a group they are
  responsible for.
- Editing a posted statement re-enters the review workflow (coordinator review and, if
  applicable, AD approval).
- Statement history records the edit and the subsequent re-posting.
- Previous versions of an edited statement are retained in the database but are not exposed
  via the web interface or `api/v1`. Liaison Coordinators and Secretariat can access prior
  versions.

#### Acceptance Criteria

- [ ] "Edit" action appears on posted statements for Liaison Managers and Coordinators.
- [ ] Editing a posted statement moves it back through the review/approval steps.
- [ ] History shows the edit event and the new post event.
- [ ] Previous versions are stored in the database and accessible to Liaison Coordinators
      and Secretariat only; they are not exposed in the public web UI or `api/v1`.

---

### ISSUE-10: Implement RFC 4053bis contact field model: add From-Liaison-Contact and To-Liaison-Contact fields

**Labels:** `data-model`, `rfc-compliance`
**Priority:** Medium
**Milestone:** M3
**Origins:** RFC 4053bis; liaison-tooling-2025a #8, #28

#### Background

RFC 4053bis defines a richer contact field model than the current tool implements:

| RFC Field | Semantics | Current tool |
|-----------|-----------|--------------|
| From-Contact | Contact emails of originating body | `from_contact` (partial) |
| **From-Liaison-Contact** ("Send Reply To") | Reply routing address, e.g. `liaison-coordination@iab.org` | **Missing** |
| To-Contact | Contact emails of receiving body | `to_contacts` |
| **To-Liaison-Contact** ("Send To") | Central distribution address; if present, statement sent only here | **Missing** |

ietf-tools/liaison-tooling-2025a#8 and ietf-tools/liaison-tooling-2025a#28 discuss
from_contact semantics but do not propose the full RFC split.

#### Requirements

- Add `from_liaison_contact` CharField to `LiaisonStatement` (the "Send Reply To" address).
- Add `to_liaison_contact` CharField to `LiaisonStatement` (the "Send To" address).
- Update `IncomingLiaisonForm` and `OutgoingLiaisonForm` to include these fields.
- Update `send_liaison_by_email` / `notify_pending_by_email` to use these fields for routing.
- Write a reversible migration.

#### Acceptance Criteria

- [ ] `LiaisonStatement` model has `from_liaison_contact` and `to_liaison_contact`.
- [ ] Forms expose both fields with appropriate help text and validation.
- [ ] Email routing uses `to_liaison_contact` when set (central distribution).
- [ ] Migration is reversible.

---

### ISSUE-11: Replace from_contact field restrictions with a dropdown of the sender's active addresses

**Labels:** `roles`, `usability`, `outgoing`
**Priority:** High
**Milestone:** M3
**Origins:** liaison-tooling-2025a #4, #8; code analysis (forms.py:529–553)

#### Background

`set_from_contact_field()` (forms.py:529) currently handles `from_contact` differently by
role: for most roles it disables the field with a hardcoded value (forms.py:540–541); for
IAB Chair and Liaison Coordinator it forces a specific initial value (forms.py:533–540).
The IAB requested a workaround (now in place) that allows free-text override for incoming
liaisons (forms.py:485–494).

ietf-tools/liaison-tooling-2025a#8 requests that users can choose any address from their
datatracker account. ietf-tools/liaison-tooling-2025a#4 states that IAB Chair, IETF Chair,
and Liaison Coordinators should have no field restrictions.

#### Requirements

- Replace the disabled field for most roles with a dropdown of the user's active registered
  email addresses (`person.email_set.filter(active=True)`).
- Remove hardcoded forced initial values for IAB Chair, IETF Chair, and Liaison Coordinator;
  these roles should have the same free-choice behavior as Secretariat.
- Secretariat retains the ability to type any valid address.
- Remove the IAB-requested workaround comment and dead code in `IncomingLiaisonForm`.

#### Acceptance Criteria

- [ ] All roles see a dropdown of active registered addresses for `from_contact`; no role
      has the field disabled or forced to a hardcoded value.
- [ ] Secretariat can type any valid address.
- [ ] IAB Chair, IETF Chair, and Liaison Coordinator have no forced initial value.
- [ ] IAB workaround dead code in `IncomingLiaisonForm` is removed.

---

### ISSUE-12: Maintain a per-group "To Contacts" list

**Labels:** `data-model`, `usability`
**Priority:** Medium
**Milestone:** M3
**Origins:** liaison-tooling-2025a #31

#### Background

ietf-tools/liaison-tooling-2025a#31 asks to maintain a stored "To Contacts" list per external
group, so that the `to_contacts` field is pre-populated based on the selected `to_group`
rather than requiring manual entry on every statement.

#### Requirements

- Store a contact list per external (SDO) group — likely as a new related model or a field
  on `Group`.
- When a `to_group` is selected in the form (via the existing AJAX endpoint), pre-populate
  `to_contacts` from the stored list.
- Users can override the pre-populated list before submitting.
- The app provides UI for Liaison Coordinators and Secretariat to maintain the per-group
  list.

#### Acceptance Criteria

- [ ] Per-group to-contact list is stored in the database.
- [ ] Selecting a `to_group` in the form triggers pre-population of `to_contacts`.
- [ ] Pre-populated value is editable.
- [ ] Liaison Coordinators and Secretariat can create, update, and delete per-group contact
      lists via the app UI.

---

### ISSUE-13: Restrict purpose field to RFC 4053bis-defined values

**Labels:** `data-model`, `rfc-compliance`
**Priority:** Medium
**Milestone:** M3
**Origins:** RFC 4053bis §2.2, §1.1 changelog

#### Background

RFC 4053bis §2.2 defines exactly three purpose values: "For Information", "For Action",
"In Response." The current tool adds "For Comment" and "Other."

RFC 4053bis §1.1 explicitly documents the removal of "For Comment" with a clear rationale:

> "The purpose 'For Comment' was removed as either 'For Information' or 'For Action' can
> be used instead; depending if a deadline is needed or not. In the current record of
> statements, 'For Comment' has been rarely used indicating that this purpose is not needed
> or at least its meaning was not clear."

"Other" has no RFC basis and has no coherent semantics — it is an escape hatch that
undermines the purpose field's utility.

#### Proposed resolution

Remove "For Comment" and "Other" as selectable values for new statements. Existing
statements retain their recorded values in the database for display purposes (historical
accuracy), but these values cannot be set on new or edited statements.

No data migration of existing records is needed; the values remain valid as stored data,
just not as form choices.

#### Implementation requirements

- Remove "For Comment" and "Other" from the `LiaisonStatementPurposeName` choices
  available in the statement creation and edit forms. (Do not delete the database rows —
  just exclude them from form choices so historical records remain readable.)
- Update form help text to describe when to use "For Information" vs "For Action" (with
  or without a deadline) in place of the removed options.
- Update `set_required_fields()` in forms.py (line 456) which currently checks for
  `purpose in ['action', 'comment']`; remove the 'comment' check.

---

### ISSUE-14: Remove `technical_contacts` field (emails not sent; RFC supersedes it)

**Labels:** `data-model`, `bug`, `rfc-compliance`
**Priority:** Medium
**Milestone:** M3
**Origins:** datatracker issue tracker (email bug); code analysis

#### Background

`LiaisonStatement.technical_contacts` (models.py:43) stores contact addresses for
technical clarification. However, emails are not actually sent to these addresses — a known
but untracked bug. RFC 4053bis folds technical experts into From-Contact. The apparent
intent (per conflict analysis) is to remove the field in the new tool. No
liaison-tooling-2025a issue tracks this.

#### Requirements

- Remove `technical_contacts` from the `LiaisonStatement` model.
- Remove from all forms (`LiaisonModelForm` field validators at forms.py:308),
  views, templates, and search queries (forms.py:218).
- Write a migration that handles existing non-empty values (log them as a history comment
  before removal, or migrate to `from_contact` where appropriate).

#### Acceptance Criteria

- [ ] `technical_contacts` field removed from model.
- [ ] All references removed from forms, views, templates, and queries.
- [ ] Migration handles existing non-empty `technical_contacts` data without data loss.

---

### ISSUE-15: Change outgoing email From address to `liaison-coordination@iab.org`

**Labels:** `email`
**Priority:** Medium
**Milestone:** M4
**Origins:** liaison-tooling-2025a #1, #38; RFC 4053bis §2.1

#### Background

The current outgoing email From address is `statements@ietf.org` (`LIAISON_UNIVERSAL_FROM`).
RFC 4053bis §2.1 defines "From-Liaison-Contact (Send Reply To)" and states:

> "For liaison statements sent by the IETF, this address should be the alias of the liaison
> manager, if applicable, or an address maintained by the IAB for liaison management such
> as liaison-coordination@iab.org."

ietf-tools/liaison-tooling-2025a#1 states external parties should not use `statements@ietf.org`
as a reply-to. ietf-tools/liaison-tooling-2025a#38 raises the question of what address to
use. The RFC answers directly: `liaison-coordination@iab.org` is the appropriate address for
IETF liaison management.

Note the RFC distinguishes email content fields (From-Liaison-Contact) from SMTP headers.
The relevant decisions for the tool are:
- **Email From: header** (`LIAISON_UNIVERSAL_FROM`): change to `liaison-coordination@iab.org`.
- **Reply-To: header**: set to the `From-Liaison-Contact` field value of the statement
  (ISSUE-10), defaulting to `liaison-coordination@iab.org` when not set.
- **`statements@ietf.org`**: retain as a receive-only alias for transition purposes only;
  it must not appear in any outgoing email, and other SDOs should be directed to stop
  using it.

#### Requirements

- Update `LIAISON_UNIVERSAL_FROM` (or equivalent setting) to `liaison-coordination@iab.org`.
- Set the `Reply-To:` header in outgoing emails to the statement's `from_liaison_contact`
  value (ISSUE-10), defaulting to `liaison-coordination@iab.org` when not set.
- Remove or update any UI help text referencing `statements@ietf.org` as the send address.
- Coordinate with infrastructure to retain `statements@ietf.org` as a receive-only alias.

#### Acceptance Criteria

- [ ] Outgoing liaison emails have `From: liaison-coordination@iab.org`.
- [ ] `Reply-To:` header reflects `from_liaison_contact` field (or default).
- [ ] No UI text instructs external parties to use `statements@ietf.org`.
- [ ] `statements@ietf.org` does not appear in any outgoing email header.

---

### ISSUE-16: Always CC `liaison-coordination@iab.org` on all liaison statement emails and approval requests

**Labels:** `email`
**Priority:** High
**Milestone:** M4
**Origins:** liaison-tooling-2025a #10, #42

#### Background

ietf-tools/liaison-tooling-2025a#10 requests that `liaison-coordination@iab.org` is CC'd on
all liaison statements. ietf-tools/liaison-tooling-2025a#42 requests it be CC'd on all
approval request notification emails. This should be a hard system requirement, not a
user-configurable option.

#### Requirements

- Modify `send_liaison_by_email()` (mails.py) to always include `liaison-coordination@iab.org`
  in the CC.
- Modify `notify_pending_by_email()` (mails.py) to always include
  `liaison-coordination@iab.org` in the CC.
- This CC must not be removable from the form's CC field.

#### Acceptance Criteria

- [ ] `liaison-coordination@iab.org` appears in CC for all statement emails.
- [ ] `liaison-coordination@iab.org` appears in CC for all approval notification emails.
- [ ] Users cannot remove this address from the CC field.

---

### ISSUE-17: Always include responsible ADs in CC

**Labels:** `email`, `outgoing`
**Priority:** High
**Milestone:** M4
**Origins:** liaison-tooling-2025a #29

#### Background

ietf-tools/liaison-tooling-2025a#29 requires that responsible ADs are always in the CC field.
Currently ADs are included or excluded depending on how the form is filled out, with no
enforcement.

#### Requirements

- When computing CC recipients in `send_liaison_by_email()` (mails.py), automatically
  include the ADs responsible for the `from_groups`.
- This enforcement must occur at the send-mail layer, not only as a form default.
- Responsible ADs cannot be removed from CC by the submitter.

#### Acceptance Criteria

- [ ] AD emails derived from `from_groups` are included in every outgoing CC.
- [ ] This is enforced at mail-send time regardless of what the user entered in the CC field.

---

### ISSUE-18: Send state-change notification emails to all relevant parties

**Labels:** `email`, `workflow`
**Priority:** High
**Milestone:** M4
**Origins:** liaison-tooling-2025a #18; ietf/datatracker #8271

#### Background

ietf-tools/liaison-tooling-2025a#18 requests email notification on state changes to:
coordination, liaison manager, From/To/CC contacts. Currently, email behavior on state
transitions is inconsistent.

#### Requirements

Define and implement notifications for each state transition:

| Transition | Who to notify |
|------------|--------------|
| Marked ready for review | Responsible ADs (if approval required); LMs for each to_group, or Coordinators for to_groups with no assigned LM |
| AD approved (gate 1 complete) | LMs for each to_group (and Coordinators standing in for groups with no LM) — gate 2 review window is now open |
| All per-group LM reviews complete (gate 2 complete) | IAB Liaison Coordinators — gate 3 review window is now open |
| Coordinator final review complete (gate 3 complete) | Submitter |
| Posted | From, To, CC contacts, Liaison Managers; submitter is CC'd on the outgoing email |
| Marked dead | Submitter, Liaison Coordinators |

#### Acceptance Criteria

- [ ] Each transition triggers the notifications listed above.
- [ ] Notification emails are defined in mails.py with appropriate subject and body templates.
- [ ] No duplicate notifications are sent if a Coordinator records a proxy gate-2 review
      that happens to be the last required one, then immediately submits the gate-3 review.

---

### ISSUE-19: Send deadline reminder emails

**Labels:** `email`, `usability`
**Priority:** Medium
**Milestone:** M4
**Origins:** liaison-tooling-2025a #17

#### Background

ietf-tools/liaison-tooling-2025a#17 requests automatic deadline reminder emails for liaison
statements with a deadline and no "action taken" tag.

#### Requirements

- A Celery task identifies statements with `purpose=action`, `deadline` set, and no
  `action_taken` tag, and sends reminder emails at configured intervals before the deadline.
- The reminder schedule (e.g., days-before-deadline at which reminders fire) is a
  global datatracker setting, configurable via the datatracker's settings infrastructure —
  not per-statement and not hardcoded.
- Emails are sent to `action_holder_contacts` and `response_contacts`.
- Reminders stop once "action taken" is recorded.

#### Acceptance Criteria

- [ ] A Celery task implements the reminder logic; no synchronous management command is used.
- [ ] Reminder email includes statement title, deadline, and a link.
- [ ] No reminders sent after "action taken" is recorded.
- [ ] Reminder schedule is defined in the global datatracker settings and controls when
      the task fires reminders.

---

### ISSUE-20: Allow preview of the email before posting

**Labels:** `usability`, `email`
**Priority:** Medium
**Milestone:** M5
**Origins:** liaison-tooling-2025a #14; liaison-tooling-2025a README

#### Background

The README states: "The editing process will clearly capture whether the statement should be
emailed when posted, and allow preview of the emailed message." Currently there is no
preview capability.

#### Requirements

- Statement editing form includes an explicit "send by email when posted" checkbox (stored
  on the statement or as a flag).
- A "Preview email" button/link renders the exact email content — headers, body, and
  attachment list — without actually sending.
- Preview is accessible during editing without submitting the form.

#### Acceptance Criteria

- [ ] "Send by email when posted" flag is persisted on the statement.
- [ ] "Preview email" renders the full email template in the browser.
- [ ] Preview is accessible without submitting.

---

### ISSUE-21: Support saving liaison statements as drafts before readying for review

**Labels:** `workflow`, `usability`, `email`
**Priority:** Medium
**Milestone:** M1
**Origins:** liaison-tooling-2025a #14; liaison-tooling-2025a README

#### Background

The README describes statements being "entered, and then edited (perhaps by multiple people)"
before being readied for review. ietf-tools/liaison-tooling-2025a#14 includes support for
drafts. Currently there is no explicit draft state; statements become pending immediately.

A draft that never leaves the `draft` state has no public record and may be hard-deleted
— this is the only case in the liaison tool where hard deletion of a statement is permitted.

#### Requirements

**Draft editing and review:**
- Allow creating a statement in the `draft` state.
- Multiple authorized users (per the role rules in the README) can edit a draft.
- Draft statements are visible only to the users authorized to edit or review them; they
  are not publicly visible and are excluded from `/api/v1`.
- History records all edits to the draft.
- "Mark as ready for review" action moves the statement into the review queue and triggers
  ISSUE-18 notifications.
- Drafts are not visible to approvers until marked ready.

**Manual deletion:**
- Any authorized editor of a draft may hard-delete it at any time while it remains in
  the `draft` state. Hard deletion removes the statement and all associated records
  (events, attachments) from the database.

**Auto-expiry:**
- A Celery task checks for draft statements that have remained in the `draft` state beyond
  a configured expiry period. The expiry period is a global datatracker setting.
- At a second configurable time before expiry, a warning email is sent to the original
  drafter and all IAB Liaison Coordinators notifying them of the impending deletion.
  Both time values are global datatracker settings.
- If the statement is still in `draft` state when the expiry period elapses, it is
  hard-deleted by the task.

#### Acceptance Criteria

- [ ] A draft statement can be created and edited without triggering approval notifications.
- [ ] "Mark as ready for review" is a distinct action that transitions the statement.
- [ ] Drafts are not shown in the coordinator pending-review queue or the approver pending-approval queue.
- [ ] Draft statements are inaccessible to unauthenticated users and to authenticated users
      without an edit/review role on that statement.
- [ ] Draft statements are excluded from `api/v1`.
- [ ] An authorized editor can hard-delete a draft; the statement and all associated records
      are removed from the database.
- [ ] A Celery task sends a warning email to the drafter and Liaison Coordinators at the
      configured warning interval before expiry.
- [ ] The Celery task hard-deletes drafts that reach the configured expiry age.
- [ ] Expiry period and warning interval are defined in global datatracker settings.

---

### ISSUE-22: Restrict replies to posted statements only

**Labels:** `workflow`, `usability`
**Priority:** Low
**Milestone:** M5
**Origins:** liaison-tooling-2025a #36; RFC 4053bis

#### Background

ietf-tools/liaison-tooling-2025a#36 questions whether replies should be possible on pending
statements. RFC 4053bis implies responses are to publicly recorded (i.e., posted) statements.
The current `_can_reply()` (views.py:33–43) allows replies regardless of statement state.

#### Requirements

- `_can_reply()` must also check `liaison.state_id == 'posted'` before returning `True`.
- The reply UI element is not shown for non-posted statements.

#### Acceptance Criteria

- [ ] `_can_reply()` returns `False` for statements not in the `posted` state.
- [ ] Reply button/link is absent from the detail view for pending or approved statements.

---

### ISSUE-23: Allow adding comment, attachment, and reference when marking "action taken"

**Labels:** `usability`
**Priority:** High
**Milestone:** M5
**Origins:** liaison-tooling-2025a #16; ietf/datatracker #9438, #9387

#### Background

ietf-tools/liaison-tooling-2025a#16 requests that marking a statement "action taken" can
simultaneously add a comment, a file attachment, and/or a reference to a related statement.
Currently the mark-as-taken action stores only the tag change with no supporting information
and does not create a `LiaisonStatementEvent` record (ietf/datatracker#9438).

Additionally, when a statement with purpose "In Response" is posted, the statement it is
responding to should be automatically marked "action taken" (ietf/datatracker#9387).

#### Requirements

- The "mark action taken" form includes optional fields: comment textarea, file upload,
  related statement selector.
- Marking action taken always creates a `LiaisonStatementEvent` (type `comment` or a
  dedicated `action-taken` event type), attributed to the person who performed the action.
- Submitted comment and attachment are recorded as additional `LiaisonStatementEvent` entries.
- Related statement reference is stored via `RelatedLiaisonStatement`.
- When a statement with purpose "In Response" is posted, automatically mark the referenced
  statement as "action taken" if it is not already marked.

#### Acceptance Criteria

- [ ] "Action taken" form includes comment, file, and related-statement fields.
- [ ] Marking action taken always creates a history event, even with no ancillary fields.
- [ ] Submitted ancillary information creates appropriate additional history events.
- [ ] All three ancillary fields are optional.
- [ ] Posting a statement with purpose "In Response" automatically marks the referenced
      statement as "action taken."

---

### ISSUE-24: Re-enable attachment removal in the UI

**Labels:** `attachments`, `usability`
**Priority:** Medium
**Milestone:** M5
**Origins:** code analysis (UI comment: "will be replaced in next generation of liaison tool")

#### Background

The current UI disables attachment removal with the comment "will be replaced in next
generation of liaison tool." The underlying `LiaisonStatementAttachment.removed` boolean
field supports hiding attachments from public view, but the UI action is not wired up.

Attachment records are never hard-deleted (per ISSUE-26's never-delete policy). The
`removed` flag hides an attachment from public view while preserving the database record.

#### Requirements

- Re-enable the attachment remove button in the statement edit UI.
- Removal sets `removed=True` and creates a `modified` history event.
- Only Secretariat and Liaison Coordinator can remove attachments.
- Removed attachments are excluded from the public detail view
  (`active_attachments()` in models.py:145 already handles this).
- The attachment record and file are retained in the database; no file deletion occurs.

#### Acceptance Criteria

- [ ] Remove button is functional for Secretariat and Liaison Coordinator.
- [ ] Removing an attachment sets `removed=True` and creates a history event.
- [ ] Removed attachments do not appear in the public statement view.
- [ ] No attachment file or database record is hard-deleted.

---

### ISSUE-25: Allow adding attachments to existing and posted statements

**Labels:** `attachments`, `usability`
**Priority:** Medium
**Milestone:** M5
**Origins:** liaison-tooling-2025a #43

#### Background

ietf-tools/liaison-tooling-2025a#43 requests allowing attachments to be added to existing
statements, including posted ones. Currently, attachments can only be added during the
initial create/edit flow before posting.

#### Requirements

- Add an "Add attachment" action on the statement detail view for posted statements.
- Authorized roles: Liaison Manager (for relevant groups), Liaison Coordinator, Secretariat.
- Adding an attachment creates a `modified` history event.
- For posted statements, the new attachment does not re-trigger the approval workflow.

#### Acceptance Criteria

- [ ] "Add attachment" button appears on posted statement detail for authorized roles.
- [ ] Attachment upload succeeds for posted statements.
- [ ] A history event is created recording the new attachment.

---

### ISSUE-26: Disposition of rejected/unapproved statements: dead state

**Labels:** `workflow`, `data-model`
**Priority:** High
**Milestone:** M1
**Origins:** liaison-tooling-2025a #34; liaison-tooling-2025a README

#### Decision

Statements and their attachments are **never deleted**. Rejected or abandoned statements
are marked `dead`. The existing `dead` state and resurrection mechanism in the codebase are
retained and extended with the following requirements:

- **Reason required**: marking a statement dead must capture a reason as a mandatory
  comment, recorded as a `LiaisonStatementEvent`.
- **Access control**: for dead statements that were never posted, all roles except Liaison
  Coordinator and Secretariat receive a 404. For dead statements that were once posted
  (i.e., previously had `state=posted`), the public detail page displays "This statement
  has been removed" rather than a 404, since the URL was previously public.
- **API exclusion**: dead statements are excluded from `/api/v1` responses.
- **Resurrection**: Liaison Coordinators and Secretariat can resurrect a dead statement
  into `pending` state via the existing resurrect view. This is unchanged from the current
  implementation.
- **Existing dead statements**: remain dead with no change.
- **Existing `not-approved` statements**: migrated to `dead` via a one-time data migration;
  a `killed` event is created for each with the description "not-approved".
- **Attachments**: attachment records are never hard-deleted; the existing `removed` flag
  on `LiaisonStatementAttachment` is used to hide them from public views.

#### Implementation requirements

- Update the `killed` event creation path to require a non-empty reason string, stored as
  the event `desc`.
- Update views: dead statements that were never posted return 404 for all roles except
  Liaison Coordinator and Secretariat; dead statements that were once posted display
  "This statement has been removed" to unauthenticated users and roles without coordinator
  or secretariat access.
- Update `/api/v1` liaison queryset to exclude `state=dead`.
- Write a data migration: for any statement with `state=not-approved`, set `state=dead`
  and create a `killed` event with `desc="not-approved"`.
- No new `LiaisonStatementState` rows are needed; the existing `dead` row is reused.

#### Acceptance Criteria

- [ ] Marking a statement dead without a reason is rejected by the form.
- [ ] Dead statements that were never posted return 404 for all roles except Liaison
      Coordinator and Secretariat.
- [ ] Dead statements that were once posted display "This statement has been removed" to
      public users rather than a 404.
- [ ] `/api/v1` does not include dead statements.
- [ ] Liaison Coordinators and Secretariat can resurrect dead statements to `pending`.
- [ ] Data migration converts `not-approved` statements to `dead` correctly.
- [ ] No statement or attachment record is ever hard-deleted.

---

### ISSUE-27: Pre-send group notification (RFC 4053bis §4.2)

**Labels:** `workflow`, `rfc-compliance`, `usability`
**Priority:** Medium
**Milestone:** M1
**Origins:** RFC 4053bis §4.2; liaison-tooling-2025a #14

#### Background

RFC 4053bis §4.2 states:

> "Even if the responsible chairs or ADs intend to send a liaison statement without
> establishing additional consensus, the originator should inform the group it represents
> prior to its transmission and not only when the statement is already sent and recorded."

This uses "should" (a recommendation), not "MUST" (a requirement). ietf-tools/liaison-tooling-2025a#14 treats it as
optional. However, leaving it entirely absent from the tool misses an opportunity to remind those
handling the statement to consider this responsibility.

#### Proposed resolution

Implement as a **required acknowledgement checkbox** on the AD approval action (ISSUE-7),
not as a hard workflow gate with external verification.

Rationale: The RFC's "should" language means blocking posting on an unverified notification
would be disproportionate. A checkbox makes the obligation explicit and deliberate without
creating a step that could introduce unnecessary delay to time-sensitive statements. The tool
cannot verify the notification actually occurred.

#### Implementation requirements

- The AD approval form (ISSUE-7) includes a required boolean field:
  "The appropriate notifications to the group(s) this statement represents prior to posting and sending it (per RFC 4053bis) have been made."
- Checking it is required to submit the approval; the check is stored as a history event.
- Applies to outgoing statements only.
- No separate workflow step, email, or external gate is needed.

---

### ISSUE-28: Handling statements posted in error

**Labels:** `workflow`, `data-model`
**Priority:** Medium
**Milestone:** M1
**Origins:** liaison-tooling-2025a #35

#### Background

ietf-tools/liaison-tooling-2025a#35 asks how to handle statements posted in error. There are
two distinct cases, with different resolutions:

**Case 1: The content is wrong and needs to be corrected.**
Use the edit-and-repost flow from ISSUE-9. The statement is edited, goes back through the
review/approval workflow, and is re-emailed with corrected content. Prior versions are
retained in the database per ISSUE-9.

**Case 2: The statement should not have been sent at all.**
A Liaison Coordinator marks the statement `dead` with a mandatory reason comment (ISSUE-26).
Marking the statement `dead` removes it from public view (the URL shows "This statement
has been removed") and excludes it from `/api/v1`. However, if the statement was already
emailed to external parties, that communication cannot be recalled — the email record
cannot be revised.

RFC 4053bis §3 emphasises that the value of a liaison statement comes precisely from its
public record: "the public record will attest that certain information has been communicated
between the organizations." This is why hard deletion is not supported.

#### Requirements

No new data model changes are required beyond ISSUE-9 and ISSUE-26. This issue documents
the policy resolution only.

#### Acceptance Criteria

- [ ] Liaison Coordinators can edit and repost a posted statement with incorrect content
      (covered by ISSUE-9).
- [ ] Liaison Coordinators can mark a posted statement `dead` with a reason when it should
      not have been sent (covered by ISSUE-26).
- [ ] No hard-delete mechanism is provided.

---

### ISSUE-29: Remove Authorized Individual from incoming liaison entry roles

**Labels:** `roles`, `incoming`
**Priority:** High
**Milestone:** M2
**Origins:** code analysis (utils.py:20; forms.py:141–146)

#### Background

`INCOMING_LIAISON_ROLES` in utils.py:20 currently includes "Authorized Individual" — a
role held by persons from an external SDO who have been granted access in the datatracker
to enter incoming liaison statements on behalf of that SDO. This role is removed: incoming
liaison statements are entered only by IETF-side roles (Liaison Manager, IAB Liaison
Coordinator, Secretariat). External parties do not enter statements into the tool directly.

The `external_groups_for_person()` function in forms.py:141–146 also grants Authorized
Individuals access to external group selection; this must be updated accordingly.

#### Requirements

- Remove "Authorized Individual" from `INCOMING_LIAISON_ROLES` in utils.py.
- Update `external_groups_for_person()` in forms.py to no longer grant external-group
  access to Authorized Individuals for the purpose of entering incoming statements.
- Verify no other incoming-liaison code paths rely on the Authorized Individual role.
- Update tests accordingly.

#### Acceptance Criteria

- [ ] `INCOMING_LIAISON_ROLES` does not include "Authorized Individual".
- [ ] Persons with only the Authorized Individual role cannot access the incoming liaison
      creation form.
- [ ] Existing Authorized Individual role assignments in the database are unaffected
      (the role may still exist for other purposes; only liaison entry access is removed).

---

### ISSUE-30: Extend comment access to all authorized parties

**Labels:** `workflow`, `usability`, `roles`
**Priority:** Medium
**Milestone:** M5
**Origins:** (new requirement)

#### Background

The `add_comment` view (views.py:221–246), `AddCommentForm` (forms.py:181–183), and both
`comment` and `private_comment` `LiaisonStatementEventTypeName` slugs already exist.
The view is restricted to Secretariat via `@role_required('Secretariat')`. The template
`detail_history.html` (lines 14–22) shows the "Add comment" button to a broader set of
roles (Area Director, Secretariat, IANA, RFC Editor), but those users receive a permission
error when they attempt to submit — a pre-existing bug.

#### Requirements

- Extend the `add_comment` view to permit any person who can edit, approve, or review the
  given statement, in addition to Secretariat.
- Fix the template permission check to match the updated view authorization.
- The comment text remains required and non-empty.
- The `private` comment option (controls visibility in the public history view) is retained.

#### Acceptance Criteria

- [ ] The "Add comment" button is shown to, and functional for, all roles with edit,
      approve, or review rights on the statement (not only Secretariat).
- [ ] The template permission check matches the view's actual authorization gate.
- [ ] Submitting an empty comment is rejected.
- [ ] The `private` checkbox continues to work as before.

---

### ISSUE-31: OPEN — Should liaison managers have a required review step for chair-entered statements?

**Labels:** `workflow`, `roles`, `outgoing`
**Priority:** Medium
**Milestone:** M1
**Origins:** (new question)

#### Question

When an outgoing liaison statement is entered by a WG Chair, Area Director, IETF Chair, or
IAB Chair, should the Liaison Manager responsible for the relevant SDO relationship be
required to review and acknowledge the statement before it proceeds to AD approval and
IAB Liaison Coordinator review?

#### Context

The current proposed workflow (ISSUE-4) routes outgoing statements directly from "mark
ready for review" to the parallel AD-approval / coordinator-review gates, with no
mandatory Liaison Manager step. Liaison Managers are notified on submission (ISSUE-18)
and can edit drafts and pending statements, but their review is not currently a blocking
gate.

A required Liaison Manager review step could catch procedural or relationship-management
concerns before a statement is sent. However, it would add a third sequential or parallel
gate to an already multi-step workflow. When no Liaison Manager is assigned for the
relevant SDO, the coordinator review step already present in the workflow covers that case.

#### Decision needed

- Is a Liaison Manager review gate required, optional, or not applicable for
  chair-entered statements?
- If required, does it run in parallel with AD approval and coordinator review, or must
  it precede them?

---

### ISSUE-32: Fix conditional reformatting of liaison statement body text

**Labels:** `bug`, `usability`
**Priority:** High
**Milestone:** M5
**Origins:** ietf/datatracker #10129

#### Background

The statement detail view applies a filter that detects lines exceeding 80 characters and
reformats them. This means the rendered body may differ from what was submitted, affecting
only some statements (those with long lines), creating inconsistent display behaviour and
making future template changes unpredictable.

#### Requirements

- Remove the 80-character line-wrapping filter from the liaison statement body rendering.
- Statement bodies are displayed exactly as submitted, with no automatic reformatting.

#### Acceptance Criteria

- [ ] Statements with lines longer than 80 characters render exactly as submitted.
- [ ] No reformatting filter is applied to the body field in any template or view.

---

### ISSUE-33: Migrate liaison attachments to the datatracker blobstore

**Labels:** `bug`, `attachments`
**Priority:** High
**Milestone:** M5
**Origins:** ietf/datatracker #7087, #6910

#### Background

Liaison attachments are currently served from `https://www.ietf.org/lib/dt/documents/LIAISON/`,
a legacy CDN path that routes through www.ietf.org infrastructure. ietf/datatracker#6910
notes this is unsustainable through planned IT infrastructure migrations; #7087 notes the
CDN rationale is no longer relevant. Existing references in RFC errata and external
documents point to the legacy URLs and must continue to resolve.

#### Requirements

- Store liaison attachments in the datatracker's blobstore and serve them via the
  datatracker's worker infrastructure, as close to the edge as possible.
- Implement redirects from the legacy `www.ietf.org/lib/dt/documents/LIAISON/` path so
  that existing external references continue to resolve.

#### Acceptance Criteria

- [ ] New attachment uploads are stored in the blobstore and served via workers.
- [ ] Existing attachment URLs resolve via redirect from the legacy path.
- [ ] No attachment links in the UI point to `www.ietf.org/lib/dt/documents/LIAISON/`.

---

### ISSUE-34: Preserve CC field contents on form validation error

**Labels:** `bug`, `usability`
**Priority:** Medium
**Milestone:** M5
**Origins:** ietf/datatracker #5340

#### Background

When the liaison statement submission form fails validation (e.g., an invalid email address
in the CC field), the CC list is silently reset to its default value rather than
re-populating with the user's input. The submitter may not notice the reset and resubmit
with unintended recipients.

#### Requirements

- On validation error, all form fields — including CC — must re-populate with the values
  the user submitted, not the field defaults.
- The validation error message must clearly identify which field failed and why.

#### Acceptance Criteria

- [ ] A validation error on any field re-renders the form with the user's submitted values
      intact across all fields.
- [ ] The CC field is never silently reset to defaults on error.

---

## Summary

| Issue | Title |
|-------|-------|
| ISSUE-1 | Remove "prior approval" bypass |
| ISSUE-2 | Redesign state machine for new workflow |
| ISSUE-3 | Auto-post attributed to (System) user |
| ISSUE-4 | Per-group LM reviews (gate 2) and coordinator final review (gate 3) for outgoing |
| ISSUE-5 | Post incoming statements after Liaison Manager entry, review, and approval |
| ISSUE-6 | Capture submitter's role |
| ISSUE-7 | Implement approval authority model for outgoing statements |
| ISSUE-8 | IAB Chair can mark "action taken" |
| ISSUE-9 | Allow Liaison Managers and Coordinators to edit posted statements and repost |
| ISSUE-10 | Add From-Liaison-Contact and To-Liaison-Contact fields |
| ISSUE-11 | Replace from_contact restrictions with dropdown of sender's active addresses |
| ISSUE-12 | Per-group To Contacts list |
| ISSUE-13 | Restrict purpose values to RFC set |
| ISSUE-14 | Remove technical_contacts field |
| ISSUE-15 | Change email From address to liaison-coordination@iab.org |
| ISSUE-16 | Always CC liaison-coordination@iab.org |
| ISSUE-17 | Always CC responsible ADs |
| ISSUE-18 | State-change notification emails |
| ISSUE-19 | Deadline reminder emails |
| ISSUE-20 | Email preview before posting |
| ISSUE-21 | Draft statement support |
| ISSUE-22 | Replies only on posted statements |
| ISSUE-23 | Comment/attachment/reference when marking action taken |
| ISSUE-24 | Re-enable attachment soft-hide in UI |
| ISSUE-25 | Add attachments to posted statements |
| ISSUE-26 | Retain `dead` state; require reason; restrict access; exclude from API; migrate `not-approved` → `dead` |
| ISSUE-27 | Pre-send group notification acknowledgement checkbox |
| ISSUE-28 | Handling statements posted in error: correct via ISSUE-9, retract via ISSUE-26 |
| ISSUE-29 | Remove Authorized Individual from incoming liaison entry roles |
| ISSUE-30 | Extend comment access to all authorized parties (currently Secretariat-only) |
| ISSUE-31 | OPEN: Should liaison managers review chair-entered statements before AD approval? |
| ISSUE-32 | Fix conditional reformatting of liaison statement body text |
| ISSUE-33 | Migrate liaison attachments to the datatracker blobstore |
| ISSUE-34 | Preserve CC field contents on form validation error |

---

## Attribution Index

| Origin | Referenced in |
|--------|--------------|
| liaison-tooling-2025a #1 | ISSUE-1, ISSUE-15 |
| liaison-tooling-2025a #2 | ISSUE-7 |
| liaison-tooling-2025a #4 | ISSUE-11 |
| liaison-tooling-2025a #8 | ISSUE-10, ISSUE-11 |
| liaison-tooling-2025a #9 | ISSUE-8 |
| liaison-tooling-2025a #10 | ISSUE-16 |
| liaison-tooling-2025a #14 | ISSUE-20, ISSUE-21, ISSUE-27 |
| liaison-tooling-2025a #16 | ISSUE-23 |
| liaison-tooling-2025a #17 | ISSUE-19 |
| liaison-tooling-2025a #18 | ISSUE-18 |
| liaison-tooling-2025a #28 | ISSUE-10, ISSUE-11 |
| liaison-tooling-2025a #29 | ISSUE-17 |
| liaison-tooling-2025a #31 | ISSUE-12 |
| liaison-tooling-2025a #34 | ISSUE-2, ISSUE-26 |
| liaison-tooling-2025a #35 | ISSUE-28 |
| liaison-tooling-2025a #36 | ISSUE-22 |
| liaison-tooling-2025a #37 | ISSUE-6 |
| liaison-tooling-2025a #38 | ISSUE-15 |
| liaison-tooling-2025a #41 | ISSUE-4, ISSUE-7 |
| liaison-tooling-2025a #42 | ISSUE-16 |
| liaison-tooling-2025a #43 | ISSUE-25 |
| liaison-tooling-2025a #45 | ISSUE-9 |
| RFC 4053bis §4.1 | ISSUE-1, ISSUE-7 |
| RFC 4053bis §4.2 | ISSUE-13, ISSUE-27 |
| RFC 4053bis (contact model) | ISSUE-10 |
| code analysis: utils.py:20; forms.py:141–146 (Authorized Individual) | ISSUE-29 |
| code analysis: forms.py:471 (auto-post) | ISSUE-5 |
| code analysis: forms.py:505–512 (prior approval) | ISSUE-1 |
| code analysis: forms.py:529–553 (field restrictions) | ISSUE-11 |
| code analysis: views.py:33–43 (_can_reply) | ISSUE-22 |
| code analysis: views.py:45–55 (_can_take_care) | ISSUE-8 |
| code analysis: models.py:43 (technical_contacts) | ISSUE-14 |
| code analysis: UI (attachment deletion disabled) | ISSUE-24 |
| ietf/datatracker #5340 | ISSUE-34 |
| ietf/datatracker #6910 | ISSUE-33 |
| ietf/datatracker #7087 | ISSUE-33 |
| ietf/datatracker #8271 | ISSUE-18 |
| ietf/datatracker #9387 | ISSUE-23 |
| ietf/datatracker #9438 | ISSUE-23 |
| ietf/datatracker #9457 | ISSUE-7 |
| ietf/datatracker #10129 | ISSUE-32 |
