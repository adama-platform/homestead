# Permit
Who you are, how you proved it, and what you were granted, per tenant
------

> Status: a first inference, written to be argued with and tuned. Nothing here is ratified.

## The idea

Goober's identity design was large and mostly right: every primary domain is its own realm, role strings scope with `/` (`admin/project_42`), host operators live in a separate `host:` namespace, and agents get tokens through OAuth with a consent screen naming the domain. What went wrong was where it lived: inside the host, beside routing, mail, and storage, with grants and requirements folded into the manifest.

Permit is identity as its own service, with its own SQLite database whose entire job is authentication, identity, and grants per tenant. It answers three questions and nothing else: **who is this** (identity), **how do we know** (authentication), and **what were they granted** (grants). Routing asks those questions on every request; permit never decides what a route does with the answer.

The same token-based grant system that secures identity also secures products. `owner` and `admin` are grants; so are `editor/project_42`, a share link that lets anyone holding it view one document, an API key for a webhook, and an agent's scoped-down token. One mechanism, one audit, one place to revoke.

## What permit holds

| object | example | what it is |
| --- | --- | --- |
| Realm | bobs_plumbing (bobsplumbing.com) | One per tenant, named by its primary domain. Accounts never cross realms. |
| User | 4294967100, alice@example.com | An account in one realm. Its id comes from pond's counter, so it is also a key: `profiles/4294967100`. |
| Credential | a passkey, a verified email, a Google subject | How a user proves who they are. |
| Session | a browser session, an agent's refresh token | A live proof, with an expiry and a device. |
| Grant | (alice, `editor/project_42`, by bob, until 2027-01-01) | A role string held by a subject. |
| Token | an access token, a share link, an API key, an invite | A signed, revocable bearer of a subject or a set of grants. |
| Audit | alice granted `editor/project_42` to carol | Every grant, revoke, sign-in, and operator entry. |

Profile data (names, avatars, addresses) is not permit's. It lives in pond buckets keyed by the user's id, written by product routes like any other object. Permit keeps only what authentication needs.

## Storage

Permit's private signing keys are not in that database: they are held through [cellar](../cellar/IDEA.md), the vault library, and loaded into memory only to sign. Permit keeps one SQLite database for every realm, apart from pond, so identity can be backed up, audited, and secured on its own. Every row is keyed by realm, so one realm can be exported, moved, or deleted along with its tenant. Every row about a user is keyed by their id too, so deleting a user (see [tenants](../common/TENANT.md#deletion)) removes their credentials, sessions, grants, and tokens; the audit keeps that the deletion happened. Permit leases id ranges from pond for user ids, so a user's id is unique everywhere and is a natural key in every bucket.

## Authentication

| method | for |
| --- | --- |
| Passkeys (WebAuthn) | The default for people. Bound to the realm's domain by the browser. |
| Email codes or links | Bootstrapping, recovery, and people without passkeys. Mail is sent by [barker](../barker/IDEA.md), homestead's one mail sender, with the tenant's sending identity. |
| External OIDC | "Sign in with Google" and similar, per realm, with the provider's subject as a credential. |
| OAuth 2.1 as a server | Agents over MCP: authorization code with PKCE, a consent screen naming the domain and the grants asked for. |

Sign-in pages, consent screens, and OAuth endpoints are served by permit under a reserved path on each domain (like `/_permit/`), proxied by barn. Cookies are set for the exact domain only.

## Tokens and grants

A token is a signed statement permit makes, which barn can verify without calling permit:

```json
{
	"realm": "bobs_plumbing",
	"kind": "agent",
	"sub": "4294967100",
	"session": "4294967555",
	"grants": ["editor/project_42"],
	"exp": 1762874000
}
```

| kind | carries | example |
| --- | --- | --- |
| user | A subject; grants are looked up live | A browser session after sign-in. |
| agent | A subject and a subset of the subject's grants, chosen at consent | An MCP agent acting for Alice, limited to project 42. |
| capability | Grants with no subject | A share link that lets anyone holding it read one document; a webhook's API key. |
| invite | Grants to attach to whoever redeems it | "Join project 42 as editor." |
| operator | A `host:` role, valid on any domain, satisfying only host requirements | Support staff, audited per tenant reached. |

**Operators sign in anywhere.** As in goober, a host operator can sign in on any tenant's domain (usually `ops.example.com`) and drop into that tenant. Their token carries only `host:` roles, so it satisfies only host requirements, never a tenant's own grants, and every tenant an operator enters is recorded in the audit with who, when, and from where.

- **Grant lists are short.** Homestead isn't that granular: a user holds fewer than 50 grants at p100, and permit refuses more than 250 per subject. So a token or a cached grant list is small, and checking a requirement is a scan of a short list, not an index lookup.
- **Grants are strings.** `owner`, `admin`, `editor/project_42`. The `/` scopes a role to something inside the realm, prefixed by kind so `project_42` and `workspace_42` never collide. `:` is reserved for `host:` roles, and `_` for system roles like `_signed_in`, which every user holds.
- **Access tokens are short-lived**, and refreshed from a session. Capability and invite tokens can be long-lived, and every token has an id permit can revoke.
- **Barn verifies tokens locally** with permit's public keys, and caches each user's live grants. Permit publishes a feed of revocations and grant changes, like pond's feed, and barn drops what it changed.
- **An agent never holds more than its subject.** An agent token's grants are a subset of the subject's at consent, and it loses any grant the subject loses.

