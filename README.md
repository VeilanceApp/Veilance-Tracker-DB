# Veilance JSON Tracker Database

Veilance JSON is the tracker-definition database for the Veilance browser
extension. It keeps tracker data separate from the extension itself so domains,
service metadata, and matching rules can be reviewed and updated without
shipping executable detector code.

Each entry describes infrastructure associated with a service. Veilance compares
the hostnames contacted during a visit against these local definitions and shows
a finding when a rule matches.

> A match identifies infrastructure, not intent. It does not automatically mean
> that a website or service is malicious.

## What belongs here

This repository is for declarative network tracker records:

- Service and organization names
- Public service websites
- Network hostnames and their subdomains
- Supported host-anchored network filters
- First-party, third-party, resource-type, and page-host constraints

This repository is not for JavaScript, blocking rules, full request URLs,
cookies, page content, telemetry, or user data. Tracker files are data only and
cannot execute code or modify a website.

## Compatibility

Veilance JSON requires Veilance Browser Extension **v0.4.0 or newer**.

Imported tracker records depend on Veilance's built-in **Network requests**
indicator. If that indicator is disabled, Veilance does not retain the host data
needed to evaluate these rules.

## Use the database

1. Download or clone this repository.
2. Open Veilance and select **Settings**.
3. Confirm that **Network requests** is enabled.
4. Select **Choose indicator folder**.
5. Choose the `trackers` directory, or a smaller category directory.
6. Review the import result. A clean database import should report no errors or
   warnings.

Folder access is one-time. Veilance does not retain the directory handle or scan
the repository in the background. Re-import the folder after downloading an
updated database.

### Current import limits

Veilance v0.4.0 applies the following limits:

- 100 custom entries total, including tracker and signal rules
- The first 100 JSON files found in a selected folder
- 256 KB per JSON file
- 100 `domains` and 100 `filters` per tracker record
- 100-character tracker names
- 320-character descriptions

If the database contains more than 100 entries, import only the category or
curated subset you want to use. Later imports update entries with the same ID
and add new entries until the custom-entry limit is reached.

## Repository layout

Keep one canonical tracker record per file. Category directories make the
database easier to review and allow users to import smaller subsets.

```text
trackers/
├── advertising/
├── analytics/
├── attribution/
├── customer-messaging/
├── session-analytics/
├── social/
└── tag-manager/
└── .../
```

Use lowercase, kebab-case filenames based on the stable organization ID:

```text
trackers/advertising/platform161.json
```

Categories organize the repository; Veilance does not restrict records to the
categories shown above.

## Tracker format

A minimal import-ready record looks like this:

```json
{
  "format": "veilance-json",
  "name": "Platform161",
  "category": "advertising",
  "website_url": "https://platform161.com/",
  "organization": "platform161",
  "domains": [
    "creative-serving.com",
    "p161.net"
  ],
  "filters": [
    "||ads.creative-serving.com^$3p",
    "||p161.net^$3p"
  ]
}
```

### Field reference

| Field | Repository policy | Description |
| --- | --- | --- |
| `format` | Required | Must be `veilance-json`. The extension can infer the format from `domains` or `filters`, but database records should identify it explicitly. |
| `name` | Required | Human-readable tracker or service name. Maximum 100 characters. |
| `organization` | Required unless `id` is used | Stable organization identifier. It becomes the default rule ID. Maximum 100 characters. |
| `id` | Optional | Explicit stable ID. Use lowercase kebab-case without the `custom.` prefix. If omitted, Veilance derives the ID from `organization`, then `name`. |
| `category` | Optional | Display and repository category. Defaults to `Advertising` and is limited to 60 characters. |
| `website_url` | Optional | Valid HTTP or HTTPS URL for the service. Maximum 512 characters. |
| `description` | Optional | Plain-language explanation of what the match means. Veilance generates one when omitted. Maximum 320 characters. |
| `severity` | Optional | `low`, `medium`, or `high`. Defaults to `low`. Use severity conservatively. |
| `defaultEnabled` | Optional | Boolean controlling the initial enabled state. Defaults to `true`. |
| `domains` | Conditionally required | Array of hostname suffixes. At least one valid domain or supported filter is required. Maximum 100 entries. |
| `filters` | Conditionally required | Array of supported host-anchored filter strings. At least one valid domain or supported filter is required. Maximum 100 entries. |

Veilance normalizes imported IDs and prefixes them with `custom.` internally. For
example, `platform161` is stored as `custom.platform161`. Every record in this
repository must resolve to a unique ID.

## Domain matching

A `domains` entry matches the exact hostname and its subdomains. For example:

```json
{
  "domains": ["metrics.example"]
}
```

matches both `metrics.example` and `api.metrics.example`, but not
`notmetrics.example`.

Repository records should use bare, lowercase hostnames:

```text
metrics.example
```

Do not include schemes, paths, ports, or wildcard prefixes:

