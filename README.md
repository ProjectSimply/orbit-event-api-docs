# Orbit event search API requirements

This repository contains the Orbit WordPress project, based on the Project Simply starter, along with a first requirements draft for discussion with the API provider. The proposed endpoint names, fields and object boundaries are not a final implementation contract; feedback on how they map to the existing backend is welcome.

This API guide is not a final technical specification. It is a starting point
for the supplier to develop its own technical specification, resolve the open
questions and confirm the final design with the client before implementation.

[Read the event search API requirements](docs/event-search-api-requirements.md)

[Read the Orbit design-system token map and component plan](docs/orbit-design-system.md)

Status: Supplier connection details recorded; endpoint names and payload contract remain draft for supplier review.

## API connection guide

The supplier has provided the following versioned base URLs. Replace `{version}`
with the agreed API version; the links below show version 1.

| Environment | Version 1 base URL |
| --- | --- |
| TEST | [orbit-test-api.kaleidaext.co.uk](https://orbit-test-api.kaleidaext.co.uk/api/eventsearch/v1/) |
| UAT | [adminapi.ticketline.dev](https://adminapi.ticketline.dev/api/eventsearch/v1/) |
| Live | [adminapi.orbit.tickets](https://adminapi.orbit.tickets/api/eventsearch/v1/) |

The base path is `/api/eventsearch/v{version}/`. Endpoint names will be confirmed
as part of the supplier's technical design; examples in the requirements remain
proposals, not confirmed endpoints.

WordPress calls the supplier API directly from its server-side PHP integration,
using `Authorization: Bearer <environment-specific-token>`. Browser requests go
to WordPress, not directly to the supplier with a reusable secret. Store tokens
in server environment/configuration, never in this repository or public assets.

The supplier has proposed bearer tokens and IP allow-lists; no separate
permissions/scopes model has been specified. WordPress will make server-side
PHP requests from dynamically hosted AWS infrastructure. Project Simply will
not provide or commit to fixed outbound source IP addresses, so IP allow-listing
cannot be a mandatory authentication requirement.

The supplier must support environment-specific bearer-token authentication
without an IP restriction. If bearer tokens alone are insufficient, the supplier
must propose another agreed machine-to-machine authentication method that does
not depend on fixed client source IP addresses.

The supplier's suggested, adjustable defaults are **600 requests per minute**
sustained (approximately 10 per second) and **50 simultaneous requests**. The
WordPress integration must use caching, request deduplication and bounded
concurrency to stay within the shared budget.

See the [detailed connection and operational notes](docs/event-search-api-requirements.md#4-general-api-conventions)
and [homepage caching/integration guide](docs/homepage-content-integration.md).

## Local development

The project is linked through Laravel Valet at [http://orbit.test](http://orbit.test) and uses the local MySQL database `orbit`.

Theme source and npm commands live in `wp-content/themes/ps-starter`.