## Requirements

A route's `permit` field is a requirement over grants, from goober's identity design:

```json
"permit": { "any": ["owner", "admin/project_{project_id}", "viewer/project_{project_id}"] }
```

- A request satisfies the requirement if its token's grants, or its subject's live grants, include any clause after substituting the route's typed parameters.
- **A parameter is bound when every clause uses it, or is realm-wide.** Only a bound parameter may choose an object (see barn's iron slots), so a caller can only address objects they hold a grant for.
- Host roles satisfy only clauses that name host roles.

## Operator roles and agent hierarchy

The people and agents who run a site hold grants too. An operator role is `ops/{role}`, in the tenant's own realm: `ops/content`, `ops/deploy`, `ops/data`, `ops/observe`, `ops/mail`, `ops/identity`, and `ops/admin`, which holds every role. The role tree (which roles exist, what each contains, which may delegate) is configuration in rancher, published on its config log, which permit follows. [Porch](../porch/IDEA.md) uses these grants to decide which tools an agent sees at `/mcp/{role}`.

- **Only the identity role gives operator roles.** `ops/` grants come from `permit.control.grant` (an owner or `ops/identity` member acting through rancher's tools) or from consent and delegation below. A route's `gives` can never name an `ops/` role, so no box can make someone an operator.
- **Host operators map in.** `host:admin` holds every role on every `ops.` domain, audited per tenant entered, as before; it still satisfies no tenant clause.
- **The tree only implies within `ops/`.** `ops/admin` holding `ops/content` is the role tree's rule, applied by `porch_roles`; product grants (`editor/project_42`) never imply each other, and `permit_require` stays exact.

**Agents delegate to agents.** A lead agent can hand a narrower job to another agent without a person in the loop, and without that agent ever holding more than the job:

| rule | |
| --- | --- |
| Who may delegate | A token holding a role the tree marks `delegates` (`admin` and `deploy` to start). |
| What a child holds | A subset of its parent's grants, after the tree's containment (a parent with `ops/admin` may give `ops/content`). Never `host:` roles, never product grants the parent lacks. |
| How long | No longer than its parent's session; one hour by default, 24 hours at most. |
| How deep | Three levels: a person's agent, its child, and one grandchild. |
| Revocation | Revoking a session revokes every session below it. A child's token carries its lineage, so barn refuses it the moment any ancestor is revoked, without permit publishing every descendant. |
| Audit | Every mint records parent, child, grants, purpose, and lifetime; every tool call's log line carries the lineage. |

A child is minted by an OAuth 2.0 token exchange (RFC 8693) at `/_permit/oauth/token`: the parent presents its own token and asks for a scope, and gets a new `agent` token with its own session. The consent was the person's, given once to the lead agent; delegation is bounded by it.

## Granting from a route

Products grant and revoke their own roles through routes, so a product's security model uses permit instead of inventing one:

```json
"permit": { "any": ["owner", "admin/project_{project_id}"] },
"gives": ["editor/project_{project_id}", "viewer/project_{project_id}"]
```

A box on that route may ask barn to grant or revoke the listed roles, with the route's bound parameters substituted, so an admin of project 42 can hand out roles in project 42 and nowhere else. The grants are committed as effects in pond's outbox, in the same write as the box's merges, and a barn delivers them to permit (see [pond](../pond/IDEA.md#the-outbox)). So a grant exists if and only if the write that gave it landed, and permit deduplicates redeliveries by the element's idempotency key. Minting a capability or invite token for those roles works the same way.

## What permit is not

- **Not object security.** Which objects and tables a request may touch is routing's job, using permit's answers.
- **Not a profile store.** User data is documents in pond, keyed by the user's id.
- **Not in the data path.** Barn verifies tokens locally; permit is called on sign-in, refresh, consent, and grant changes.

## The invariants

- **One realm per tenant.** Accounts, sessions, and grants never cross realms.
- **A user's id is a pond id.** Unique everywhere, and a key in every bucket.
- **Grants are strings, scoped by `/`.** `:` is for host roles and `_` for system roles; neither can be substituted in.
- **Every token is signed, expiring or revocable, and audited.** Barn can verify it without permit.
- **An agent never holds more than its subject, or its parent.** Sub-agents are subsets, shorter-lived, at most three deep, and revoked with any ancestor.
- **Operator roles are `ops/{role}` grants, and no route can give one.**
- **Private keys live in cellar,** loaded only to sign.
- **Only bound parameters choose objects.** A caller addresses only what a grant names.
- **Grants from a route ride pond's outbox.** A grant exists if and only if its write landed.
- **Grant lists are short.** At most 250 per subject.
- **Permit decides who; routing decides what.**
