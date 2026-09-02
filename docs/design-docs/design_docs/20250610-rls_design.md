# Milvus Row-Level Security (RLS) Design

- **Feature DRI:** TBD
- **Primary Approver:** TBD
- **Independent Approver:** TBD
- **Design Review:** TBD

## Overview

Row-Level Security (RLS) restricts row-bearing operations with
collection-scoped policies. It is disabled by default and enabled with the
`rls.enabled` collection property.

The caller supplies an application-level `rls_principal`; it is not derived
from the authenticated Milvus username. Policies may reference that principal
and its tags. RLS fails closed when required metadata or an applicable
permissive policy is unavailable.

This release does not support changing `rls.enabled`, overriding the principal
inside a sub-search, atomically upserting and deleting tags in one request, or
enforcing RLS on bulk import and external collection refresh.

## Collection Switch

`rls.enabled` is immutable after collection creation. Policy and principal APIs
are available only on enabled collections. Disabled collections bypass RLS.

Every row-bearing request on an enabled collection must provide a non-empty
top-level `rls_principal` or request `skip_rls=true`. Sub-searches inherit the
top-level decision. A skip is allowed only when authorization is disabled or
the authenticated Milvus user has `SkipRLS` on the collection.

`rls.force=true` rejects `skip_rls=true`. Any future dynamic disable must delete
all RLS metadata and invalidate Proxy state before reporting success; future
dynamic enable must make complete policy metadata available before requests can
observe the enabled state.

## Principal Tags

Principal tags are identified by `(collectionID, principalName)`, exchanged as
a JSON object, stored by RootCoord, and cached by Proxy.

Supported principal tag APIs:

| API | Behavior |
| --- | --- |
| `set_rls_principal_tags` | Incrementally upsert a non-empty tag map for one principal. Supplied keys are added or overwritten, while unspecified tags are preserved; an empty map is rejected. |
| `get_rls_principal_tags` | Return the tag map for one principal. |
| `list_rls_principals` | List principals with tags on one collection. |
| `delete_rls_principal_tags` | Delete selected tag keys. If no keys remain, delete the principal tag record. |

Passing no tag keys deletes the principal. A non-empty list deletes only those
tags, and deleting the last tag also deletes the principal. Deleting a missing
principal succeeds; repeated keys are deduplicated and limits are enforced.

One principal-tag request currently performs either incremental upserts or
deletions, not both. Applications that need both changes issue separate
requests and should revoke tags before granting new ones so a partial failure
fails closed. A future atomic patch API may carry tag upserts and removals in
one collection-serialized mutation.

Policy expressions may reference:

| Variable | Meaning |
| --- | --- |
| `$current_principal` | The request `rls_principal` value. |
| `$current_principal_tags['key']` | The tag value for `key` on the current principal. |

Tag keys cannot contain a single quote because policy tag references use the
single-quoted `$current_principal_tags['key']` syntax and do not support key
escaping.

If an enabled RLS policy references a missing principal tag at runtime, that
policy predicate evaluates to false for the current request.

Principal tag values support `string`, `int64`, and `double`. JSON serialization
may normalize an integral floating-point value to an integer representation, so
authorization depends on value rather than preservation of the original numeric
wire type. String tags match only string fields. Integer and floating-point tags
may match either numeric field family when conversion preserves the value
exactly. Any incompatible or lossy conversion evaluates that policy predicate
to false; Milvus does not coerce between strings and numeric values.

## Row Policies

Each policy belongs to one collection and has a unique `policy_name` in that
collection. `CreateRowPolicy` rejects any existing policy with the same name,
even when the requested definition is identical. `UpdateRowPolicy` updates an
existing policy by name and preserves its internal `policy_id`.

Supported policy APIs:

| API | Behavior |
| --- | --- |
| `create_row_policy` | Create a new named policy. Any existing policy with the same name causes the request to fail. |
| `update_row_policy` | Replace an existing named policy while preserving its `policy_id`. |
| `drop_row_policy` | Drop a named policy. Dropping a missing policy succeeds as an idempotent retry. |
| `list_row_policies` | List policies on one collection. |

Supported actions:

| Action | Uses `using_expr` | Uses `check_expr` |
| --- | --- | --- |
| `query` | Yes | No |
| `query_iterator` | Yes | No |
| `search` | Yes | No |
| `search_iterator` | Yes | No |
| `hybrid_search` | Yes | No |
| `delete` | Yes | No |
| `insert` | No | Yes |
| `upsert` | Yes, for existing rows | Yes, for written rows |

`Get` is a client-side convenience over `Query`; it is not a separate RLS
action.

## Policy Evaluation

RLS is deny-by-default when enabled. If an operation has no applicable policy
for its action and required expression kind, the operation fails with a
privilege error.

Policies are combined by policy type:

```text
(permissive_policy_1 OR permissive_policy_2 OR ...)
AND
(restrictive_policy_1 AND restrictive_policy_2 AND ...)
```

At least one applicable permissive policy is required. If only restrictive
policies match an action, the final predicate is false.

A field referenced by a policy cannot be dropped or changed incompatibly until
that policy is removed or updated.

`CreateRowPolicy` evaluates the prospective complete policy set and rejects it
when any action's combined `using_expr` or `check_expr` exceeds the current
`proxy.rls.maxCombinedExpressionLength`. `UpdateRowPolicy` does not apply this
creation guard because an update may reduce an already oversized policy set.
Proxy checks the complete expression whenever it compiles a runtime predicate,
so an update or a refreshable limit reduction that leaves the set oversized
causes later row-bearing requests to fail with a quota error until the policies
or configuration are adjusted.

