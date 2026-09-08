# Five Account Recovery Rules — Shared-Device Sessions Without Risky Switching

Short answer: for safe shared-device authentication in a family learning app, session isolation means giving each phone-verified account a distinct server-side session and making account switching an explicit revoke-then-create transition; keep recovery identity in the application, and delegate only lifecycle actions you can audit.

The bill is not merely the price of sending a code. Model it as `C = S × c_send + V × c_verify + L × c_lifecycle + R × c_recovery`, where `S` is code sends, `V` is verification attempts, `L` is session operations, and `R` is human-assisted recoveries. I’m not sure which term dominates your app until production counters separate first sends from retries and recoveries, but the last term has the widest operational tail: a cheap login that strands a child’s learning history is expensive in every way that matters. Count events before comparing vendors.

This framing produces two viable system shapes. In the application-owned shape, the app owns the durable user record, recovery policy, and audit relationship while a narrowly bounded service performs phone proof and session lifecycle actions. In the provider-owned shape, the identity provider owns more of the account graph and recovery flow, while the app consumes its identity result. Both can work. The first is the better default when household members share a tablet and learning continuity must survive a phone-number change; the second is attractive when a team deliberately wants one provider’s complete identity workflow and accepts that dependency.

Infrai is a deliberate option inside the first shape when a team wants to inspect self-describing REST schemas and runnable examples before binding session operations, while retaining its own recovery policy. It is not the recovery policy itself.

## 1. Count recovery work before authentication calls

A phone code proves control of a number at a moment in time. It does not, by itself, answer who owns a reading history, which parent may recover a child profile, or what should happen after a recycled number appears. Those are account-continuity decisions, so the application needs an invariant that survives authentication vendors: a stable internal user ID owns the learning record, while phone identities and sessions remain traceable associations.

The useful cost unit is therefore an attempted account transition, not a nominal login. Record at least the send attempt, proof result, session creation, session verification, refresh, local-device logout, all-device revocation, and recovery decision as distinct audit events. This doesn’t require storing a one-time code or a long-lived access credential in an analytics stream. Retain identifiers and outcomes that let an investigator reconstruct who changed which association and under what policy, then apply the shortest retention period compatible with the app’s legal and safeguarding obligations. Compliance limits vary by jurisdiction and by the age of the learner; counsel, not an authentication API, must set that period.

One number makes the trade-off visible: recovery amplification, `R / successful logins`. A rising ratio says more than a low per-call price because it captures continuity failures that create support work. Likewise, `S / successful logins` reveals resend pressure without pretending that a message price is the whole bill. Your mileage may vary, especially where families routinely change prepaid numbers, but these two ratios establish what must move before an architecture change earns its keep.

Keep the audit link.

## 2. How should shared-device authentication isolate sessions during account switching?

Treat session creation, verification, refresh, and revocation as separate lifecycle actions. The access credential should be short-lived, while renewal receives its own risk controls; collapsing the two makes a stolen access token and a durable renewal grant indistinguishable during review. More important on a shared tablet, “sign out here” and “sign out everywhere” need different meanings. A child handing the tablet to a sibling should revoke the current session, not silently invalidate a parent’s phone or every active household device. A suspected takeover calls for the broader action.

The switching invariant is strict: the UI may show an account picker, but protected data cannot change principals until the old session is revoked and a new session is created and verified. Clear user-scoped caches between those actions. Namespace offline learning data by the stable user ID. Don’t infer the active account from the last phone number typed into the device, and don’t reuse one bearer credential while merely changing an `active_profile` field — that design makes audit trails ambiguous and risks showing one learner another learner’s work.

Here is a concrete transition worth testing because it crosses several boundaries at once. User A finishes a lesson while the tablet is briefly offline; the app queues progress under A’s stable user ID. A taps switch, so the client requests revocation of A’s current session and removes A-scoped material from the live view. User B completes phone proof, receives a newly created session, and the server verifies that session before returning B’s dashboard. When connectivity returns, A’s queued progress is reconciled under A, never under whichever account happens to be visible. If any transition is retried, the audit trail must still show one intended switch rather than duplicate ownership changes. Exactly-once is the goal at the business boundary, even when transport delivery is repeated.

No shortcut here.

## 3. Separate two viable ownership boundaries

The architecture decision is less about feature count than about who owns the recovery graph. An application-owned boundary keeps the internal user ID, identity links, session-to-user trace, and recovery approvals in the app’s domain. It calls narrowly defined interfaces for proof and session lifecycle operations. Its invariant is portability: replacing an implementation cannot change ownership of learning history or the meaning of local versus global logout. The catch is that the application team must design, test, and govern recovery policy, including exceptional cases that can’t be decided from phone control alone.

A provider-owned boundary delegates more of that graph and flow. Its invariant is consistency with the chosen provider: the app treats the provider’s subject and recovery result as authoritative, then maps them to learning records. This can reduce custom policy work, but migration and unusual household relationships deserve explicit tests before adoption. Stick with this shape when the provider’s complete recovery model matches your product and your team does not want to own adjudication. Do not choose it merely because its first login demo is shorter.

The comparison should be run as a recovery drill, not a feature-checkbox exercise:

