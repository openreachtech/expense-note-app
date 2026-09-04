# #sign-in  Signing in, renewing, signing out, and asking who is signed in
<!-- spec: sign-in @ sha256:e8d736e09febcf1f -->
<!-- repositories: backend, frontend-staff -->

Scope: **four operations, not three.** §10.1 adds `renewAccessToken` alongside `signIn`,
       `signOut` and `signedInStaffMember`, and `SignInResult` now carries `accessToken`. Q1
       is closed: the session is the two-token cookie flow, and its nine tables are
       #data-model's

Constraint: **the staff audience adds NO `healthCheck`.** The backend row ships one for its own
            audiences, and copying it across out of habit is the obvious thing to do — do not.
            §10.1 is the complete operation list, and §7's Authentication row is written
            against it staying that way: "every operation authenticates by one of two
            credentials". A fully-public operation nobody declared makes that false

Constraint: **two operations are reachable without a session, and only two.** `signIn`, which
            begins one, and `renewAccessToken`, which authenticates by the refresh-token cookie
            alone because it has to work once the access token has expired. `signOut` also
            proves itself by the cookie but still requires a session. Whatever the engine's
            skip-filter list ends up holding, those three are the operations that skip the
            access-token header — and nothing else does

Constraint: **rate limiting is this feature's work, and it is new scope** (§7). `signIn`, 10
            FAILED attempts per 15 minutes PER EMAIL ADDRESS — keyed on the address, not the
            caller's IP, because the staff share one office address and an IP-keyed limit
            would let one wrong password lock out everybody. Successes count for nothing.
            `renewAccessToken`, 60 per hour per refresh-token series

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

Note: the same holds for `renewAccessToken`'s three failure states — an expired, a revoked and
      an already-spent cookie are refused identically, and the refusal reveals which in none of
      them. A spent one revokes the whole series, which is the reuse detection §9.7's
      `used_at` and `revoked_at` exist for

Note: `renewAccessToken` belongs to the frontend's client layer and to no screen's call table
      (§10.2). Checkpoint 14 builds it; checkpoints 15 and 16 give it no UI

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