For query, search, and delete, the final `using_expr` predicate is merged into
the request plan with logical AND. For insert, the final `check_expr` predicate
is evaluated against each input row in Proxy. For upsert, existing rows must pass
the `using_expr` check, and the final row written by the upsert must pass
`check_expr`.

Local insert and upsert checks use SQL three-valued logic, matching Segcore
filter evaluation. Comparisons involving NULL produce UNKNOWN, boolean
operators preserve UNKNOWN according to SQL semantics, and only a final TRUE
result admits a row.

## Expression Support

RLS expressions intentionally use a restricted subset of Milvus boolean
expressions so Proxy can both merge predicates into plans and locally evaluate
write checks.

Supported expression forms:

- `true` / `false`
- equality comparisons between a top-level scalar field and a literal or
  supported template value
- `in` with literal value lists
- `array_contains`, `array_contains_all`, and `array_contains_any` on primitive
  array fields
- `$current_principal` as a string template value and
  `$current_principal_tags['key']` as a string, int64, or double template value

Each individual `using_expr` or `check_expr` must contain exactly one simple
predicate. Inline `and`, `or`, and boolean `not` are rejected; policy authors
compose predicates through multiple permissive or restrictive policies.

RLS pseudo variables follow the normal Milvus expression-template syntax: only
unquoted variable tokens are converted to template variables. Identical text in
normal or raw string literals remains literal data.

Unsupported forms include vector fields, JSON fields, nested/element-level
fields, system fields, inline boolean composition, ordered comparisons,
field-to-field comparisons, and dynamic functions such as `now()`.

## Metadata And Sync

RootCoord owns and persists policy and principal metadata. Records use the
globally unique collection ID as identity; database and collection names are
descriptive. RootCoord collection metadata keeps complete policies in a
name-keyed map, including their internal IDs, so point operations do not scan
the policy etcd prefix.

RLS mutations use CChannel broadcast tasks with the same
`SharedDBName + ExclusiveCollectionName` resources as collection DDL. Messages
carry a complete post-image or stable drop identity. The ACK callback persists
the mutation, updates RootCoord metadata, and invalidates Proxy caches;
callback failures are retried. This serializes RLS mutation with collection
drop and schema changes. Dropping a collection also removes its RLS metadata.

Proxy caches policies per collection and tags per
`(collectionID, principal)`. It does not preload RLS state. An RLS-enforced
request synchronously loads missing policy or principal state through
`GetRLSMetadata`; refresh failure denies the request.

Policy freshness is checked on use. Principal entries are removed by a
ten-minute scanner after `proxy.rls.metaRefreshInterval` and reloaded on their
next use. Policy changes invalidate the collection policy cache. Principal
updates and deletes invalidate only that principal; creation needs no
notification because missing principals are not cached.

The manager lock protects the collection-state map, the dependency lock
protects refresh dependencies, and each collection-state lock protects policy,
principal, compiled-expression, identity, version, and refresh state. Keyed
collection and principal locks order refresh RPCs against invalidation without
serializing unrelated entries.

RLS messages are eligible for generic CDC replication and replay the same
idempotent ACK callbacks on a secondary. Dedicated RLS CDC compatibility and
recovery validation remains follow-up work.

## Configuration

| Config | Meaning |
| --- | --- |
| `proxy.rls.maxPoliciesPerCollection` | Maximum policies on one collection. |
| `proxy.rls.maxPrincipalsPerCollection` | Maximum principals on one collection. Existing principals remain updatable if the limit is lowered. |
| `proxy.rls.maxTagsPerPrincipal` | Maximum tags on one collection-scoped principal. |
| `proxy.rls.maxExpressionLength` | Maximum length of one `using_expr` or `check_expr`. |
| `proxy.rls.maxCombinedExpressionLength` | Maximum length of the final combined expression. |
| `proxy.rls.maxPolicyNameLength` | Maximum policy name length. |
| `proxy.rls.maxPolicyDescriptionLength` | Maximum policy description length in bytes. |
| `proxy.rls.maxPrincipalNameLength` | Maximum principal name length. |
| `proxy.rls.maxTagKeyLength` | Maximum principal tag key length. |
| `proxy.rls.maxTagValueLength` | Maximum principal tag value length. |
| `proxy.rls.maxArrayLiteralElements` | Maximum literal elements in `in` and `array_contains*` expressions. |
| `proxy.rls.metaRefreshInterval` | Maximum policy-cache age before request-time refresh, and principal-cache age before the periodic scanner removes an entry for refresh on its next use. |

## Compatibility And Rollout

There is no previously released RLS metadata to migrate. Enable RLS only after
all serving Proxy and RootCoord instances understand its API, WAL messages, and
cache invalidation contract. A cluster with RLS-enabled collections must not be
rolled back to a version that cannot enforce RLS.

## Observability And Verification

The initial release uses existing request errors, component logs, and
broadcaster/WAL diagnostics; it adds no dedicated RLS metrics.

The verification plan covers policy combination, tag types, all enforced operations,
bypass authorization, restart recovery, cache refresh and invalidation,
collection/schema DDL ordering, and collection cleanup. Dedicated CDC E2E
validation is a follow-up.

## Alternatives And Follow-ups

RLS uses an explicit application principal instead of Milvus username, lazy
principal loading instead of collection-wide Proxy snapshots, and the existing
broadcast/ACK path instead of direct catalog mutation.

Follow-ups are dynamic enable/disable, atomic tag patch, RootCoord principal
storage and lookup scaling, and dedicated CDC validation.
