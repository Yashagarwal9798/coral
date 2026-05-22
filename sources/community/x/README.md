# X/Twitter

Query public and account-authorized X/Twitter data from Coral.

This source uses the official X API v2 and is read-only. It focuses on user
lookup, recent post search, user timelines, mentions, followers, and following
lists.

## Setup

### Get a Bearer Token

Create or use an approved app in the X Developer Platform and copy an API bearer
token. Access to individual endpoints depends on your X API plan and app
settings.

For the v1 source, use a token that can read public X API v2 user and post
endpoints. For OAuth 2.0 user-context tokens, the minimum scopes are typically:

- `tweet.read`
- `users.read`

### Add the Source

```bash
X_BEARER_TOKEN="..." coral source add --file sources/community/x/manifest.yaml
```

For bundled installs:

```bash
X_BEARER_TOKEN="..." coral source add x
```

## Tables

X API tiers gate endpoints differently. Users on the Free tier will receive
endpoint-not-allowed errors on `recent_search`. See the
[X API access levels](https://developer.x.com/en/docs/x-api/getting-started/about-x-api)
for current details.

| Endpoint | Free | Basic | Pro | Enterprise |
|----------|------|-------|-----|------------|
| `users_by_id` / `users_by_username` | ✓ | ✓ | ✓ | ✓ |
| `recent_search` | ✗ | ✓ | ✓ | ✓ |
| `user_posts` / `user_mentions` | very low limit | ✓ | ✓ | ✓ |
| `followers` / `following` | very low limit | ✓ | ✓ | ✓ |

### `users_by_username`

Fetch a user profile by username. Use this first when you have a handle and
need the stable user ID for timeline, mention, follower, or following queries.

**Requires:** `username`

```sql
SELECT id, name, username, public_metrics
FROM x.users_by_username
WHERE username = 'XDevelopers';
```

### `users_by_id`

Fetch a user profile by numeric user ID.

**Requires:** `id`

```sql
SELECT id, name, username, created_at, verified
FROM x.users_by_id
WHERE id = '2244994945';
```

### `recent_search`

Search recent public posts using X API query syntax. Only returns posts from
the last 7 days. For older posts, use the X full-archive search
(`/2/tweets/search/all`) which requires a higher API tier.

**Requires:** `query`

Optional filters: `start_time`, `end_time`, `since_id`, `until_id`,
`sort_order`.

Common query operators: `from:user` (author), `to:user` (replied to),
`@user` (mentioned), `lang:en` (language), `is:reply` / `is:retweet` (type),
`has:media` / `has:links` (content), `url:example.com` (URL filter),
`-keyword` (negation). See the
[X query building guide](https://developer.x.com/en/docs/x-api/tweets/search/integrate/build-a-query/standard-operators)
for the full operator list.

```sql
SELECT id, author_id, text, created_at, public_metrics
FROM x.recent_search
WHERE query = 'from:XDevelopers'
LIMIT 25;
```

### `user_posts`

List posts from a user's timeline.

**Requires:** `user_id`

Optional filters: `start_time`, `end_time`, `since_id`, `until_id`, `exclude`.
Use `exclude = 'replies'`, `exclude = 'retweets'`, or a comma-separated value
accepted by X.

```sql
SELECT id, text, created_at, public_metrics
FROM x.user_posts
WHERE user_id = '2244994945'
LIMIT 25;
```

### `user_mentions`

List posts mentioning a user.

**Requires:** `user_id`

Optional filters: `start_time`, `end_time`, `since_id`, `until_id`.

```sql
SELECT id, author_id, text, created_at
FROM x.user_mentions
WHERE user_id = '2244994945'
LIMIT 25;
```

### `followers`

List profiles that follow a user.

**Requires:** `user_id`

```sql
SELECT id, username, name, public_metrics
FROM x.followers
WHERE user_id = '2244994945'
LIMIT 50;
```

### `following`

List profiles followed by a user.

**Requires:** `user_id`

```sql
SELECT id, username, name, public_metrics
FROM x.following
WHERE user_id = '2244994945'
LIMIT 50;
```

## Authentication

The source sends:

```http
Authorization: Bearer <X_BEARER_TOKEN>
```

The v1 tables avoid `GET /2/users/me` because that endpoint requires OAuth
user-context authentication. The public lookup/search/timeline endpoints can be
used with a suitable app bearer token or OAuth token, subject to the token's X
API access tier.

## Pagination

The source uses X API cursor pagination:

- request cursor parameter: `pagination_token`
- response cursor path: `meta.next_token`
- page size parameter: `max_results`

Endpoint-specific limits:

- `recent_search`: `10-100`
- `user_posts`: `5-100`
- `user_mentions`: `5-100`
- `followers`: `1-1000`
- `following`: `1-1000`

The manifest uses conservative default page sizes to reduce accidental rate
limit pressure.

## Example Queries

### Find an account ID by username

```sql
SELECT id, name, username
FROM x.users_by_username
WHERE username = 'XDevelopers';
```

### Recent posts matching a product or incident query

```sql
SELECT id, author_id, text, created_at
FROM x.recent_search
WHERE query = 'coral source lang:en'
LIMIT 50;
```

### Recent posts from an account, excluding replies

```sql
SELECT id, text, created_at, public_metrics
FROM x.user_posts
WHERE user_id = '2244994945'
  AND exclude = 'replies'
LIMIT 25;
```

### Accounts mentioning a user

```sql
SELECT author_id, COUNT(*) AS mention_count
FROM x.user_mentions
WHERE user_id = '2244994945'
GROUP BY author_id
ORDER BY mention_count DESC
LIMIT 20;
```

### Follower inventory

```sql
SELECT id, username, name, public_metrics
FROM x.followers
WHERE user_id = '2244994945'
LIMIT 100;
```

## Notes and Limitations

- X API access is tiered. A valid token may still receive provider errors if
  the app's plan does not include a requested endpoint.
- The source is read-only. Posting, deleting, liking, reposting, following,
  unfollowing, bookmarking, list mutation, direct messages, webhooks, and
  streaming are out of scope.
- `created_at` fields are exposed as `Timestamp` columns parsed from ISO 8601.
  This enables natural time-range predicates such as
  `WHERE created_at > NOW() - INTERVAL '24 hours'`.
- Nested fields such as `public_metrics`, `entities`, `attachments`,
  `referenced_tweets`, `context_annotations`, `note_tweet`, `affiliation`, and
  `withheld` are exposed as `Json`.
- Long-form posts (over 280 characters) are supported. The `note_tweet` Json
  column on post tables contains the full text when `text` is truncated.
- Popular accounts can have very large follower/following lists. Use `LIMIT`
  and narrow workflows to avoid rate limits.
- Public X data can still contain personal or sensitive information. Follow X
  Developer Terms and apply care when joining this data with other sources.
