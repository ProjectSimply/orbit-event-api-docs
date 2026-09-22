# Homepage content and event API integration

Status: Frontend fixture implemented; supplier connection details recorded; API adapter pending final endpoints and credentials

## Ownership

The homepage has two deliberately separate content sources:

| Content | Owner | Initial source |
| --- | --- | --- |
| Homepage rows, source mode and ordering | WordPress | ACF/Flexible Content |
| Selected Event and category references | WordPress | ACF fields populated from the supplier API |
| Event facts displayed by each row | Event supplier | Local-only normalized design fixture, then validated API/cache data |
| Section headings and supporting copy | WordPress | Normalized PHP fixture, later ACF/Flexible Content |
| Promoter CTA and benefit cards | WordPress | Normalized PHP fixture, later Fluid Content/ACF |
| Footer, navigation and newsletter copy | WordPress | Theme fixture, later global options |

Templates consume the normalized payload returned by
`OrbitHomepageContent::get()`. They do not know whether an event came from the
fixture, the supplier API or a cache. ACF owns each row's presentation and source
configuration; the future adapter resolves its category query or selected Event
references into `event_rows[*].items` through the `orbit_homepage_content` filter.

The committed event fixture is enabled only in a local environment (or by the
explicit `ORBIT_USE_DESIGN_FIXTURES` development flag). It is never a
production fallback. When the API integration is enabled, only validated
supplier data or its last-known-good cache may populate event cards.

## Flexible ACF Featured Listings layout

Editors can add one or more Featured Listings blocks to the homepage through
ACF/Flexible Content. Each block should provide:

- heading and supporting copy;
- source type: `category` or `selected_events`;
- supplier category reference when using `category`;
- API-supported sort when using `category`, including `trending`;
- an ordered list of stable Event references when using `selected_events`;
- result limit from one to six for the current design, passed to the standard
  Events search `limit` parameter and also enforced by WordPress;
- optional call-to-action label and link.

In category mode, `trending` is an Event sort option rather than a supplier-owned
homepage collection or Event flag. WordPress decides which category rows use the
sort; the supplier owns the ranking calculation and deterministic result order.

In selected-events mode, the WordPress editor must be able to search public
future Events in the administration area using the same `/v1/events`
search/browse capability as the public website, choose specific shows and
reorder them. WordPress performs the authenticated API call server-side; the
supplier token is not exposed in the admin browser. Store stable references in
ACF. A saved label may be retained solely to make the field intelligible when
editing; it is not authoritative public Event data.

The picker may also use the existing `/v1/search/suggestions` endpoint for
type-ahead before loading full Event results. This reuses the same public search
capabilities and does not introduce an admin-only or homepage-only endpoint.

A category-driven block can, for example, select `festivals` as its category and
`trending` as its order. This is ordinary configuration of the standard Events
query, not a separate Homepage collection.

This model supersedes the supplier diagram questions `HOME-P1`, `HOME-P2` and
`HOME-P3`. Those questions assume supplier-owned Homepage collections and a
dedicated Homepage response, so they are not applicable to the agreed
WordPress/ACF approach. Standard Event-query behaviour and caching still apply.

## Supplier requests

Use the standard event-list response defined in
`event-search-api-requirements.md`:

- Admin Event picker: `/v1/events?q=faithless&limit=10`
- Category row: `/v1/events?category=gigs&sort=trending&limit=6`
- Selected Events: `/v1/events?reference=event-a&reference=event-b&limit=6`

The final batch-query syntax can be adapted to the supplier's existing API, but
WordPress must be able to resolve several selected Event references efficiently
without making one supplier request per Event. WordPress restores the ordered
ACF selection after normalizing the response.

A dedicated `/homepage` endpoint, supplier-owned Featured collection, Homepage
Featured flag or Homepage Trending flag is not required. The existing Events
search/browse endpoint powers the public search, the WordPress Event picker and
category-driven Featured Listings blocks. The repeated-reference filter is part
of that same endpoint and supports efficient rendering of manually selected
shows. Homepage composition is a WordPress editorial concern; the API remains
authoritative for the facts about each selected or queried Event.

These endpoint names are still proposals pending the supplier's technical design.
The `/v1/...` paths above are shorthand for `/api/eventsearch/v1/...` on the
selected environment host. Use the TEST, UAT or Live base URL recorded in the
[API connection guide](event-search-api-requirements.md#4-general-api-conventions);
do not append another version segment to that base URL.

WordPress calls the supplier directly from server-side PHP using
`Authorization: Bearer <environment-specific-token>`. Run event-query refreshes
with bounded concurrency and shared request deduplication. Never send the token
to browser JavaScript or introduce a separate proxy service by default.

The supplier's suggested, adjustable defaults are 600 requests per minute and
50 simultaneous requests. Plan these as a shared outbound budget across
web requests and cache-warming workers; configured homepage rows must not
trigger fresh supplier requests for every visitor. Enforcement scope, burst
behaviour and response headers remain to be confirmed.

The supplier has proposed bearer tokens and IP allow-lists; no separate
permissions/scopes model has been specified. Project Simply will not provide or
commit to fixed outbound IP addresses for AWS-hosted PHP requests. Mandatory IP
allow-listing is not supported. The supplier must accept environment-specific
bearer tokens without an IP restriction or propose another agreed
machine-to-machine authentication method compatible with dynamic AWS egress.
See the [authentication constraint](event-search-api-requirements.md#authentication-constraint--no-fixed-source-ips).

## Cache policy

Cache each normalized category query or selected-reference set independently so
one slow or malformed response does not invalidate the other homepage rows.

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
5. Warm the source queries for all currently configured homepage rows from
   WP-Cron every four minutes. A page request remains allowed to refresh a cold
   cache.
6. Store the supplier `request_id`, response age and result count as metadata
   for logs, but never render them into public HTML.
7. Purge affected API response keys after adapter/schema deployments. Saving the
   homepage immediately changes its ACF composition; unchanged supplier response
   caches can be safely reused by the new row configuration.

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
   integration without IP restriction. Project Simply will not provide fixed
   outbound IP addresses. If bearer tokens alone are unacceptable, agree another
   machine-to-machine authentication method compatible with dynamic AWS egress.
   Supplier confirms final endpoint names/version, category and batch-reference
   query support, media hosts, cache headers and the enforcement details of the
   proposed request limits.
2. Add an `OrbitEventsApiClient` responsible only for authenticated HTTP,
   response limits and conditional requests.
3. Add an `OrbitEventNormalizer` with payload-based tests for valid,
   incomplete and malicious payloads.
4. Add an `OrbitHomepageEventsProvider` for per-query and per-reference-set
   fresh/stale caches and locking.
5. Attach the provider to `orbit_homepage_content`; confirm that production
   renders cached supplier data or omits a row's unavailable Events when no valid
   cache exists.
6. Add WP-Cron warming and an authenticated WP-CLI cache purge/status command.
7. Expose cache age and last supplier request ID in an admin-only health panel.
