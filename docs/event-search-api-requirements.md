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

The website will also display search results on a map. The map uses the same
standard event results and location filter as the list, but can plot only Events
whose Venues include coordinates.

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
access control through bearer tokens and IP allow-lists. Project Simply does not
accept mandatory IP allow-listing for this integration.

### Authentication constraint — no fixed source IPs

WordPress will make server-side PHP requests from dynamically hosted AWS
infrastructure. Project Simply will not provide or commit to specific outbound
public IP addresses for web servers or background/cache-warming workers.
IP-based allow-listing must not be a mandatory authentication or access control
for the Orbit website integration.

The supplier must support the proposed environment-specific bearer tokens
without an IP restriction. If bearer-token authentication alone is not
acceptable, the supplier must propose another machine-to-machine authentication
method compatible with dynamic AWS egress and not dependent on fixed client
source IP addresses. The parties must agree and test that method before enabling
the integration.

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
| Access control | Mandatory IP allow-listing is not supported. Use environment-specific bearer tokens without an IP restriction, or another agreed machine-to-machine method that does not depend on fixed client source IP addresses. |
| Request limits | Suggested defaults: 600 requests/minute sustained and 50 simultaneous; adjustable by agreement. |
| Content type | `application/json; charset=utf-8` |
| References | Stable, immutable, URL-safe strings. WordPress uses the event reference to build its local `/events/{reference}` route. |
| Dates | Calendar dates use ISO 8601 `YYYY-MM-DD`. |
| Date/times | RFC 3339 with an explicit UTC offset, plus an IANA timezone on the event. |
| Pagination | Opaque cursor. Results must remain in a deterministic order while paging. |
| Images | HTTPS URL, width, height and alt text. At least one card-ready crop is required. |
| Coordinates | Optional. When supplied, use WGS84 decimal latitude and longitude (`EPSG:4326`). Latitude is `-90` to `90`; longitude is `-180` to `180`. |
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
| `reference` | string[] | No | Repeat to retrieve a set of specifically selected Events efficiently. The caller preserves the required display order. Final batch-query syntax may be adapted by agreement. |
| `location_reference` | string | No | Reference selected from a location suggestion. |
| `venue_reference` | string | No | Reference selected from a venue suggestion. |
| `starts_on_or_after` | date | No | Inclusive local calendar date. |
| `starts_on_or_before` | date | No | Inclusive local calendar date. |
| `category` | string | No | One category ID. Omit for All. |
| `genre` | string[] | No | Repeat the parameter for multiple genre IDs. Matching is OR within this filter. |
| `sort` | enum | No | `date`, `name`, `recently_added` or `trending`. |
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
        "town": "Manchester",
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

WordPress constructs human-readable result headings such as “Gigs in
Manchester” from `total_count` and the labels in `applied_filters`. Presentation
copy and localisation are not owned by the API.

### Required event-card fields

Every item must supply enough data to render the card without another API request:

- a stable event reference;
- event title;
- card image and alt text;
- local start date and time;
- venue name and town;
- attendance mode;
- availability/status label, when applicable;
- enough identity data for WordPress to build the local event route.

Physical venue results include a Venue, but `venue.location` is optional because
Orbit does not hold coordinates for every Venue. Online-only Events may return
`venue: null` and `attendance_mode: "online"`. An Event whose coordinates are
unknown remains available in list results but cannot be plotted and must not be
silently placed at a town-centre or other fallback coordinate.

The same standardized `/v1/events` items power both the list and map. There is
no separate map response. The website plots only results containing a valid
`venue.location`, while `location_reference` remains the user-facing location
filter.

The `artist` parameter filters events by a credited artist or performer name rather than by event-title text alone. The provider should apply its authoritative artist data and alias rules instead of having WordPress infer artists from titles.

### Homepage event queries

The homepage uses the standard Events search/browse endpoint and event-card
representation. It does not require a dedicated Homepage endpoint, a
supplier-managed Homepage collection or Homepage Featured/Trending flags.

The Events endpoint must support these two query patterns:

1. **Category/query** — a category reference, an API-supported sort and a result
   limit, for example
   `GET /v1/events?category=gigs&sort=trending&limit=6`. `trending` is a standard
   Event sort whose ranking calculation and deterministic result order are owned
   by the supplier.
