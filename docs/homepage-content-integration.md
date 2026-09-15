# Homepage content and event API integration

Status: Frontend fixture implemented; supplier connection details recorded; API adapter pending final endpoints and credentials

## Ownership

The homepage has two deliberately separate content sources:

| Content | Owner | Initial source |
| --- | --- | --- |
| Featured and trending event cards | Event supplier | Local-only normalized design fixture |
| Section headings and supporting copy | WordPress | Normalized PHP fixture, later Fluid Content/ACF |
| Promoter CTA and benefit cards | WordPress | Normalized PHP fixture, later Fluid Content/ACF |
| Footer, navigation and newsletter copy | WordPress | Theme fixture, later global options |

Templates consume the normalized payload returned by
`OrbitHomepageContent::get()`. They do not know whether an event came from the
fixture, the supplier API or a cache. The future adapter should replace only
`featured.items` and `event_rows[*].items` through the
`orbit_homepage_content` filter.

The committed event fixture is enabled only in a local environment (or by the
explicit `ORBIT_USE_DESIGN_FIXTURES` development flag). It is never a
production fallback. When the API integration is enabled, only validated
supplier data or its last-known-good cache may populate event cards.

## Supplier requests

Use the standard event-list response defined in
`event-search-api-requirements.md`:

- `/v1/events?collection=homepage_featured&limit=6`
- `/v1/events?category=gigs&sort=trending&limit=6`
- `/v1/events?category=festivals&sort=trending&limit=6`
- `/v1/events?category=sport&sort=trending&limit=6`

These endpoint names are still proposals pending the supplier's technical design.
The `/v1/...` paths above are shorthand for `/api/eventsearch/v1/...` on the
selected environment host. Use the TEST, UAT or Live base URL recorded in the
[API connection guide](event-search-api-requirements.md#4-general-api-conventions);
do not append another version segment to that base URL.

WordPress calls the supplier directly from server-side PHP using
`Authorization: Bearer <environment-specific-token>`. Run collection refreshes
with bounded concurrency and shared request deduplication. Never send the token
to browser JavaScript or introduce a separate proxy service by default.

The supplier's suggested, adjustable defaults are 600 requests per minute and
50 simultaneous requests. Plan these as a shared outbound budget across
web requests and cache-warming workers; four homepage collections must not
trigger four new supplier requests for every visitor. Enforcement scope, burst
behaviour and response headers remain to be confirmed.

The supplier has proposed bearer tokens and IP allow-lists; no separate
permissions/scopes model has been specified. Fixed outbound IP addresses for
AWS-hosted PHP requests are not currently guaranteed. The supplier should confirm
whether bearer-token access can be supported without mandatory IP allow-listing.
Fixed-IP provision is subject to AWS hosting assessment and client agreement,
not an unconditional Project Simply commitment. If allow-listing is essential,
assess stable egress configuration and costs before confirming addresses for
all calling web servers and cache-warming workers. See the
[AWS hosting and allow-listing discussion](event-search-api-requirements.md#aws-hosting-and-ip-allow-listing--subject-to-assessment).

## Cache policy

Cache each collection independently so one slow or malformed response does not
invalidate the others.

1. Use an object-cache key derived from API environment, version, locale and an allowlisted
   normalized query; never derive a cache key from an unbounded request URL.
2. Keep a fresh entry for five minutes. Persist the last-known-good validated
   response separately, with its fetched timestamp, so transient eviction does
   not replace real supplier content with fixtures.
3. Serve fresh immediately. If fresh is missing, acquire a short lock and
   refresh; other requests receive stale data instead of stampeding the API.
4. On timeout, non-2xx response, invalid JSON or schema failure, serve the
   last-known-good entry within the agreed maximum stale window. If no valid
   cache exists, omit that event section and log the failure; never use the
   design fixture.
5. Warm all four collections from WP-Cron every four minutes. A page request
   remains allowed to refresh a cold cache.
6. Store the supplier `request_id`, response age and result count as metadata
   for logs, but never render them into public HTML.
7. Purge homepage collection keys after adapter/schema deployments, not on
   ordinary WordPress content saves.

Recommended limits: 2 second connect timeout, 4 second total timeout, 2 MB
maximum response, no automatic retry within the same visitor request, and an
initial six-hour maximum stale window. Product and supplier teams should confirm
the stale window because availability can change quickly.

## Validation and security

- Read the selected environment's base URL and bearer token from server environment/config constants; keep TEST, UAT and Live credentials and caches separate.
- Require HTTPS and an allowlisted hostname in non-local environments.
- Use `wp_safe_remote_get()` and do not follow redirects to a different host.
- Build query strings from an allowlist; do not proxy arbitrary browser
  parameters.
- Require a JSON object, a valid `items` array and a `request_id`.
- Normalize every event into the existing view shape and discard unknown keys.
- Validate references as URL-safe identifiers before constructing local routes.
- Validate image URLs as HTTPS and permit only agreed media hosts.
- Treat supplier text as plain text. Templates must continue using
  `esc_html()`, `esc_attr()` and `esc_url()`.
- Reject impossible dates and unknown availability statuses rather than
  guessing.
- Log failures without credentials, full authorization headers or visitor data.

## Normalized event shape

```php
[
    'reference'  => 'faithless-manchester-2027-03-14',
    'title'      => 'Faithless',
    'venue'      => 'Manchester Academy, Manchester',
    'date_label' => 'Sat 14 Mar · 19:00',
    'image'      => 'https://media.example.com/events/faithless-card.jpg',
    'image_alt'  => 'Faithless',
    'status'     => 'Selling fast',
    'status_key' => 'selling-fast',
]
```

`date_label`, `venue` and the visual status class are derived by the adapter
from validated API fields. Availability meaning remains supplier-owned; colour
and presentation remain Orbit-owned.

## Implementation sequence

1. Use the supplied environment base URLs and direct server-side bearer-token
   integration. Obtain credentials securely and ask the supplier whether IP
   allow-listing is mandatory. If essential, assess stable AWS egress and agree
   configuration/costs with the client before supplying outbound addresses.
   Supplier confirms final endpoint
   names/version, featured collection support, media hosts, cache headers and the
   enforcement details of the proposed request limits.
2. Add an `OrbitEventsApiClient` responsible only for authenticated HTTP,
   response limits and conditional requests.
3. Add an `OrbitEventNormalizer` with payload-based tests for valid,
   incomplete and malicious payloads.
4. Add an `OrbitHomepageEventsProvider` for per-collection fresh/stale caches
   and locking.
5. Attach the provider to `orbit_homepage_content`; confirm that production
   renders cached supplier data or omits a collection when no valid cache exists.
6. Add WP-Cron warming and an authenticated WP-CLI cache purge/status command.
7. Expose cache age and last supplier request ID in an admin-only health panel.
