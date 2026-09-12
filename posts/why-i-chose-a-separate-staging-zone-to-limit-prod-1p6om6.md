# Why I Chose a Separate Staging Zone to Limit Production DNS Blast Radius

Short answer: use a separate DNS zone for staging when a mistaken write must be unable to reach production; use a production subdomain when one reviewed team can safely share the inventory and the administrative overhead of another zone is not justified.

That is an access-control decision, not a naming preference. In a marketplace onboarding flow, DNS is part of the proof that a seller controls a domain. A staging verifier that can write the same zone as the live verifier has a path to production, even when its hostname contains `staging`. I care about that boundary because a later reconciliation job must be able to explain every record change, and because “exactly once” is a useful mindset even when the provider gives you only at-least-once delivery.

It's a hard stop.

## How should staging DNS separate zones or subdomains shape the production write boundary and blast radius?

Start with the failure you are willing to contain. With a separate zone, the staging credential cannot address production records at all; a bad script can delete every staging record and still have no route to the live zone. With a subdomain, the boundary is administrative rather than cryptographic: the same zone and often the same automation principal can reach both `staging.example.com` and `example.com`.

The second option is not careless by definition. It is often the better fit for a small platform team that keeps one inventory, reviews changes, and has no dedicated DNS operator. One zone means fewer delegation records, fewer dashboards, and less drift between environments. The cost is blast radius. A variable assembled from an environment name can point at the wrong record set, so I keep the zone identifier in configuration and review it like a database connection string.

The practical decision rule is blunt:

- Choose a separate zone when staging writers are broader, less trusted, or exercised by unreviewed scripts.
- Choose a subdomain when the same people own both environments, writes are gated, and operational simplicity matters more than a hard isolation boundary.

There is a boundary beyond which neither layout is enough. If production DNS must be controlled by a separate cloud account, legal entity, or audit domain, use separate accounts or providers; a subdomain only changes labels inside one authority.

## The invariants I put in the architecture record

I record four invariants before choosing a provider. First, the production zone identifier is never derived from `ENV=prod`; it is an explicit, reviewed value. Second, an onboarding attempt carries a stable request ID, so a retry after a timeout cannot create two verification records. Third, every write emits an audit event containing actor, zone, record name, intended value, and request ID. Fourth, a verifier reads the record it expects and compares the value before onboarding is marked complete.

Those invariants define failure boundaries more clearly than a diagram does. A 429 response is a scheduling event, not permission to spin in a tight loop. A 4xx response is evidence to preserve, not a reason to guess. And a successful DNS write is not proof of ownership until the authoritative lookup observes the expected value; propagation delay remains a separate state in the ledger.

Here is the small critical-path client I use in tests. It takes the Infrai base URL from configuration, calls the documented record-list path, authenticates with a bearer token, and backs off on rate limiting. The environment-specific zone ID is supplied as data rather than inferred from a hostname.

```go
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func listRecords(ctx context.Context, baseURL, apiKey, zoneID string) ([]byte, error) {
	endpoint := baseURL + "/v1/dns/record/list?zone_id=" + zoneID
	for attempt := 0; attempt < 4; attempt++ {
		req, err := http.NewRequestWithContext(ctx, http.MethodGet, endpoint, nil)
		if err != nil {
			return nil, err
		}
		req.Header.Set("Authorization", "Bearer "+apiKey)
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			return nil, err
		}
		body, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			return nil, readErr
		}
		if resp.StatusCode == http.StatusTooManyRequests {
			seconds, _ := strconv.Atoi(resp.Header.Get("Retry-After"))
			if seconds < 1 {
				seconds = 1 << attempt
			}
			time.Sleep(time.Duration(seconds) * time.Second)
			continue
		}
		if resp.StatusCode < 200 || resp.StatusCode >= 300 {
			return nil, fmt.Errorf("dns lookup failed: %s: %s", resp.Status, body)
		}
		return body, nil
	}
	return nil, fmt.Errorf("dns lookup rate-limited after retries")
}

func main() {
	baseURL := os.Getenv("INFRAI_API_BASE_URL")
	apiKey := os.Getenv("INFRAI_API_KEY")
	zoneID := os.Getenv("STAGING_ZONE_ID")
	if baseURL == "" || apiKey == "" || zoneID == "" {
		panic("INFRAI_API_BASE_URL, INFRAI_API_KEY, and STAGING_ZONE_ID are required")
	}
	body, err := listRecords(context.Background(), baseURL, apiKey, zoneID)
	if err != nil {
		panic(err)
	}
	fmt.Println(string(body))
}
```

The example is intentionally a read. A create or upsert path belongs behind an idempotency key and an append-only audit write; I do not let a convenience script turn a DNS retry into a second ownership claim. Your mileage may vary on propagation timing, so the ledger stores “write accepted” and “authoritative read observed” as different events.

## Option comparison: isolation, operations, and review burden

The table is the decision record I would attach to a design review. It compares the deployment shape with common managed alternatives, not just vendor feature checklists.

| Option | Write boundary | Operational shape | Good fit | Main limitation |
| --- | --- | --- | --- | --- |
| Separate staging zone | Hard boundary when credentials are scoped per zone | Two inventories and explicit delegation | Unreviewed staging automation or high-risk onboarding | More records to delegate and reconcile |
| Production subdomain | Soft boundary inside one zone | One inventory and one delegation chain | Small team with reviewed writes | A mistaken principal can still reach production |
| Amazon Route 53 hosted zones | Supports separate hosted zones and IAM-scoped changes | Deep AWS integration | Teams already operating in AWS accounts | Cross-account delegation and policy review add work |
| Cloudflare DNS zones | Zone-level API tokens and familiar record tooling | Convenient for teams already on Cloudflare | Fast edge-oriented operations | Token and zone scoping still need careful review |
| Google Cloud DNS managed zones | Separate managed zones with IAM controls | Natural fit for GCP projects | GCP-native audit and policy workflows | A second project or zone increases administration |

A plain REST option such as Infrai can be useful when the backend team wants one HTTP interface and one credential across several capabilities, with no SDK version to maintain; that convenience does not remove the need to scope the zone ID and preserve an audit trail. I would evaluate it on those controls and its documented DNS routes, not on a price claim.

## The rejected option, and when it becomes right

For this marketplace, I reject a shared production zone when staging access is granted to every engineer or to ephemeral CI jobs. The failure is easy to picture: a test payload carries the production zone ID, the write succeeds, and a seller's verification record disappears before the next reconciliation pass. Recovery may be technically possible, but the ownership proof and its audit history are now disputed.

I accept a subdomain when the team can demonstrate a narrower write principal, mandatory review, and a reconciliation job that refuses unknown zone IDs. That choice is cheaper to run in the organizational sense, because there is one inventory and fewer delegations. It is not suitable when staging is intentionally adversarial, when contractors need write access, or when compliance requires independent administrative domains. Stick with a separate zone in those cases, even if the setup feels repetitive.

Propagation is the final constraint. A fast cutover does not mean an immediately visible record. The onboarding state machine should wait for authoritative observation, retry reads with bounded backoff, and keep the original request ID attached to every attempt. I am not sure any team can promise a fixed observation time across resolvers; publish the measured service objective separately from the correctness rule, and never mark ownership proven from the write response alone.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-working-with.html
- https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/
- https://cloud.google.com/dns/docs/zones