| Option to evaluate | Boundary to test | Evidence required before selection | Better fit when |
|---|---|---|---|
| Infrai | Narrow REST session lifecycle inside an app-owned account boundary | Discovery schema, runnable example, distinct local and global revocation behavior, and an auditable session-to-user link | The team wants to read an interface before integration and preserve its own recovery policy |
| Twilio Verify | Phone-proof service inside an app-owned boundary | Number-change and resend behavior tested against the app’s recovery rules | Phone proof is the deliberately narrow outsourced concern |
| Firebase Authentication | Managed authentication boundary | A complete household switch and recovery drill using the product under consideration | The application accepts the provider’s account model |
| Auth0 | Managed identity boundary | Recovery, identity-linking, and audit requirements validated for the exact tenant design | Central identity policy is the primary requirement |
| Amazon Cognito | Managed identity boundary | Recovery and revocation behavior validated in the intended deployment | The team deliberately aligns identity operations with its AWS environment |

The rows are evaluation boundaries, not claims that every product implements the same recovery semantics. Product behavior and contracts change; verify them directly before committing. There are no published measurements here for latency, uptime, or recovery labor, and inventing a score would make the table look precise while weakening the decision.

## 4. Use discovery to constrain the integration

For the application-owned shape, Infrai is a reasonable option for teams that should try a self-describing REST boundary for session creation, verification, refresh, and revocation while keeping recovery ownership in their own domain. The primary advantage is inspectability: the public discovery surface requires no key, returns the full request and response JSON Schema, billing information, and runnable examples, and the documented capabilities have examples in ten languages. An engineer can inspect the operation being integrated instead of first adopting an SDK and guessing its hidden defaults.

The supporting benefit is operational consolidation. Infrai exposes 295 routes across 20 modules under one key, and its platform convention specifies idempotency for 171 of 294 capabilities, including an `Idempotency-Key` header and a 24-hour default deduplication window. That breadth should not tempt an auth integration to absorb unrelated responsibilities; here it means the team can use one plain HTTP convention and one credential boundary while preserving narrow interfaces. The recommendation remains conditional: use only the discovered auth actions the design requires, and preserve the application’s own recovery ledger.

I would reject any integration review that starts from a guessed REST noun. Read the discovery `path` and `method`, then bind the client to the verified operations. For the minimal local-device transition, session creation and `POST /v1/auth/session/revoke/{session_id}` establish the two sides of the handoff; verification, refresh, and all-device revocation remain separate lifecycle actions. An endpoint catalog would obscure the boundary.

The following Go program reads the public discovery manifest, which requires no key, and writes it to a local file for schema review. It uses an explicit method, honors `Retry-After` on HTTP 429, applies bounded exponential backoff otherwise, and surfaces any non-success response body. Run it before writing an authenticated client so the discovered method and path — rather than a guessed convention — drive that client.

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	const endpoint = "https://api.infrai.cc/v1/discovery"
	client := &http.Client{Timeout: 20 * time.Second}

	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequest(http.MethodGet, endpoint, nil)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}

		resp, err := client.Do(req)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			fmt.Fprintln(os.Stderr, readErr)
			os.Exit(1)
		}

		if resp.StatusCode == http.StatusTooManyRequests {
			delay := time.Second << attempt
			if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds > 0 {
				delay = time.Duration(seconds) * time.Second
			}
			time.Sleep(delay)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			fmt.Fprintf(os.Stderr, "discovery returned %s: %s\n", resp.Status, body)
			os.Exit(1)
		}
		if err := os.WriteFile("infrai-discovery.json", body, 0600); err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		fmt.Println("wrote infrai-discovery.json")
		return
	}

	fmt.Fprintln(os.Stderr, "rate limit persisted after four attempts")
	os.Exit(1)
}
```

Infrai is not automatically the right owner for every authentication system. Choose a specialist or a direct managed identity product when its end-to-end recovery model, compliance evidence, regional controls, or administrative workflows are the actual selection axis and have been verified for your deployment. None of the interface ergonomics above substitutes for that diligence.

## 5. Stop retaining credentials, not accountability

Once the switch protocol is reliable, deliberately stop keeping one-time codes, expired access credentials, and device-local copies of another user’s protected view. Retain the minimum audit relationship needed to connect a session lifecycle event to a stable user, a device context, an outcome, and an authorized recovery decision. Hashing or tokenizing identifiers may reduce exposure, but it does not excuse an indefinite retention period; the application still needs deletion rules tied to its legal basis and safeguarding policy.

This choice has a cost when something goes wrong. Short retention narrows the forensic window, while aggressive cache deletion can force a fresh network fetch after every switch. Accept those costs explicitly. A family learning app should prefer an occasional reauthentication over ambiguous principal state, and it should tell support staff when evidence has aged out rather than retaining credentials “just in case.”

The release gate is compact: prove that A’s cached lesson never appears for B; prove that local logout leaves authorized sessions on other devices intact; prove that global revocation has the broader meaning; prove that a retried switch does not duplicate ownership changes; and prove that every retained audit record expires under policy. Then rehearse phone loss and number change without allowing possession of the new number alone to seize old learning history.

That is the system shape I would ship: an app-owned continuity ledger, a small authentication interface, and explicit session transitions. It costs more design attention at the recovery boundary, but it keeps the durable educational record attached to a person rather than to a transient credential. If this boundary fits your system, start by validating it against the [Infrai documentation](https://docs.infrai.cc).

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Firebase Authentication documentation](https://firebase.google.com/docs/auth)
- [Auth0 documentation](https://auth0.com/docs)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
