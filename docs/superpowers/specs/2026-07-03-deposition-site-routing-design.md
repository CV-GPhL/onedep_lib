# Deposition Site Routing Design

## Problem

The client currently starts from `DepositConfig.hostname`, but a deposition can belong to a different OneDep site after country/site routing. Redirect handling updates the in-memory API client, but the persisted config still points at the default site. Resuming or modifying an existing deposition can therefore use the wrong host and token scope.

There is also an inconsistency in redirect URL handling: normal requests assign `extras.base_url` directly to the API client base URL, while chunked uploads append `/api/{version}/`. The client needs one canonical way to turn a site returned by the server into a versioned API request URL.

## URL Ownership

The server owns site assignment and user-facing view URLs.

- The server returns the assigned site root, for example `https://dev.wwpdb.org/deposition`.
- The server may also return a fully constructed deposition view URL, for example `https://dev.wwpdb.org/deposition/api/v1/depositions/D_800039/view`.
- The client persists the view URL as opaque metadata. It does not derive API behavior from it.

The client owns API URL construction.

- The client stores/uses the assigned site root and derives `.../api/{version}/` locally.
- The existing `HttpApiClient` version argument remains the authority for whether requests use `v1` or a future version.
- Redirect responses should be normalized through one helper before updating client state.

## Session Model

Add a separate session field for the assigned site root:

```python
site_base_url: str | None = None
```

Keep the existing `site_url` field for the server-provided user-facing deposition view URL.

This avoids overloading `site_url` with two meanings and keeps resumed sessions independent of the default config host.

## Runtime Flow

For new sessions, `deposit_init()` continues to create `HttpApiClient` from `DepositConfig.hostname`.

When the API reports that a deposition belongs to another site:

1. Extract the assigned site root from the redirect response.
2. Normalize it to a site root, accepting both root-like and accidental API-root-like values for compatibility.
3. Update the client's effective site root and derived API base URL.
4. Refresh or exchange credentials for the new site through the existing auth provider path.
5. Persist the new site's tokens under `[auths.<new_site_fqdn>]`.

After deposition creation, persist both:

- `remote_dep_id`
- `site_base_url`
- `site_url`

For resumed sessions, `deposit_resume()` loads the session first. If `session.site_base_url` exists, it creates an effective config/client using that URL so `TokenStore` reads the matching `[auths.<fqdn>]` entry. If not present, it falls back to `config.hostname` for pre-submit and older sessions.

## Error Handling

If redirect is disabled and the server returns an assigned site root, raise `ApiError` with the normalized site root in the message.

If a redirect response lacks a usable site root, keep the current error path from the API response rather than guessing.

If a resumed session has only `site_url` but no `site_base_url`, do not parse the view URL as the primary strategy. Fall back to `config.hostname`. A migration can backfill later if needed, but the implementation should avoid fragile derivation from a view route.

## Testing

Add focused tests for:

- URL normalization from site root to versioned API base.
- Redirect handling in normal JSON requests.
- Redirect handling in chunked uploads using the same normalization path.
- `JsonSessionStore` persistence of `site_base_url` and `site_url`.
- `deposit_resume()` using `site_base_url` to build the resumed API client/auth scope.
- Backward compatibility when older sessions have no `site_base_url`.

## Scope

This change does not introduce a full registry of sites or pre-routing by country. The server remains responsible for routing decisions.
