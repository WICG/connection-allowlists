# Connection Allowlists: Embedded Enforcement for iframes and nested contexts

## Authors:

- Brandon Maslen (brandm@microsoft.com), Microsoft Edge

## Participate
- Issue tracker: https://github.com/WICG/connection-allowlists/issues
- Specification PR: https://github.com/WICG/connection-allowlists/pull/29
- Original discussion: https://github.com/WICG/connection-allowlists/issues/1

## Table of Contents

- [Introduction](#introduction)
- [User-Facing Problem](#user-facing-problem)
  - [Goals](#goals)
  - [Non-goals](#non-goals)
- [User research](#user-research)
- [Proposed Approach](#proposed-approach)
  - [Dependencies on non-stable features](#dependencies-on-non-stable-features)
  - [Solving goal 1: requiring an allowlist of an embedded iframe](#solving-goal-1-requiring-an-allowlist-of-an-embedded-iframe)
  - [Solving goal 2: opting in by asserting a matching allowlist](#solving-goal-2-opting-in-by-asserting-a-matching-allowlist)
  - [Solving goal 3: guaranteeing the constraint cannot be escaped by nesting](#solving-goal-3-guaranteeing-the-constraint-cannot-be-escaped-by-nesting)
- [Alternatives considered](#alternatives-considered)
- [Accessibility, Internationalization, Privacy, and Security Considerations](#accessibility-internationalization-privacy-and-security-considerations)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

## Introduction

[Connection Allowlists](https://github.com/WICG/connection-allowlists) let a document or worker
declare the set of network endpoints it is permitted to connect to, as a defense in depth mitigation
against data exfiltration. Today that declaration only constrains a context's own connections, via
its own `Connection-Allowlist` response header. This proposal extends the feature with embedded
enforcement: a parent document can require a connection allowlist of the iframes (and the documents
and workers nested within them) that it embeds. The embedded content opts in, by asserting an
allowlist that is at least as strict as the requirement, by returning an
`Allow-Connection-Allowlist-From` response header naming the embedder, or by being a local-scheme
document such as `about:srcdoc` or `data:`, and otherwise it is not loaded. Once applied, the requirement
is enforced on the embedded document and inherited by its descendants, so embedded content cannot
escape the parent's restriction by nesting. The design mirrors
[CSP Embedded Enforcement](https://www.w3.org/TR/csp-embedded-enforcement/), reusing an enforcement
model the platform already ships.

## User-Facing Problem

A web application that has accepted a strict connection allowlist for itself, as a guarantee that it
cannot exfiltrate data to arbitrary endpoints, currently has no way to extend that guarantee to the
content it embeds. Real pages are rarely a single origin's document: they compose cross-origin
third-party embeds (ad frames, analytics widgets, embedded players, comment widgets) and the
documents and workers nested inside those frames. Any of those embedded contexts can open its own
connections. If they are not held to a comparable restriction, they become a bypass: data the
parent cannot send directly could be handed to a more permissive child frame that ships it onward.
The protection is only as strong as its weakest nested context.

The gap is sharpest for cross-origin embeds. A document fetched from another origin establishes its
own policy container from its own response, so it does not inherit the embedder's connection
allowlist. An embedder therefore has no way today to guarantee that a third-party iframe it embeds is
held to a restriction at least as strict as its own; the framed document's own asserted policy
governs only itself.

### Goals

- Let a parent document **require** a connection allowlist of the content it embeds, including
  cross-origin content that supplies its own policy container and so does not inherit the embedder's
  allowlist, and have that requirement enforced on the embedded document.
- Guarantee the requirement can only **tighten, never loosen**, and is **inherited** by descendants
  and workers, so a constrained subtree cannot escape its constraints by nesting, redirecting, or
  navigating itself.
- Require the embedded content to **opt in**, so an embedder cannot silently impose a policy on
  unwilling cross-origin content.
- Reuse the existing CSP Embedded Enforcement model and machinery rather than inventing a second,
  parallel enforcement mechanism.

### Non-goals

- Defining how a worker's own required allowlist should intersect with an embedder's requirement
  before being passed on for further nesting. The mechanism already threads through workers via the
  policy container, but the intersection semantics are left to follow-up.
- Deciding whether same-origin contexts with different required allowlists may freely communicate
  (for example, via BroadcastChannel, a shared worker, or direct scripting), which may interact with
  the agent cluster key. This is captured as an open question and is out of scope for the initial
  design.
- Changing the base Connection Allowlists threat model. Policies are asserted by servers and
  embedders; an attacker who can already set arbitrary response headers on the framed origin is out
  of scope.
- Providing semantic analysis of whether one URL pattern subsumes another. The strictness comparison
  is a conservative, syntactic set-membership test.

## User research

No formal user research has been conducted. The design is informed by the prior art and developer
demand behind CSP Embedded Enforcement (the `csp` iframe attribute), which addresses the analogous
problem for Content Security Policy, and by the motivating scenarios raised in the
[Connection Allowlists proposal](https://github.com/WICG/connection-allowlists) and
[issue #1](https://github.com/WICG/connection-allowlists/issues/1). Feedback from the WICG review of
the specification PR has shaped the mechanics (see Stakeholder Feedback).

## Proposed Approach

An embedder declares the connection allowlist it requires of a frame with a `connectionAllowlist`
content attribute on the `<iframe>`. The value uses the same grammar as the `Connection-Allowlist`
response header. When the user agent navigates the frame, it communicates the requirement to the
framed document in a `Sec-Required-Connection-Allowlist` request header. The framed document opts in
in one of three ways, and if it does not opt in, the navigation is blocked:

1. It asserts its own `Connection-Allowlist` that **satisfies** (is at least as strict as) the
   requirement.
2. It returns an `Allow-Connection-Allowlist-From` response header naming the embedder's origin (or
   `*`), which is compared byte for byte against the request's origin exactly as
   `Access-Control-Allow-Origin` is compared in a CORS check.
3. It is a local-scheme document (`about:srcdoc`, `data:`), which opts in implicitly because it
   inherits its embedder's policy container and cannot set headers.

When the frame opts in, the required allowlist is added to the framed document's policy container
alongside any allowlist the document asserts for itself, and the requirement is inherited by the
frame's descendants. Because the requirement is stored on the policy container as an unresolved,
serialized string and parsed only when it is applied, the `response-origin` token in the requirement
re-resolves against each frame's own response as the requirement flows down the tree.

The full handshake:

```
Embedder markup:
  <iframe connectionAllowlist='("https://good.example" response-origin)'
          src="https://widget.example/"></iframe>

Request the user agent sends:
  GET / HTTP/1.1
  Host: widget.example
  Sec-Required-Connection-Allowlist: ("https://good.example" response-origin)

Any one of these response headers opts the widget in:
  (a) Connection-Allowlist: ("https://good.example" response-origin)   ; asserts an at-least-as-strict policy
  (b) Allow-Connection-Allowlist-From: https://embedder.example        ; or: *
  (c) delivery from a local scheme (about:srcdoc, data:)               ; implicit

Opted in  -> the required allowlist is enforced on the document and inherited by its descendants.
Not opted -> the frame is blocked.
```

The mechanism reuses the model of
[CSP Embedded Enforcement](https://www.w3.org/TR/csp-embedded-enforcement/): the
`connectionAllowlist` attribute plays the role of the `csp` attribute,
`Sec-Required-Connection-Allowlist` plays the role of `Sec-Required-CSP`,
`Allow-Connection-Allowlist-From` plays the role of `Allow-CSP-From`, and the requirement is threaded
through the policy container in the same way.

### Dependencies on non-stable features

- **Connection Allowlists** (the base feature) is itself incubating in the WICG and is not yet a
  stable, multi-engine feature. This proposal builds directly on it.
- The design reuses concepts and integration points from **CSP Embedded Enforcement**
  (`Sec-Required-CSP`, `Allow-CSP-From`, the required-policy item on the policy container, and the
  target snapshot params and navigation params threading), which is a W3C Working Draft.
- URL pattern matching relies on the **URLPattern** structures used elsewhere in the Connection
  Allowlists spec.

### Solving goal 1: requiring an allowlist of an embedded iframe

An embedder constrains a widget so it can only talk to `https://good.example` and the widget's own
origin. The widget acknowledges the embedder to opt in:

```html
<!-- On https://embedder.example -->
<iframe connectionAllowlist='("https://good.example" response-origin)'
        src="https://widget.example/"></iframe>
```

```http
GET / HTTP/1.1
Host: widget.example
Sec-Required-Connection-Allowlist: ("https://good.example" response-origin)

HTTP/1.1 200 OK
Allow-Connection-Allowlist-From: https://embedder.example
```

The widget loads, and for the lifetime of that document any `fetch()`, WebSocket, WebTransport, or
similar connection it opens is restricted to `https://good.example` and `https://widget.example`.
A request to `https://evil.example` from inside the widget is blocked. Had the response omitted a
valid opt-in, the frame would not have loaded at all.

### Solving goal 2: opting in by asserting a matching allowlist

A document that embeds other content often needs to guarantee that the embedded content cannot be
used to bypass the embedder's own restriction and egress data elsewhere. Embedded content can opt in
without an explicit `Allow-Connection-Allowlist-From` header, simply by asserting a
`Connection-Allowlist` of its own that is at least as strict as what the embedder requires. A widget
that already restricts its own connections therefore satisfies an embedder's requirement
automatically, and the policy is enforced on it:

```html
<!-- On https://embedder.example -->
<iframe connectionAllowlist='("https://good.example" response-origin)'
        src="https://widget.example/"></iframe>
```

```http
GET / HTTP/1.1
Host: widget.example
Sec-Required-Connection-Allowlist: ("https://good.example" response-origin)

HTTP/1.1 200 OK
Connection-Allowlist: (response-origin)
```

The widget asserts `(response-origin)`, which permits a subset of the required
`("https://good.example" response-origin)` set, so it satisfies the requirement and loads. Its own
connections are then held to that stricter policy. Had the widget asserted a broader allowlist the
requirement does not cover, or asserted nothing and returned no `Allow-Connection-Allowlist-From`, it
would be blocked rather than loaded unconstrained, so it cannot be used to egress data around the
embedder's restriction.

### Solving goal 3: guaranteeing the constraint cannot be escaped by nesting

The requirement is inherited by descendants and re-checked on every navigation, and a descendant's
own `connectionAllowlist` attribute is honored only when it is at least as strict as what it
inherited. A constrained frame therefore cannot embed a more permissive grandchild to launder a
connection:

```html
<!-- Parent requires (response-origin) of the child -->
<iframe connectionAllowlist='(response-origin)' src="https://child.example/"></iframe>
```

```html
<!-- Inside https://child.example, an attempt to loosen the requirement is ignored: -->
<!-- the grandchild still inherits (response-origin), and if it does not opt in it is blocked. -->
<iframe connectionAllowlist='("https://anywhere.example" response-origin)'
        src="https://grandchild.example/"></iframe>
```

Because the requirement is stored unresolved and the `response-origin` token re-resolves against each
frame's own response, "restricted to your own origin" continues to mean the correct origin at every
level of the tree.

## Alternatives considered

### A second, standalone enforcement path (not modeled on CSP Embedded Enforcement)

Embedded enforcement could have defined its own attribute-to-header-to-application flow from scratch.

#### Pros
* Freedom to shape the mechanics specifically for connection allowlists.

#### Cons
* Duplicates machinery the platform already has for CSP Embedded Enforcement.
* Two divergent ways to do embedded policy enforcement are harder for implementers and authors to
  reason about, and harder to keep aligned over time.

#### Reason for rejection
Reusing the CSP Embedded Enforcement model (the `Sec-Required-*` request header, the `Allow-*-From`
opt-in, the required-policy item on the policy container, and the snapshot and navigation params
threading) keeps embedded policy enforcement a single, well understood pattern. A follow-up could
even extract the shared integration text into HTML so the two features literally share it.

### Storing the requirement as a resolved connection allowlist object

The requirement could be parsed into a connection allowlist struct at the embedder and stored in
that resolved form.

#### Pros
* Slightly less parsing at the point of use.

#### Cons
* The `response-origin` token would be frozen to the embedder's origin and would be wrong for every
  descendant as the requirement is inherited down the tree.

#### Reason for rejection
The requirement must stay an unresolved, serialized string so that `response-origin` re-resolves
against each frame's own response. This also matches how CSP Embedded Enforcement stores its required
policy (serialized, parsed only when applied).

### Container-policy storage model (fix at navigable creation, reload to change)

The requirement could have been stored in the navigable's container policy (as `allowfullscreen` is),
fixed when the navigable is created, so that an embedder would reload the frame to change it.

#### Pros
* The requirement would be constant for the lifetime of the navigable until an explicit reload.

#### Cons
* Diverges from how the platform already snapshots per-navigation embedding constraints such as
  sandboxing flags.

#### Reason for rejection
The design instead snapshots the requirement from the container at navigation time (the sandbox
model), so it binds every navigation of the navigable, including frame-self-initiated ones, and an
attribute change takes effect on the next navigation. This was discussed with reviewers, who
converged on the snapshot model; the container-policy alternative was withdrawn.

### No opt-in (embedder imposes the requirement unilaterally)

The embedder could impose its requirement on any framed document without requiring consent.

#### Pros
* Simpler; no handshake.

#### Cons
* Lets an embedder silently constrain unwilling cross-origin content, which can be used to probe or
  break it, and is the exact concern that motivated requiring an opt-in for the `csp` iframe
  attribute.

#### Reason for rejection
Opt-in is mandatory. Cross-origin content must assert a satisfying allowlist or return
`Allow-Connection-Allowlist-From`; local-scheme content opts in implicitly because it is effectively
same-origin to the embedder.

### Semantic URL pattern subsumption instead of syntactic set-membership

The strictness comparison could try to determine whether one URL pattern logically subsumes another.

#### Pros
* Could accept more responses that are "obviously" strict enough despite differing pattern syntax.

#### Cons
* Deciding whether one URL pattern fully encompasses another is a hard problem given their syntax.

#### Reason for rejection
A conservative, syntactic set-membership test (a response's asserted patterns must each appear in the
requirement) is sufficient for developers' use cases today and is far simpler to specify and
implement correctly.

## Accessibility, Internationalization, Privacy, and Security Considerations

**Accessibility.** The feature has no user-visible UI and introduces no new user interaction. A
blocked frame surfaces as a load failure with a developer-console message, consistent with other
navigation blocks.

**Internationalization.** The attribute and header values are ASCII structured-header serializations
of URL patterns and origins; there is no human-facing or localizable text, so there are no
internationalization concerns.

**Privacy.** The mechanism only restricts connections; it does not expose new data to the page and
adds no new persistent state or fingerprinting surface. The `Sec-Required-Connection-Allowlist`
request header is visible to the framed server by design, so that the server can decide whether to
opt in, directly analogous to `Sec-Required-CSP`.

**Security.** The threat model is inherited from the base Connection Allowlists proposal: policies
are asserted by servers and embedders, and an attacker who can already set arbitrary response headers
on the framed origin is out of scope. The goal is to cap the connections a constrained context and
its descendants can make.

- **Mandatory opt-in** prevents an embedder from imposing a policy on unwilling cross-origin content.
- **Origin, not URL, comparison.** `Allow-Connection-Allowlist-From` is compared byte for byte
  against the byte-serialized request origin (or `*`), matching the CORS check, with no URL parsing.
- **Compromised-renderer assumption.** The renderer only reflects and length-validates the attribute;
  the browser process makes every trust decision (selecting the requirement, setting the request
  header, resolving opt-in against the final response, storing the requirement, and re-checking it on
  each navigation). A compromised renderer cannot loosen an inherited requirement.
- **No relaxation on inheritance.** The requirement is inherited by descendants and by local-scheme
  documents, and a descendant's attribute is honored only when it is at least as strict, so a
  constrained subtree cannot escape via a more permissive child.
- **Malformed input is conservative.** A malformed asserted `Connection-Allowlist` parses to null and
  does not count as an opt-in.
- **Contexts that cannot be governed are blocked.** A context whose requests are not governed by its
  embedder's policy container (for example, a fenced frame) is blocked when an embedder requires an
  allowlist of it, rather than being allowed to load unconstrained.

One open security question is captured for follow-up: whether same-origin contexts with different
required allowlists may communicate with each other, which may require that the required allowlist be
part of the agent cluster key. This does not need to be resolved for the initial design.

## Stakeholder Feedback / Opposition

- **Microsoft Edge**: Positive; prototyping the feature and authoring the specification change.
- **Google / WICG**: Engaged in the design and development of the feature, with detailed review that
  has shaped the Fetch and HTML integration and the storage model.
- **WebKit**: No signal.
- **Gecko / Firefox**: No signal.

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Domenic Denicola
- Mike West
- Noam Rosenthal

Thanks to the following prior work that influenced this proposal:

- [Connection Allowlists](https://github.com/WICG/connection-allowlists)
- [CSP Embedded Enforcement](https://www.w3.org/TR/csp-embedded-enforcement/)
- [Content Security Policy](https://www.w3.org/TR/CSP3/)
- [Fetch](https://fetch.spec.whatwg.org/)
- [HTML Standard](https://html.spec.whatwg.org/)