2. **Selected Events** — several stable Event references resolved efficiently in
   one request, for example
   `GET /v1/events?reference=event-a&reference=event-b&limit=6`. The supplier may
   propose equivalent batch-query syntax on the same standard Events API. Each
   returned item must retain its stable reference so the caller can restore the
   required display order.

The same `/v1/events` search capability, optionally preceded by
`/v1/search/suggestions`, must allow public future Events to be found before
their stable references are selected. This does not require an admin-only or
Homepage-specific endpoint.

The supplier API remains authoritative for Event identity, title, images,
dates, venue, status and availability. Homepage composition and presentation
are outside the API contract. If a selected Event is no longer publicly
available, the API must not replace it with another Event or placeholder.

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
  "ends_at": "2027-02-26T22:30:00+00:00",
  "timezone": "Europe/London",
  "attendance_mode": "physical",
  "capacity": {
    "value": 2500,
    "qualifier": "event_configuration"
  },
  "availability": {
    "status": "on_sale",
    "display_label": "Buy Tickets"
  },
  "venue": {
    "reference": "venue-cymru-theatre",
    "name": "Venue Cymru Theatre",
    "address": "The Promenade",
    "town": "Llandudno",
    "county": "Conwy",
    "postcode": "LL30 1BB",
    "location": {
      "latitude": 53.321,
      "longitude": -3.816
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

- Each Event represents one individual show rather than a tour, production or
  grouping of performances.
- Each event response contains its date, venue, availability and purchase details directly.
- Past, cancelled or private events are excluded by default unless product requirements say otherwise.
- The response must clearly identify an event that is sold out, postponed, rescheduled or off sale.
- Return `404` when the event reference does not exist or is not publicly visible.
- The detail response should support `ETag` or `Last-Modified` validation.

The supplier has confirmed that each individual Event has an Event Name, Event
Description, Event Info, Event Date, Door Time, Event End Date and Time, Venue,
capacity, Event Status, one or more Genres, Ticket Price Types and Ticket Shop
configuration. These map respectively to `title`, `description`,
`additional_information`, `starts_at`, `doors_at`, `ends_at`, `venue`,
`capacity`, `availability`, `genres`, the ticket-shop purchase experience and
`purchase`. `additional_information` preserves labelled Event Info without
requiring the website to interpret supplier-specific prose.

Extended venue fields are optional and belong on the full event response rather than search cards. `box_office_phone` uses E.164 format for calling links, while `box_office_phone_display` contains locally formatted copy. Event capacity may vary by seating or configuration, so `capacity.qualifier` should state whether the figure is `maximum`, `seated`, `standing` or `event_configuration`.

Orbit has confirmed that its current Venue fields are Venue Name, address, Town,
County, Postcode, latitude and longitude. These map to `venue.name`,
`venue.address`, `venue.town`, `venue.county`, `venue.postcode` and, when both
coordinates are available, `venue.location`. Coordinates are not mandatory in
the source system. If either coordinate is unavailable or invalid,
`venue.location` must be omitted or returned as `null`; a partial coordinate
pair must not be returned. This confirmation does not establish that venue
contacts, a website, seating-map data or the extended information fields below
exist in Orbit. Those fields must remain optional and be omitted when the
supplier has no authoritative value; the website must handle their absence.

`venue.information` is a structured object rather than a free-form repeater. Its fixed optional fields are `accessibility`, `parking`, `public_transport`, `opening_hours`, `ticket_pickup` and `facilities`; their display labels are owned by Orbit. Two optional custom slots, `custom_1` and `custom_2`, each accept a `label` and `value`. Unused fields and custom slots should be omitted rather than returned as empty strings. Values are plain text unless a safe rich-text format is explicitly agreed.

The design includes a ticket purchase embed supplied by the ticketing provider. `purchase.method` should support at least `embed` and `redirect`. The confirmed Ticket Price Types are consumed within the ticket-shop purchase experience. The iframe is the sole source of ticket types, quantities, pricing, fees and live inventory; those values are not duplicated in this discovery API. A native Orbit ticket selector would require a separate transactional inventory and reservation contract.

`venue.seating_map` is optional and represents the venue's general seating plan. `method` should support `embed` and `external_link`. WordPress is responsible for rendering the iframe or link, but the supplier must provide an HTTPS URL from an agreed, allowlisted origin. The provider must also confirm its iframe requirements, including Content Security Policy, `frame-ancestors`, cookies and any required sandbox permissions.

## 10. Availability values

Orbit has confirmed the following existing Event Status values. The API should
return both a stable machine-readable `status` and its customer-facing
`display_label`:

| Proposed `status` | Confirmed `display_label` |
| --- | --- |
| `on_sale_soon` | On Sale Soon |
| `on_sale` | Buy Tickets |
| `sold_out` | Sold Out |
| `contact_venue` | Contact Venue |
| `off_sale` | Off Sale |
| `postponed` | Postponed |
| `cancelled` | Cancelled |
| `temporarily_unavailable` | Please Try Later |
| `tickets_on_door` | Tickets Available on the Door |
| `check_fanticks` | Check on Fanticks |

The final machine-readable codes may follow an existing supplier convention,
but they must remain stable even if the display wording changes. The API owns
the Event Status meaning; the website owns its visual colour and styling.

Where a status implies an action, the Event detail must include the data needed
to perform it. `on_sale` requires the Event-specific purchase configuration;
`contact_venue` requires the relevant public venue contact details; and
`check_fanticks` requires the appropriate HTTPS destination. A status must not
produce a call to action that has no valid destination.

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

## 13. Project Simply responses to supplier diagram questions

These responses record Project Simply's proposed website-integration position
against the purple questions in the supplier's six workflow diagrams. They are
not the supplier's final technical specification and do not replace client
approval where a question determines product behaviour.

### Common processing

**COMMON-P1 — AWS outbound IP addresses**

Project Simply will not provide or commit to fixed outbound IP addresses.
Mandatory IP allow-listing is not supported. Use environment-specific bearer
tokens without an IP restriction or agree another machine-to-machine
authentication method compatible with dynamic AWS egress.

### Event details

**DETAIL-P1 — Summary, images and alternative text**

A separate `summary` is optional because the current Event page requires the
full description. If supplied, it must be plain text suitable for previews.
Every discoverable Event must provide a card image suitable for an `800 × 800`
crop, its HTTPS URL, intrinsic dimensions and meaningful alternative text. A
larger detail image of at least 1,200 pixels wide is preferred. A focal point is
desirable when one source image must support several crops.

**DETAIL-P2 — Purchase iframe**

The supplier must provide the TEST, UAT and Live iframe origins and document all
Content Security Policy, `frame-ancestors`, cookie, redirect, popup, payment,
`sandbox` and `allow` requirements. The complete purchase flow must work over
HTTPS in current desktop and mobile browsers. Orbit will allow only the minimum
required origins and browser capabilities. Reliance on third-party cookies
should be avoided where possible and explicitly identified where unavoidable.

The supplier must also define how the purchase iframe is bound to the specific
Event or performance being displayed. The preferred contract is for the Event
detail response to return a complete, ready-to-render, event-specific HTTPS URL
in `purchase.url`. If the URL must instead be constructed by the website, the
supplier must document the required Event/performance identifier, how it relates
to the API Event reference, every required path or query parameter, and any
session, signing, expiry and refresh rules. The website must not expose the API
bearer token or any other reusable server credential to the browser. The
supplier must also define the response when the Event cannot currently be
purchased or embedded.

**DETAIL-P3 — Detail cache**

Use a five-minute fresh WordPress TTL and a 30-minute maximum validated
last-known-good window during a supplier failure. Support conditional requests
using `ETag`/`If-None-Match` or `Last-Modified` where possible. An authenticated
purge webhook keyed by Event reference is desirable but not required for the
initial release.

### Event filters

**FILTER-P1 — Definitions and defaults**

The required filters are free text, Artist/Performer, Location, Venue, inclusive
start/end dates, one primary Category and multiple Genres using OR matching.
Supported sorts are Date, Name, Recently added and Trending. Defaults are All
Categories, no Genre or Location restriction, Trending order and 24 results per
page. Filter labels, stable IDs and supported sorts come from the API.

**FILTER-P2 — Definition cache**

Cache filter definitions in WordPress for one hour. Validated last-known-good
definitions may be served for up to 24 hours during supplier failure. A purge
webhook is optional because these definitions should change infrequently.

### Event search and browse

**SEARCH-P1 — Human-readable headings**

WordPress supplies headings such as “Gigs in Manchester”. The API supplies the
structured applied filters, their display labels and the total result count.

**SEARCH-P2 — Page and image sizes**

Use 24 Events per page by default and allow a maximum of 48, with opaque cursor
pagination and deterministic ordering. Event cards require a square image
suitable for an `800 × 800` crop, plus its HTTPS URL, intrinsic dimensions and
meaningful alternative text.

**SEARCH-P3 — Search cache and stale responses**

Cache each normalized query in WordPress for five minutes. A validated
last-known-good result may be served for up to 30 minutes during supplier
failure. If no valid cache exists, return a clear unavailable/empty state; never
substitute design fixtures or invented Event data. Purge/webhook invalidation is
desirable but optional for the initial release.

### Homepage Events — questions superseded

The supplier diagram questions `HOME-P1`, `HOME-P2` and `HOME-P3` are redundant
because they assume supplier-owned Homepage collections and a dedicated
`/homepage` endpoint. That architecture is not required.

Homepage composition is owned by the website rather than the supplier API. A
homepage event row either:

- selects and orders specific Event references; or
- selects a Category, a standard API sort such as `trending`, and a display
  limit.

The existing `/v1/events` search/browse endpoint supplies both modes. Standard
Event-query caching applies to those requests; there is no separate Homepage
collection, Homepage cache contract, Homepage Featured/Trending flag or
Homepage-specific API question for the supplier to resolve.

The current design requests between one and six Events per row through the
standard Events search `limit` parameter, for example
`GET /v1/events?category=festivals&sort=trending&limit=6`. This is standard
search-query behaviour rather than a Homepage-specific collection decision.

### Search suggestions

**SUGGEST-P1 — Spelling tolerance**

Yes: use modest spelling tolerance for Event names, Artists, Venues and
Locations. Rank exact and prefix matches above fuzzy matches. Do not apply fuzzy
matching to stable identifiers or very short input.

**SUGGEST-P2 — Minimum query length**

Use two trimmed characters. WordPress does not request suggestions for zero- or
one-character queries. Return five results per group by default, with a maximum
configurable limit of ten.

**SUGGEST-P3 — Suggestion cache**

Cache normalized suggestion results in WordPress for 60 seconds. Validated stale
results may be used for up to five minutes during a temporary supplier failure.
Webhook invalidation is optional for the initial release; if supported, it
should identify affected Event, Venue, Location or taxonomy tags.

All stale-data policies above apply only to previously validated supplier data.
Local design fixtures are never a production fallback.

## 14. Supplier and client decisions required

Only the following points remain unresolved and require supplier or client
confirmation before implementation. Decisions already stated in this document
are requirements, not questions to be reopened.

1. Provisioning and rotation of environment-specific bearer tokens without IP
   restriction. If bearer tokens alone are unacceptable, the supplier must
   propose a machine-to-machine authentication method that supports dynamic AWS
   egress and does not depend on fixed client source IPs.
2. Inclusive date-range and timezone behaviour for Events spanning midnight or
   several days.
3. Discovery visibility and required website action for each confirmed Event
   Status, particularly `contact_venue`, `postponed`, `cancelled`,
   `temporarily_unavailable`, `tickets_on_door` and `check_fanticks`.
4. Search ranking and synonym behaviour, including whether `artist` matching
   supports exact names, partial names and aliases.
5. Search-index freshness, plus confirmation or amendment of the suggested 600
   requests/minute and 50 simultaneous limits, their enforcement scope, burst
   policy and rate-limit headers.
6. The stable, URL-safe Event reference format used by website routes and saved
   editorial selections.
7. Supported locales and countries at launch.
8. The event-specific purchase iframe URL/identifier contract and URL lifetime,
    plus supplier ownership and availability of seating-map data and the
    permitted domains and browser requirements for both iframe types.
9. The batch-query syntax for resolving a set of editorially selected Event
    references efficiently without one supplier request per Event.
