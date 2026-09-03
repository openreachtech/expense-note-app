# #sign-in  Signing in, signing out, and asking who is signed in
<!-- spec: sign-in @ sha256:d750e631554b125c -->
<!-- repositories: backend, frontend-staff -->

**Blocked: Q1 (blocking: yes).** Nothing in spec §9 holds a session, and
`SignInResult(staffMemberId)` carries nothing the client can authenticate with. Two of this
feature's own acceptance criteria cannot be met as designed. The mechanism is a design
decision routed to `/hora-spec` stage 4 — **checkpoint 1 does not start until it is settled**
(`../../questions/1.0.0/open.md`)

Conflict: this feature **opens the staff audience** — the SDL, the engine, the context pair and
          the one hand-written entry in `server/index.js` (which starts its servers with no
          scanning). #expense-entry and #monthly-summary then add resolver files only, which
          register by directory scanning. It also writes the scalars / `Pagination` / `Sort`
          block the later two append beside

Conflict: appends to the staff audience's SDL and to its GraphQL type declarations under
          `types/`. Two other features carry the same mark

Constraint: signing up and resetting a password by mail are out of scope **for now** (spec §4).
            Accounts are issued by an operator outside the product, so build **no sign-up
            operation and no sign-up screen** — and leave the password a one-way hash on the
            member of staff's own row, so a later reset changes that row alone

Constraint: there is no second actor (spec §3). Build no admin screen and no operator screen.
            The operator never signs in

Note: spec §7 forbids a password or a password hash reaching **any** response or **any** log
      line, and this feature's acceptance criteria repeat it. It is checkpoint 8's business as
      well as checkpoint 6's

Note: the two refusals — an address with no account, and a correct address with the wrong
      password — must be **identical**, and neither may say which of the two it was
      (spec §10 acceptance). That is a single error code, not two

Note: `signedInStaffMember` exists because a session is a cookie (spec §10.1). The screens ask
      it on opening; it is what sends an already-signed-in person on rather than asking twice

## Spec gate
- [ ] 1. Draft or confirm the specification
- [ ] 2. Verify the use cases can be met

## Backend gate
- [ ] 3. DB and API schemas
- [ ] 4. Stub API
- [ ] 5. The modules the implementation needs
- [ ] 6. Actual API
- [ ] 7. Worker
- [ ] 8. Security audit
- [ ] 9. Verify the use cases again, against the built API

## Frontend gate
- [ ] 10. Open the frontend
- [ ] 11. Reconfirm UI/UX and the use cases
- [ ] 12. Component design
- [ ] 13. The frontend modules the implementation needs
- [ ] 14. API client
- [ ] 15. UI
- [ ] 16. Wire the data-fetching logic in
- [ ] 17. Local test environment

## Acceptance gate
- [ ] 18. Acceptance (E2E and unit both)
