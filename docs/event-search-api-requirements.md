# Orbit event search API requirements

Status: Draft for supplier review  
Audience: API provider and website implementation team  
Source design: [Figma — B2C/B2B websites, desktop UI](https://www.figma.com/design/5oN02SUp3cWDasZmfGRnnG/B2C---B2B-%E2%80%A8websites?node-id=3303-80&p=f&t=3JoZdNHK2fzhf0jy-0)

## 1. Purpose

The Orbit consumer website needs a read-only event discovery API for:

- grouped, type-ahead search from the site header;
- browsing and filtering events;
- displaying standard event results on a map;
- displaying event cards and result counts;
- displaying full event and event-series pages;
- supporting event series that have several dates or venues;
- handling empty results and incremental “load more” pagination.

This document describes the data the website needs. Endpoint names are proposed and can be mapped to an existing supplier API if the behaviour and fields are equivalent.

Checkout, basket management, ticket inventory reservation, customer accounts and order management are outside this document.

## 2. Working with the existing system

This document is a first guide to the information and behaviour the website needs. It is not intended to prescribe a greenfield backend design or require the supplier to replace working APIs, domain objects or established terminology.

The proposed endpoint names, field names and object boundaries may differ from the current system. Feedback is welcome where an alternative would fit the existing architecture more naturally while still meeting the website requirements. The aim is to produce an API that works well for Orbit without creating unnecessary duplication, awkward translations or competing sources of truth in the backend.

The API provider should identify:

- how its current event, occurrence/performance, venue, taxonomy, price and availability objects map to this document;
- which proposed fields already exist under different names or in different objects;
- where existing endpoints can satisfy the requirement without introducing new endpoints;
- fields or behaviours that are unavailable, expensive or inconsistent with the current data model;
- established pagination, filtering, caching and search-index conventions that should be retained;
- any suggested response shapes that reduce backend complexity without making the website integration fragile.

Exact payload structure can be adapted by agreement. The important outcomes are that the website receives the required information, relationships and behaviour with clear ownership and reliable performance. Where the supplier proposes a different contract, it should provide an example payload and a short mapping back to the relevant requirement in this document.

A small transformation in WordPress or an integration layer is acceptable when it keeps responsibilities clear. Search meaning, availability, pricing and other business-critical rules should remain owned by the authoritative backend rather than being reconstructed independently in the browser.

## 3. Experience represented in the design

### Header search

The global search accepts an event, genre, venue or location. While the visitor types, results are grouped into:

- Events — up to five results with image, title, venue summary and starting price;
- Venues — venue name;
- Locations — city or area name.

Selecting an event opens either an individual event or an event-series page. WordPress owns that public route and builds it from the API reference. Selecting a venue or location opens the browse page with that filter applied.

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

| Concern | Requirement |
| --- | --- |
| Base path | Versioned HTTPS endpoint, shown below as `/v1` |
| Authentication | Supplier to confirm. Prefer a server-side bearer token; no reusable secret may be exposed in browser code. |
| Content type | `application/json; charset=utf-8` |
| References | Stable, immutable, URL-safe strings. WordPress uses the event reference to build its local `/events/{reference}` route. |
| Dates | Calendar dates use ISO 8601 `YYYY-MM-DD`. |
| Date/times | RFC 3339 with an explicit UTC offset, plus an IANA timezone on the venue/occurrence. |
| Money | Integer minor units plus ISO 4217 currency, for example `1850` and `GBP`. Never use floating-point prices. |
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
| `GET` | `/v1/events/{reference}` | Full details for an event or event series |

`/v1/search/suggestions` is intentionally separate because live autocomplete returns several resource types and has a smaller response shape and tighter latency target than event browsing.

`/v1/events` is the single event-list endpoint. Its representation is selected with query parameters:

- `view=filters` returns filter and sort definitions;
- omitting `view` returns the normal event-result collection.

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
        "reference": "faithless",
        "result_type": "event_series",
        "title": "Faithless",
        "image": {
          "url": "https://media.example.com/events/faithless-card.jpg",
          "width": 800,
          "height": 800,
          "alt": "Faithless"
        },
        "venue_summary": "Various venues",
        "price_from": {
          "amount_minor": 1850,
          "currency": "GBP",
          "includes_fees": false
        },
        "occurrence_count": 6
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
GET /v1/events?location_reference=manchester&starts_on_or_after=2027-03-10&starts_on_or_before=2027-03-16&category=gigs&genre=rock&genre=indie&sort=date&limit=24
```

### Response

```json
{
  "request_id": "req_01JABC125",
  "total_count": 59,
  "heading": "Gigs in Manchester",
  "applied_filters": {
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
      "event_reference": "neon-parallels",
      "result_type": "event_occurrence",
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
      "price_from": {
        "amount_minor": 1850,
        "currency": "GBP",
        "includes_fees": false
      },
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
    "next_cursor": "eyJhZnRlciI6Im9jY18yNCJ9",
    "has_more": true
  }
}
```

### Required event-card fields

Every item must supply enough data to render the card without another API request:

- stable occurrence and parent-event references;
- event title;
- card image and alt text;
- local start date and time;
- venue name and city, or a `venue_summary` for a multi-venue series;
- lowest currently purchasable price and currency, when available;
- availability/status label, when applicable;
- enough identity data for WordPress to build the local event route.

Physical venue results must include `venue.location`. Online-only events may return `venue: null` and `attendance_mode: "online"`. An event whose coordinates are unknown must not be silently placed at a city-centre fallback coordinate.

The same standardized `/v1/events` items power both the list and map. There is no separate map response. WordPress plots each result using `venue.location`, while `location_reference` remains the user-facing location filter.

`price_from` may be `null` for a free, registration-only, not-yet-priced or unavailable event, but the provider must supply a machine-readable reason such as `price_display: "free"`, `"register"`, `"coming_soon"` or `"unavailable"`.

### Multi-date and multi-venue events

When one browse result represents a series rather than a single occurrence:

- set `result_type` to `event_series`;
- include `occurrence_count`;
- include `next_occurrence_at` instead of presenting an arbitrary date as the only date;
- include `venue_summary`, for example `Various venues`;
- use the lowest purchasable price across visible future occurrences;
- return the parent event reference so WordPress can route to the series page where the visitor chooses a date or venue.

### Zero results

A valid query with no matches returns HTTP `200`, `total_count: 0`, `items: []`, `has_more: false` and `next_cursor: null`. It is not a `404`.

Featured events displayed beneath the zero-results message are managed by WordPress and do not require a discovery API query.

## 9. Event details

`GET /v1/events/{reference}`

The `{reference}` is the stable parent event reference returned by search and browse responses. WordPress uses the same reference in its local `/events/{reference}` route and calls this API endpoint to render the page.

An occurrence selected from a listing can be identified with an optional query parameter:

```http
GET /v1/events/10cc?occurrence_reference=10cc-2027-02-26
```

### Response

```json
{
  "request_id": "req_01JABC127",
  "reference": "10cc",
  "event_format": "series",
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
  "selected_occurrence_reference": "10cc-2027-02-26",
  "occurrences": [
    {
      "reference": "10cc-2027-02-26",
      "starts_at": "2027-02-26T19:30:00+00:00",
      "doors_at": "2027-02-26T17:00:00+00:00",
      "timezone": "Europe/London",
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
        "information": [
          {
            "type": "accessibility",
            "label": "Accessibility",
            "value": "Step-free access and accessible seating are available."
          },
          {
            "type": "parking",
            "label": "Parking",
            "value": "Public parking is available nearby."
          }
        ]
      },
      "price_from": {
        "amount_minor": 25000,
        "currency": "GBP",
        "includes_fees": false
      },
      "purchase": {
        "method": "embed",
        "url": "https://tickets.example.com/embed/occ_10cc_2027_02_26"
      }
    }
  ]
}
```

### Detail behaviour

- `event_format` is `single` or `series`.
- A single event normally returns one occurrence; a series returns all currently visible future occurrences.
- `occurrence_reference` selects the date/venue to highlight. It does not change the parent event identity.
- Past, cancelled or private occurrences are excluded by default unless product requirements say otherwise.
- The response must clearly identify a selected occurrence that is sold out, postponed, rescheduled or off sale.
- Return `404` when the event reference does not exist or is not publicly visible.
- Return `400` when `occurrence_reference` does not belong to the event.
- The detail response should support `ETag` or `Last-Modified` validation.

Extended venue fields are optional and belong on the full event response rather than search cards. `box_office_phone` uses E.164 format for calling links, while `box_office_phone_display` contains locally formatted copy. Capacity may vary by seating or event configuration, so `capacity.qualifier` should state whether the figure is `maximum`, `seated`, `standing` or `event_configuration`.

`venue.information` is an extensible array for facts such as accessibility, parking, public transport, opening hours, collection instructions and facilities. `type` is machine-readable, while `label` and `value` are display content. The supplier must document supported types and return plain text unless a safe rich-text format is explicitly agreed.

The design includes a ticket purchase embed supplied by the ticketing provider. `purchase.method` should support at least `embed` and `redirect`. When the embed owns ticket types, quantities, fees and live inventory, those values should not be duplicated in this discovery API. A native Orbit ticket selector would require a separate transactional inventory and reservation contract.

`venue.seating_map` is optional and represents the venue's general seating plan. `method` should support `embed` and `external_link`. WordPress is responsible for rendering the iframe or link, but the supplier must provide an HTTPS URL from an agreed, allowlisted origin. The provider must also confirm its iframe requirements, including Content Security Policy, `frame-ancestors`, cookies and any required sandbox permissions.

If an event can contain too many occurrences to return efficiently, the provider may paginate them through `GET /v1/events/{reference}/occurrences`; this is not required for the initial contract unless real catalogue data demonstrates the need.

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

These are proposed targets for supplier confirmation:

- suggestion response: p95 no more than 300 ms at the API edge;
- event search response: p95 no more than 700 ms at the API edge;
- event, price and availability changes searchable within five minutes;
- 99.9% monthly availability, excluding agreed maintenance;
- gzip or Brotli response compression;
- explicit rate-limit headers and documented quotas;
- search/filter responses safe to cache briefly, with `Cache-Control` and `ETag` where possible;
- no personally identifiable information in requests, responses or URLs;
- separate non-production and production credentials;
- a supplier status page and support/escalation route.

The browser will debounce type-ahead calls and cancel stale requests, but the API must tolerate concurrent requests and results arriving out of order.

## 13. Supplier decisions required

The API provider should confirm or amend the following before implementation:

1. Production and non-production base URLs.
2. Authentication method and whether calls must be proxied through the Orbit backend.
3. Exact location model: city/region references and the source and accuracy of venue coordinates.
4. Inclusive date-range and timezone behaviour for events spanning midnight or several days.
5. Definitions and tie-break rules for Trending and Recently added.
6. Whether multiple genres use OR matching, as proposed, or AND matching.
7. Source of result headings such as “Gigs in Manchester”: API or frontend.
8. Price semantics: fees, VAT, free events and events without a published price.
9. Visibility rules for sold-out, postponed, rescheduled and cancelled events.
10. Search ranking, synonyms, spelling tolerance and minimum query length.
11. Maximum page size, rate limits, caching and index freshness.
12. The stable, URL-safe reference format and the WordPress routing distinction between event-series and occurrence pages.
13. Supported locales, currencies and countries at launch.
14. Whether purchase uses an embed or redirect, and which system owns ticket types and live inventory.
15. Which venue information types are supplied and whether capacity represents a maximum or event-specific configuration.
16. Seating-map ownership and the domains and browser permissions required for iframe embedding.