```text
https://metrics.example/collect    incorrect
*.metrics.example                  incorrect
metrics.example:443                incorrect
```

Avoid overly broad shared infrastructure such as a public cloud, CDN, hosting,
or registrable-domain suffix unless every matching subdomain can be attributed
to the named service. Broad rules create false positives.

## Supported network filters

Veilance supports host-anchored filters in this form:

```text
||hostname.example^$option,option
```

Examples:

| Filter | Meaning |
| --- | --- |
| `||tracker.example^` | Match the host or any of its subdomains. |
| `||tracker.example^$3p` | Match only when the request is third-party. |
| `||tracker.example^$1p` | Match only when the request is first-party. |
| `||tracker.example^$3p,script` | Match third-party script requests. |
| `||tracker.example^$image,~script` | Include image requests and exclude script requests. |
| `||tracker.example^$3p,domain=publisher.example` | Match on `publisher.example` and its subdomains. |
| `||tracker.example^$3p,domain=publisher.example\|~private.publisher.example` | Include the publisher domain but exclude its private subdomain. |

### Party options

- Third-party: `3p` or `third-party`
- First-party: `1p`, `first-party`, `~3p`, or `~third-party`

### Resource-type options

- `document`
- `subdocument`
- `script`
- `stylesheet`
- `image`
- `font`
- `media`
- `object`
- `xmlhttprequest` or `xhr`
- `ping`
- `websocket`
- `other`

Prefix a resource type with `~` to negate it. Options are comma-separated.

### Page-host constraints

Use `domain=` to constrain the page on which a request can match. Separate page
hosts with `|` and prefix an excluded page host with `~`:

```text
domain=allowed.example|news.example|~private.allowed.example
```

Page-host constraints use the same exact-host-or-subdomain boundary as normal
domain matching.

## Unsupported filters

Veilance intentionally does not support rules that require data its local visit
history does not retain. The following filters are skipped:

- Exception filters beginning with `@@`
- URL path-, query-, or fragment-specific filters
- Cosmetic or element-hiding filters
- Regular-expression filters
- Redirect rules
- Unsupported filter modifiers

If a record also contains a valid domain or another supported filter, Veilance
imports the record and reports a warning. If nothing usable remains, the record
fails validation.

This boundary is deliberate. Veilance retains hostnames, party classification,
aggregate request counts, and resource types—not full URLs, request bodies, or
response bodies.

## Accepted document shapes

Canonical database files should contain one tracker object. The importer also
accepts:

- An array of tracker objects
- An object containing a `trackers` array
- An object containing a `rules` array
- An object containing an `indicators` array

A distribution pack may therefore use:

```json
{
  "trackers": [
    {
      "format": "veilance-json",
      "name": "Example Analytics",
      "organization": "example-analytics",
      "category": "analytics",
      "domains": ["analytics.example"]
    }
  ]
}
```

Canonical one-record files remain the source of truth. Generated packs should
not be edited by hand and must stay within Veilance's file and entry limits.

## Add or update a tracker

1. Confirm the service name, organization, and website from a reliable public
   source.
2. Verify that every submitted hostname belongs to or is operated for that
   service.
3. Search the database for an existing `id`, `organization`, domain, and service
   name before creating a new record.
4. Add one lowercase, kebab-case JSON file to the closest category directory.
5. Keep domains and filters unique and sorted where practical.
6. Validate the JSON syntax.
7. Import the containing folder into Veilance v0.4.0 or newer and require zero
   errors and zero warnings.
8. In the change description, include the public evidence used to add, change,
   or remove each hostname.

Do not submit a broad domain solely because one tracking URL happened to use it.
Shared infrastructure must be narrowed to the service-controlled hostname.

## Validation checklist

Before submitting a change, confirm that:

- The file is valid JSON with no comments or trailing commas.
- `format` is exactly `veilance-json`.
- `name` and a stable `organization` or `id` are present.
- At least one valid domain or supported filter remains.
- The resolved ID is unique across the repository.
- Domain suffixes cannot accidentally match unrelated services.
- Filters use only supported anchors and modifiers.
- The record imports with no warnings.
- The built-in **Network requests** indicator is enabled during testing.
- The resulting finding describes infrastructure without claiming malicious
  intent.

If `jq` is available, this command checks every canonical file for basic JSON
syntax errors:

```bash
find trackers -type f -name '*.json' -print0 | xargs -0 -n1 jq empty
```

The Veilance importer remains the authoritative semantic validator.

## Privacy and security

The database contains public tracker metadata only. Do not add credentials,
tokens, private endpoints, customer-specific hostnames, personal information,
captured request content, or any other sensitive data.

Veilance uses these records locally. Importing a tracker database does not
enable telemetry uploads, execute remote code, or turn Veilance into a request
blocker.

## License

Veilance JSON database files are available under the license included in this
repository. Service names and trademarks remain the property of their
respective owners.
