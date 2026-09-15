# Orbit event search API requirements

Status: Supplier connection details recorded; endpoint names and payload contract remain draft for supplier review.

Audience: API provider and website implementation team  
Source design: [Figma — B2C/B2B websites, desktop UI](https://www.figma.com/design/5oN02SUp3cWDasZmfGRnnG/B2C---B2B-%E2%80%A8websites?node-id=3303-80&p=f&t=3JoZdNHK2fzhf0jy-0)

This document is not a final technical specification or an approved
implementation contract. It is a starting point describing the website's needs
and the supplier information received so far. The supplier is responsible for
developing its own technical specification, resolving the open questions and
confirming the final design with the client before implementation. Proposed
endpoints, payloads and behaviours below remain subject to that process.

## 1. Purpose

The Orbit consumer website needs a read-only event discovery API for:

- grouped, type-ahead search from the site header;
- browsing and filtering events;
- displaying standard event results on a map;
- displaying event cards and result counts;
- displaying full event pages;
- handling empty results and incremental “load more” pagination.

This document describes the data the website needs. Endpoint names are proposed and can be mapped to an existing supplier API if the behaviour and fields are equivalent.

Checkout, basket management, ticket inventory reservation, customer accounts and order management are outside this document.

## 2. Working with the existing system

This document is a first guide to the information and behaviour the website needs. It is not intended to prescribe a greenfield backend design or require the supplier to replace working APIs, domain objects or established terminology.

The proposed endpoint names, field names and object boundaries may differ from the current system. Feedback is welcome where an alternative would fit the existing architecture more naturally while still meeting the website requirements. The aim is to produce an API that works well for Orbit without creating unnecessary duplication, awkward translations or competing sources of truth in the backend.

The API provider should identify:

- how its current show/event, venue, taxonomy and availability objects map to this document;
- which proposed fields already exist under different names or in different objects;
- where existing endpoints can satisfy the requirement without introducing new endpoints;
- fields or behaviours that are unavailable, expensive or inconsistent with the current data model;
- established pagination, filtering, caching and search-index conventions that should be retained;
- any suggested response shapes that reduce backend complexity without making the website integration fragile.

Exact payload structure can be adapted by agreement. The important outcomes are that the website receives the required information, relationships and behaviour with clear ownership and reliable performance. Where the supplier proposes a different contract, it should provide an example payload and a short mapping back to the relevant requirement in this document.

A small transformation in WordPress's server-side integration is acceptable when it keeps responsibilities clear. WordPress calls the supplier API directly; a separate middleware/proxy service is not required. Search meaning, availability and other business-critical rules should remain owned by the authoritative backend rather than being reconstructed independently in the browser.

## 3. Experience represented in the design

### Header search

The global search accepts an event, genre, venue or location. While the visitor types, results are grouped into:

- Events — up to five results with image, title and venue summary;
- Venues — venue name;
- Locations — city or area name.

Selecting an event opens its event page. WordPress owns that public route and builds it from the API reference. Selecting a venue or location opens the browse page with that filter applied.

### Browse page

The browse page provides:

- location selection;
- a start and end date;
- sort by Date, Name, Recently added or Trending;
- one primary category: All, Gigs, Festivals, Sport, Theatre, Arts, Food & Drink or Comedy;
- one or more genre tags, including Alternative, Blues, Classical, Country, Dance, Easy Listening, Electronic, Family, Folk, Funk & Soul, Hip Hop, Indie, Jazz, Metal, Pop, Punk, Reggae & Dub, Rock and World;
- reset filters;
- result count and a human-readable result heading;
- event cards;
- a zero-results state;
- cursor-based “load more” pagination.

The website will also display search results on a map. Events with a physical venue therefore require accurate coordinates, but the map uses the same standard event results and location filter as the list.

The genre list should be treated as API-managed data rather than a permanent hard-coded list.

## 4. General API conventions

### Supplier-provided connection details

| Environment | Base URL pattern | Version 1 example |
| --- | --- | --- |
| TEST | `https://orbit-test-api.kaleidaext.co.uk/api/eventsearch/v{version}/` | [TEST v1](https://orbit-test-api.kaleidaext.co.uk/api/eventsearch/v1/) |
| UAT | `https://adminapi.ticketline.dev/api/eventsearch/v{version}/` | [UAT v1](https://adminapi.ticketline.dev/api/eventsearch/v1/) |
| Live | `https://adminapi.orbit.tickets/api/eventsearch/v{version}/` | [Live v1](https://adminapi.orbit.tickets/api/eventsearch/v1/) |

`{version}` is replaced by the agreed version number. Endpoint names will be
confirmed as part of the supplier's technical design. For readability, the
proposed paths below use `/v1/...` as shorthand for
`/api/eventsearch/v1/...`; do not append another `v1` to the configured base URL.
For example, the proposed `/v1/events` endpoint would resolve in TEST to
`https://orbit-test-api.kaleidaext.co.uk/api/eventsearch/v1/events`.

WordPress calls the API directly from its server-side PHP integration. The
browser calls WordPress for public search results; WordPress performs the
authenticated supplier request and returns only validated public data. This
keeps the bearer token private and allows shared caching and request limits.

Every supplier request passes the environment's token in the HTTP header:

```http
Authorization: Bearer <environment-specific-token>
```

Tokens must be provisioned securely and stored in server environment/configuration,
not committed, included in URLs, exposed in browser code or logged. No separate
permissions/scopes model has been specified at present; the supplier has proposed
access control through bearer tokens and IP allow-lists. Whether IP allow-listing
must be mandatory remains to be agreed.

### AWS hosting and IP allow-listing — subject to assessment

WordPress will make server-side PHP requests from AWS. Fixed outbound public IP
addresses are not currently guaranteed, as the hosting infrastructure may scale
or replace instances. Stable egress is possible through suitable AWS networking
such as a public NAT gateway with Elastic IPs, but the current hosting setup has
not been assessed and this may require additional configuration and cost.
Provision of a fixed address or range is therefore subject to hosting assessment,
not an unconditional Project Simply commitment.

The supplier should respond to the following before the technical specification
is finalised:

> Can environment-specific bearer-token authentication be supported without
> mandatory IP allow-listing? If fixed-IP allow-listing is essential, please
> confirm it as a hosting dependency so Project Simply can assess the required
> stable egress configuration and associated costs with the client.

If mandatory allow-listing is agreed, Project Simply must first assess and agree
the hosting changes and costs with the client. Only then can the outbound public
IP addresses for all calling web servers and background/cache-warming workers
be confirmed and supplied for allow-listing in the relevant API environments.
Successful authenticated access must be tested before enabling the integration.

### Supplier-suggested request limits

These defaults can be amended by agreement; they are not final quotas:

| Limit | Suggested default | Purpose |
| --- | --- | --- |
| Sustained | 600 requests per minute | Approximately 10 searches per second continuously |
| Concurrency | 50 simultaneous requests | Bound simultaneous supplier work from the WordPress servers |

The WordPress client should use caching, debounce, request deduplication and
bounded outbound concurrency rather than treating the quota as a target. Apply
the budget across web and background workers, not independently per PHP request.
The supplier still needs to confirm whether limits are per token, IP or
environment, the burst policy, and rate-limit/`Retry-After` headers. Handle `429`
responses without a retry storm; serve eligible last-known-good cached data or
a clear unavailable state, never fabricated event results.

### Response conventions

| Concern | Requirement |
| --- | --- |
| Base path | `/api/eventsearch/v{version}/` on the selected environment host; `/v1` below is shorthand for `/api/eventsearch/v1`. |
| Integration | WordPress calls the supplier directly from server-side PHP; browser requests go through WordPress. |
| Authentication | `Authorization: Bearer <environment-specific-token>`; no reusable secret may be exposed in browser code. |
| Access control | Bearer authentication specified; supplier to confirm whether IP allow-listing is mandatory. Fixed outbound IP provision is subject to AWS hosting assessment and client agreement; no separate permissions/scopes model specified. |
| Request limits | Suggested defaults: 600 requests/minute sustained and 50 simultaneous; adjustable by agreement. |
| Content type | `application/json; charset=utf-8` |
| References | Stable, immutable, URL-safe strings. WordPress uses the event reference to build its local `/events/{reference}` route. |
| Dates | Calendar dates use ISO 8601 `YYYY-MM-DD`. |
| Date/times | RFC 3339 with an explicit UTC offset, plus an IANA timezone on the event. |
| Pagination | Opaque cursor. Results must remain in a deterministic order while paging. |
| Images | HTTPS URL, width, height and alt text. At least one card-ready crop is required. |
| Coordinates | WGS84 decimal latitude and longitude (`EPSG:4326`). Latitude is `-90` to `90`; longitude is `-180` to `180`. |
| Website routes | Owned by WordPress. The discovery API returns references rather than Orbit website URLs. External purchase, seating-map and venue website URLs are allowed where the external party owns them. |
| Nullability | Omit genuinely unavailable optional values or return `null` consistently; do not use empty strings as missing values. |

All endpoints should return a `request_id` that can be supplied to the API provider during support investigations.

## 5. Endpoint summary

| Method | Proposed path | Purpose |
| --- | --- | --- |
| `GET` | `/v1/search/suggestions` | Lightweight grouped type-ahead results for events, venues and locations |
| `GET` | `/v1/events` | Filter definitions, free-text search and filtered browsing |
| `GET` | `/v1/events/{reference}` | Full details for an event |

`/v1/search/suggestions` is intentionally separate because live autocomplete returns several resource types and has a smaller response shape and tighter latency target than event browsing.

`/v1/events` is the single event-list endpoint. Its representation is selected with query parameters:

- `view=filters` returns filter and sort definitions;
- omitting `view` returns the normal event-result list.

## 6. Grouped search suggestions

`GET /v1/search/suggestions`

### Query parameters

| Name | Type | Required | Notes |
| --- | --- | --- | --- |
| `q` | string | Yes | Trimmed user input. Search should be case- and accent-insensitive. Minimum two characters is recommended. |
| `limit_per_group` | integer | No | Default `5`, maximum `10`. |
| `locale` | string | No | BCP 47 language tag; default agreed with Orbit. |

Example:

```http
GET /v1/search/suggestions?q=manchester&limit_per_group=5
```

### Response

```json
{
  "request_id": "req_01JABC123",
  "query": "manchester",
  "groups": {
    "events": [
      {
        "reference": "faithless-manchester-2027-03-14",
        "title": "Faithless",
        "image": {
          "url": "https://media.example.com/events/faithless-card.jpg",
          "width": 800,
          "height": 800,
          "alt": "Faithless"
        },
        "venue_summary": "Manchester Academy, Manchester"
      }
    ],
    "venues": [
      {
        "reference": "manchester-academy",
        "name": "Manchester Academy",
        "location": "Manchester"
      }
    ],
    "locations": [
      {
        "reference": "manchester",
        "name": "Manchester",
        "country_code": "GB"
      }
    ]
  }
}
```

### Behaviour

- Return each entity once within a group.
- Prefer future, publicly visible events.
- Do not return cancelled, private, draft or expired events unless explicitly agreed.
- Ranking must be stable and should combine textual relevance with event popularity and recency.
- An empty group is returned as `[]`; the `groups` keys remain present.
- HTML markup is not required in match labels.

## 7. Filter definitions

`GET /v1/events?view=filters`

This endpoint lets Orbit render current categories, genres and supported sort orders without coupling a release to supplier taxonomy changes.

### Response

```json
{
  "request_id": "req_01JABC124",
  "categories": [
    { "id": "gigs", "label": "Gigs", "position": 10 },
    { "id": "festivals", "label": "Festivals", "position": 20 },
    { "id": "sport", "label": "Sport", "position": 30 }
  ],
  "genres": [
    { "id": "alternative", "label": "Alternative", "position": 10 },
    { "id": "blues", "label": "Blues", "position": 20 },
    { "id": "rock", "label": "Rock", "position": 190 }
  ],
  "sorts": [
    { "id": "date", "label": "Date" },
    { "id": "name", "label": "Name" },
    { "id": "recently_added", "label": "Recently added" },
    { "id": "trending", "label": "Trending" }
  ],
  "defaults": {
    "sort": "trending",
    "page_size": 24
  }
}
```

Taxonomy IDs must remain stable even if a display label changes.

## 8. Search and browse events

`GET /v1/events`

### Query parameters

| Name | Type | Required | Notes |
| --- | --- | --- | --- |
| `q` | string | No | Free-text event search. |
| `artist` | string | No | Free-text artist or performer name. Matching should be case- and accent-insensitive. |
| `location_reference` | string | No | Reference selected from a location suggestion. |
| `venue_reference` | string | No | Reference selected from a venue suggestion. |
| `starts_on_or_after` | date | No | Inclusive local calendar date. |
| `starts_on_or_before` | date | No | Inclusive local calendar date. |
| `category` | string | No | One category ID. Omit for All. |
| `genre` | string[] | No | Repeat the parameter for multiple genre IDs. Matching is OR within this filter. |
| `sort` | enum | No | `date`, `name`, `recently_added` or `trending`. |
| `collection` | string | No | A supplier-managed curated collection, initially `homepage_featured`. Cannot be combined with free-text search. |
| `cursor` | string | No | Opaque cursor from the preceding response. |
| `limit` | integer | No | Default `24`, maximum `48`. |
| `locale` | string | No | BCP 47 language tag. |

Example:

```http
GET /v1/events?artist=Neon%20Parallels&location_reference=manchester&starts_on_or_after=2027-03-10&starts_on_or_before=2027-03-16&category=gigs&genre=rock&genre=indie&sort=date&limit=24
```

### Response

```json
{
  "request_id": "req_01JABC125",
  "total_count": 59,
  "heading": "Gigs in Manchester",
  "applied_filters": {
    "artist": "Neon Parallels",
    "location": { "reference": "manchester", "label": "Manchester" },
    "date_range": {
      "from": "2027-03-10",
      "to": "2027-03-16"
    },
    "category": { "id": "gigs", "label": "Gigs" },
    "genres": [
      { "id": "rock", "label": "Rock" },
      { "id": "indie", "label": "Indie" }
    ],
    "sort": "date"
  },
  "items": [
    {
      "reference": "neon-parallels-2027-03-12",
      "title": "Neon Parallels",
      "image": {
        "url": "https://media.example.com/events/neon-parallels-card.jpg",
        "width": 800,
        "height": 800,
        "alt": "Neon Parallels"
      },
      "starts_at": "2027-03-12T19:00:00+00:00",
      "timezone": "Europe/London",
      "venue": {
        "reference": "academy-2-manchester",
        "name": "Academy 2",
        "city": "Manchester",
        "country_code": "GB",
        "location": {
          "latitude": 53.4641,
          "longitude": -2.2323
        }
      },
      "attendance_mode": "physical",
      "availability": {
        "status": "on_sale",
        "display_label": "On sale 24th Jul"
      },
      "categories": ["gigs"],
      "genres": ["alternative", "indie"]
    }
  ],
  "page": {
    "limit": 24,
    "next_cursor": "eyJhZnRlciI6ImV2ZW50XzI0In0",
    "has_more": true
  }
}
```

### Required event-card fields

Every item must supply enough data to render the card without another API request:

- a stable event reference;
- event title;
- card image and alt text;
- local start date and time;
- venue name and city;
- attendance mode;
- availability/status label, when applicable;
- enough identity data for WordPress to build the local event route.

Physical venue results must include `venue.location`. Online-only events may return `venue: null` and `attendance_mode: "online"`. An event whose coordinates are unknown must not be silently placed at a city-centre fallback coordinate.

The same standardized `/v1/events` items power both the list and map. There is no separate map response. WordPress plots each result using `venue.location`, while `location_reference` remains the user-facing location filter.

The `artist` parameter filters events by a credited artist or performer name rather than by event-title text alone. The provider should apply its authoritative artist data and alias rules instead of having WordPress infer artists from titles.

### Homepage event collections

The homepage reuses the standard event-card representation rather than introducing
a second card schema:

- Featured events: `GET /v1/events?collection=homepage_featured&limit=6`
- Trending gigs: `GET /v1/events?category=gigs&sort=trending&limit=6`
- Trending festivals: `GET /v1/events?category=festivals&sort=trending&limit=6`
- Trending sport: `GET /v1/events?category=sport&sort=trending&limit=6`

`homepage_featured` is an ordered, supplier-managed collection. The response
must preserve its curated order and omit events that are no longer publicly
visible. If an item becomes unavailable, the remaining items move up without
returning a placeholder. The standard `/v1/events` response and event-card
fields still apply.

Orbit owns homepage headings, explanatory copy, buttons and promoter content in
WordPress. The supplier owns event identity, images, dates, venues, availability,
trending order and the featured collection. This keeps business-critical event
facts authoritative while allowing the homepage presentation to evolve
independently.

### Event identity

Each browse result represents one independently selectable show, with its own date, venue and stable event reference. If a tour or production has several dates or venues, each show is returned as a separate event rather than grouped beneath another API resource.

### Zero results

A valid query with no matches returns HTTP `200`, `total_count: 0`, `items: []`, `has_more: false` and `next_cursor: null`. It is not a `404`.

Featured events displayed beneath the zero-results message are managed by WordPress and do not require a discovery API query.

## 9. Event details

`GET /v1/events/{reference}`

The `{reference}` is the stable event reference returned by search and browse responses. WordPress uses the same reference in its local `/events/{reference}` route and calls this API endpoint to render the page.

### Response

```json
{
  "request_id": "req_01JABC127",
  "reference": "10cc-2027-02-26",
  "title": "10cc",
  "summary": "10cc live in concert",
  "description": "Among the most inventive and influential bands in popular music...",
  "images": {
    "hero": {
      "url": "https://media.example.com/events/10cc-hero.jpg",
      "width": 1600,
      "height": 900,
      "alt": "10cc performing live"
    },
    "card": {
      "url": "https://media.example.com/events/10cc-card.jpg",
      "width": 800,
      "height": 800,
      "alt": "10cc"
    }
  },
  "categories": [
    { "id": "gigs", "label": "Gigs" }
  ],
  "genres": [
    { "id": "rock", "label": "Rock" }
  ],
  "restrictions": [
    { "type": "age", "label": "14+" }
  ],
  "additional_information": [
    { "label": "More event info", "value": "Doors 5pm" }
  ],
  "starts_at": "2027-02-26T19:30:00+00:00",
  "doors_at": "2027-02-26T17:00:00+00:00",
  "timezone": "Europe/London",
  "attendance_mode": "physical",
  "availability": {
    "status": "on_sale",
    "display_label": "On sale"
  },
  "venue": {
    "reference": "venue-cymru-theatre",
    "name": "Venue Cymru Theatre",
    "city": "Llandudno",
    "country_code": "GB",
    "address": {
      "line_1": "The Promenade",
      "postal_code": "LL30 1BB"
    },
    "location": {
      "latitude": 53.321,
      "longitude": -3.816
    },
    "capacity": {
      "value": 2500,
      "qualifier": "maximum"
    },
    "contacts": {
      "box_office_phone": "+441234567890",
      "box_office_phone_display": "01234 567890",
      "box_office_email": "boxoffice@venue.example.com"
    },
    "website_url": "https://venue.example.com",
    "seating_map": {
      "method": "embed",
      "label": "View seating map",
      "url": "https://venue.example.com/seating-map"
    },
    "information": {
      "accessibility": "Step-free access and accessible seating are available.",
      "parking": "Public parking is available nearby.",
      "public_transport": "Llandudno station is a 10-minute walk from the venue.",
      "opening_hours": "The box office opens two hours before the event.",
      "ticket_pickup": "Collect prepaid tickets from the box office with photo ID.",
      "facilities": "Bars and a cloakroom are available.",
      "custom_1": {
        "label": "Bag policy",
        "value": "Only small bags are permitted."
      }
    }
  },
  "purchase": {
    "method": "embed",
    "url": "https://tickets.example.com/embed/10cc_2027_02_26"
  }
}
```

### Detail behaviour

- Each event response contains its date, venue, availability and purchase details directly.
- Past, cancelled or private events are excluded by default unless product requirements say otherwise.
- The response must clearly identify an event that is sold out, postponed, rescheduled or off sale.
- Return `404` when the event reference does not exist or is not publicly visible.
- The detail response should support `ETag` or `Last-Modified` validation.

Extended venue fields are optional and belong on the full event response rather than search cards. `box_office_phone` uses E.164 format for calling links, while `box_office_phone_display` contains locally formatted copy. Capacity may vary by seating or event configuration, so `capacity.qualifier` should state whether the figure is `maximum`, `seated`, `standing` or `event_configuration`.

`venue.information` is a structured object rather than a free-form repeater. Its fixed optional fields are `accessibility`, `parking`, `public_transport`, `opening_hours`, `ticket_pickup` and `facilities`; their display labels are owned by Orbit. Two optional custom slots, `custom_1` and `custom_2`, each accept a `label` and `value`. Unused fields and custom slots should be omitted rather than returned as empty strings. Values are plain text unless a safe rich-text format is explicitly agreed.

The design includes a ticket purchase embed supplied by the ticketing provider. `purchase.method` should support at least `embed` and `redirect`. The iframe is the sole source of ticket types, quantities, pricing, fees and live inventory; those values are not duplicated in this discovery API. A native Orbit ticket selector would require a separate transactional inventory and reservation contract.

`venue.seating_map` is optional and represents the venue's general seating plan. `method` should support `embed` and `external_link`. WordPress is responsible for rendering the iframe or link, but the supplier must provide an HTTPS URL from an agreed, allowlisted origin. The provider must also confirm its iframe requirements, including Content Security Policy, `frame-ancestors`, cookies and any required sandbox permissions.

## 10. Availability values

Recommended machine-readable values are:

| Status | Meaning |
| --- | --- |
| `coming_soon` | Publicly visible but sales have not opened |
| `on_sale` | Tickets can currently be purchased |
| `selling_fast` | Supplier-defined low-availability state |
| `sold_out` | No purchasable inventory remains |
| `off_sale` | Sales have closed without implying sold out |
| `cancelled` | Cancelled; excluded from discovery by default |
| `postponed` | Date is under review |
| `rescheduled` | Date has changed |

The API owns the status; the website owns visual colour and styling. `display_label` may provide wording such as “Selling fast”, but Orbit must be able to fall back to its own copy using `status`.

## 11. Errors

Use standard HTTP status codes and a consistent body:

```json
{
  "request_id": "req_01JABC126",
  "error": {
    "code": "invalid_date_range",
    "message": "starts_on_or_before must not be earlier than starts_on_or_after",
    "field": "starts_on_or_before"
  }
}
```

Expected statuses:

- `400` invalid query or unsupported filter value;
- `401` missing or invalid credentials;
- `403` credential is not permitted to access the resource;
- `429` rate limit exceeded, with `Retry-After`;
- `500` unexpected supplier error;
- `503` temporarily unavailable.

## 12. Performance and operational requirements

The connection details and suggested request-limit defaults are recorded in
section 4. The following additional targets remain proposed for supplier confirmation:

- suggestion response: p95 no more than 300 ms at the API edge;
- event search response: p95 no more than 700 ms at the API edge;
- event and availability changes searchable within five minutes;
- 99.9% monthly availability, excluding agreed maintenance;
- gzip or Brotli response compression;
- explicit rate-limit headers and documented quotas;
- search/filter responses safe to cache briefly, with `Cache-Control` and `ETag` where possible;
- no personally identifiable information in requests, responses or URLs;
- separate non-production and production credentials;
- a supplier status page and support/escalation route.

The browser will debounce type-ahead calls to WordPress and cancel stale requests,
but cancellation in the browser does not guarantee cancellation of an outbound
supplier request. WordPress must also bound and deduplicate its outbound calls;
the API must tolerate concurrent requests and results arriving out of order.

## 13. Supplier decisions required

The API provider should confirm or amend the following before implementation:

1. Final endpoint names beneath the supplied `/api/eventsearch/v{version}/` base URLs and the version available for implementation.
2. Provisioning and rotation of environment-specific bearer tokens, and whether token-authenticated access can be supported without mandatory IP allow-listing. If fixed-IP allow-listing is essential, confirm the hosting dependency so Project Simply can assess stable AWS egress and associated costs with the client before committing to addresses. WordPress's direct server-side integration is established.
3. Exact location model: city/region references and the source and accuracy of venue coordinates.
4. Inclusive date-range and timezone behaviour for events spanning midnight or several days.
5. Definitions and tie-break rules for Trending and Recently added.
6. Whether multiple genres use OR matching, as proposed, or AND matching.
7. Source of result headings such as “Gigs in Manchester”: API or frontend.
8. Visibility rules for sold-out, postponed, rescheduled and cancelled events.
9. Search ranking, synonyms, spelling tolerance, minimum query length, and whether `artist` matching supports exact names, partial names and aliases.
10. Maximum page size, caching and index freshness; confirmation or amendment of the suggested 600 requests/minute and 50 simultaneous limits, their enforcement scope, burst policy and rate-limit headers.
11. The stable, URL-safe event reference format used by WordPress routes.
12. Supported locales and countries at launch.
13. Whether purchase uses an embed or redirect, and confirmation that the iframe owns ticket types, pricing, fees and live inventory.
14. The final fixed venue-information labels — currently proposed as Accessibility, Parking, Public transport, Opening hours, Ticket pickup and Facilities — the availability of each field, any length limits for the two custom slots, and whether capacity represents a maximum or event-specific configuration.
15. Seating-map ownership and the domains and browser permissions required for iframe embedding.
