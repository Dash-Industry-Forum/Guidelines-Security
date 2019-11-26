# DASH-IF interoperable license request model # {#CPS-lr-model}

The interactions involved in acquiring [=licenses=] and [=content keys=] in DRM workflows have historically been proprietary, requiring a DASH client to be customized in order to achieve compatibility with specific [=DRM systems=] or license server implementations. This chapter defines an interoperable model to encourage the creation of solutions that do not require custom code in the DASH client in order to play back encrypted content. Use of this model is optional but recommended.

Any conformance statements in this chapter apply to clients and services that opt in to using this model (e.g. a "SHALL" statement means "SHALL, if using this model," and has no effect on implementations that choose to use proprietary mechanisms for license acquisition). The authorization service and license server are considered part of the DASH service.

In performing license acquisition, a DASH client needs to:

1. Be able to prove that the user and device have the right to use the requested [=content keys=].
1. Handle errors in a manner agnostic to the specific [=DRM system=] and license server being used.

This license request model defines a mechanism for achieving both goals. This results in the following interoperability benefits:

* DASH clients can execute DRM workflows without [=solution-specific logic and configuration=].
* Custom code specific to a license server implementation is limited to backend business logic.

These benefits increase in value with the size of the solution, as they reduce the development cost required to offer playback of encrypted content on a wide range of DRM-capable client platforms using different [=DRM systems=], with [=licenses=] potentially served by different license server implementations.

## Proof of authorization ## {#CPS-lr-model-authz}

An <dfn>authorization token</dfn> is a [[!jwt|JSON Web Token]] used to prove to a license server that the caller has the right to use one or more [=content keys=] under certain conditions. Attaching this proof of authorization to a license request is optional, allowing for architectures where a "license proxy" performs authorization checks in a manner transparent to the DASH client.

The basic structural requirements for [=authorization tokens=] are defined by [[!jwt]] and [[!jws]]. This document adds some additional constraints to ensure interoperability. Beyond that, the license server implementation is what defines the contents of the [=authorization token=] (the set of claims it contains), as the data needs to express implementation-specific license server business logic parameters that cannot be generalized.

Note: An [=authorization token=] is divided into a header and body. The distinction between the two is effectively irrelevant and merely an artifact of the [[!jwt|JWT specification]]. License servers may use existing fields and define new fields in both the header and the body.

Implementations SHALL process claims listed in [[!jwt]] 4.1 "Registered Claim Names" when they are present (e.g. `exp` "Expiration Time" and `nbf` "Not Before"). The `typ` header parameter ([[!jwt]] 5.1) SHOULD NOT be present. The `alg` header parameter defined in [[!jws]] SHALL be present.

<div class="example">
JWT headers, specifying digital signature algorithm and expiration time (general purpose fields):

<xmp highlight="json">
{
    "alg": "HS256",
    "exp": "1516239022"
}
</xmp>

JWT body with list of authorized [=content key=] IDs (an example field that could be defined by a license server):

<xmp highlight="json">
{
    "authorized_kids": [
        "1611f0c8-487c-44d4-9b19-82e5a6d55084",
        "db2dae97-6b41-4e99-8210-493503d5681b"
    ]
}
</xmp>

The above data sets are serialized and digitally signed to arrive at the final form of the [=authorization token=]: `eyJhbGciOiJIUzI1NiIsImV4cCI6IjE1MTYyMzkwMjIifQ.eyJhdXRob3JpemVkX2tpZHMiOlsiMTYxMWYwYzgtNDg3Yy00NGQ0LTliMTktODJlNWE2ZDU1MDg0IiwiZGIyZGFlOTctNmI0MS00ZTk5LTgyMTAtNDkzNTAzZDU2ODFiIl19.tBvW6XVPHBRp1JEwItsVnbHwIqoqnQAVQfTV9PGMkIU`
</div>

[=Authorization tokens=] are issued by an authorization service, which is part of a solution's business logic. The authorization service has access to project-specific context that it needs to make its decisions (e.g. the active session, user identification and database of purchases/entitlements). A single authorization service can be used to issue [=authorization tokens=] for multiple license servers, simplifying architecture in solutions where multiple license server vendors are used.

<figure>
	<img src="Diagrams/LicenseRequestModel-BaselineArchitecture.png" />
	<figcaption>Role of the authorization service in DRM workflow related communication.</figcaption>
</figure>

An authorization service SHALL digitally sign any issued [=authorization token=] with an algorithm from the "HMAC with SHA-2 Functions" or "Digital Signature with ECDSA" sets in [[!jwt]]. The HS256 algorithm is recommended as a highly compatible default, as it is a required part of every JWT implementation. License server implementations SHALL validate the digital signature and reject tokens with invalid signatures or tokens using signature algorithms other than those referenced here. The license server MAY further constrain the set of allowed signature algorithms.

Successful signature verification requires that keys/certificates be distributed and trust relationships be established between the signing parties and the validating parties. The specific mechanisms for this are implementation-specific and out of scope of this document.

### Obtaining authorization tokens ### {#CPS-lr-model-authz-requesting}

To obtain an [=authorization token=], a DASH client needs to know the URL of the authorization service. DASH services SHOULD specify the authorization service URL in the MPD using the `dashif:authzurl` element (see [[#CPS-mpd-drm-config]]).

If no authorization service URL is provided by the MPD nor made available at runtime, a DASH client SHALL NOT attach an [=authorization token=] to a license request. Absence of this URL implies that authorization operations are performed in a manner transparent to the DASH client (see [[#CPS-lr-model-deployment]]).

<figure>
	<img src="Images/AuthzTokenSharingAndSelection.png" />
	<figcaption>[=Authorization tokens=] are requested from all authorization services referenced by the selected adaptation sets.</figcaption>
</figure>

DASH clients will use zero or more [=authorization tokens=] depending on the number of authorization service URLs defined for the set of [=content keys=] in use. One [=authorization token=] is requested from each distinct authorization service URL. The authorization service URL is specified individually for each [=DRM system=] and [=content key=] (i.e. it is part of the [=DRM system configuration=]). Services SHOULD use a single [=authorization token=] covering all [=content keys=] and [=DRM systems=] but MAY divide the scope of [=authorization tokens=] if appropriate (e.g. different [=DRM systems=] might use different license server vendors that use mutually incompatible authorization token formats).

Note: Path or query string parameters in the authorization service URL can be used to differentiate between license server implementations (and their respective [=authorization token=] formats).

DASH clients SHOULD cache and reuse [=authorization tokens=] up to the moment specified in the token's `exp` "Expiration Time" claim (defaulting to "never expires"). DASH clients SHALL discard the [=authorization token=] and request a new one if the license server indicates that the [=authorization token=] was rejected (for any reason), even if the "Expiration Time" claim is not present or the expiration time is in the future (see [[#CPS-lr-model-errors]]).

Before requesting an [=authorization token=], a DASH client SHALL take the authorization service URL and add or replace the `kids` query string parameter containing a comma-separated list in ascending alphanumeric order of `default_KID` values obtained from the MPD. This list SHALL contain every `default_KID` for which proof of authorization is requested from this authorization service (i.e. every distinct `default_KID` for which the same set of URLs was specified using `dashif:authzurl` elements).

To request an [=authorization token=], a DASH client SHALL make an HTTP GET request to this modified URL, attaching to the request any standard contextual information used by the underlying platform and allowed by active security policy (e.g. HTTP cookies). This data can be used by the authorization service to identify the user and device and assess their access rights.

Note: For DASH clients operating on the web platform, effective use of the authorization service may require the authorization service to exist on the same origin as the website hosting the DASH client in order to share the session cookies.

If the HTTP response status code indicates a successful result and `Content-Type: text/plain`, the HTTP response body is the authorization token.

<div class="example">
Consider an MPD that specifies the authorization service URL `https://example.com/Authorize` for the [=content keys=] with `default_KID` values `1611f0c8-487c-44d4-9b19-82e5a6d55084` and `db2dae97-6b41-4e99-8210-493503d5681b`.

The generated URL would then be `https://example.com/Authorize?kids=1611f0c8-487c-44d4-9b19-82e5a6d55084,db2dae97-6b41-4e99-8210-493503d5681b` to which a DASH client would make a GET request:

<xmp highlight="xml">
GET /Authorize?kids=1611f0c8-487c-44d4-9b19-82e5a6d55084,db2dae97-6b41-4e99-8210-493503d5681b HTTP/1.1
Host: example.com
</xmp>

Assuming authorization checks pass, the authorization service would return the authorization token in the HTTP response body:

<xmp>
HTTP/1.1 200 OK
Content-Type: text/plain

eyJhbGciOiJIUzI1NiIsImV4cCI6IjE1MTYyMzkwMjIifQ.eyJhdXRob3JpemVkX2tpZHMiOlsiMTYxMWYwYzgtNDg3Yy00NGQ0LTliMTktODJlNWE2ZDU1MDg0IiwiZGIyZGFlOTctNmI0MS00ZTk5LTgyMTAtNDkzNTAzZDU2ODFiIl19.tBvW6XVPHBRp1JEwItsVnbHwIqoqnQAVQfTV9PGMkIU
</xmp>
</div>

If the HTTP response status code indicates a failure, a DASH client needs to examine the response to determine the cause of the failure and handle it appropriately (see [[#CPS-lr-model-errors]]). DASH clients SHOULD NOT treat every failed [=authorization token=] request as a fatal error - if multiple [=authorization tokens=] are used to authorize access to different [=content keys=], it may be that some of them fail but others succeed, potentially still enabling a successful playback experience. The examination of whether playback can successfully proceed SHOULD be performed only once all license requests have been completed and the final set of available [=content keys=] is known. See also [[#CPS-unavailable-keys]].

DASH clients SHALL follow HTTP redirects signaled by the authorization service.

### Issuing authorization tokens ### {#CPS-lr-model-authz-issuing}

The mechanism of performing authorization checks is implementation-specific. Common approaches might be to identify the user from a session cookie, query the entitlements/purchases database to identify what rights are assigned to the user and then assemble a suitable authorization token, taking into account the license policy configuration that applies to the [=content keys=] being requested.

The structure of the [=authorization tokens=] is unconstrained beyond the basic requirements defined in [[#CPS-lr-model-authz]]. Authorization services need to issue tokens that match the expectations of license servers that will be using these tokens. If multiple different license server implementations are served by the same authorization service, the path or query string parameters in the authorization service URL allow the service to identify which output format to use.

<div class="example">
Example authorization token matching the requirements of a hypothetical license server.

JWT headers, specifying digital signature algorithm and expiration time:

<xmp highlight="json">
{
    "alg": "HS256",
    "exp": "1516239022"
}
</xmp>

JWT body with list of authorized [=content key=] IDs (an example field that could be defined by a license server):

<xmp highlight="json">
{
    "authorized_kids": [
        "1611f0c8-487c-44d4-9b19-82e5a6d55084",
        "db2dae97-6b41-4e99-8210-493503d5681b"
    ]
}
</xmp>

Serialized and digitally signed: `eyJhbGciOiJIUzI1NiIsImV4cCI6IjE1MTYyMzkwMjIifQ.eyJhdXRob3JpemVkX2tpZHMiOlsiMTYxMWYwYzgtNDg3Yy00NGQ0LTliMTktODJlNWE2ZDU1MDg0IiwiZGIyZGFlOTctNmI0MS00ZTk5LTgyMTAtNDkzNTAzZDU2ODFiIl19.tBvW6XVPHBRp1JEwItsVnbHwIqoqnQAVQfTV9PGMkIU`
</div>

An authorization service SHALL NOT issue [=authorization tokens=] that authorize the use of [=content keys=] that are not in the set of requested [=content keys=] (as defined in the request's `kids` query string parameter). An authorization service MAY issue [=authorization tokens=] that authorize the use of only a subset of the requested [=content keys=], provided that at least one [=content key=] is authorized. If no [=content keys=] are authorized for use, an authorization service SHALL [[#CPS-lr-model-errors|signal a failure]].

Note: During [=license=] issuance, the license server may further constrain the set of available [=content keys=] (e.g. as a result of examining the [=robustness level=] of the [=DRM system=] implementation requesting the [=license=]). See [[#CPS-unavailable-keys]].

[=Authorization tokens=] SHALL be returned by an authorization service using JWS Compact Serialization [[!jws]] (the `aaa.bbb.ccc` format). The serialized form of an [=authorization token=] SHOULD NOT exceed 5000 characters to ensure that a license server does not reject a license request carrying the token due to excessive HTTP header size.

### Attaching authorization tokens to license requests ### {#CPS-lr-model-authz-using}

[=Authorization tokens=] are attached to license requests using the `Authorization` HTTP request header, signaling the `Bearer` authorization type.

<div class="example">
HTTP request to a hypothetical license server, carrying an [=authorization token=].

<xmp>
POST /AcquireLicense HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIsImV4cCI6IjE1MTYyMzkwMjIifQ.eyJhdXRob3JpemVkX2tpZHMiOlsiMTYxMWYwYzgtNDg3Yy00NGQ0LTliMTktODJlNWE2ZDU1MDg0IiwiZGIyZGFlOTctNmI0MS00ZTk5LTgyMTAtNDkzNTAzZDU2ODFiIl19.tBvW6XVPHBRp1JEwItsVnbHwIqoqnQAVQfTV9PGMkIU

(opaque license request blob from DRM system goes here)
</xmp>
</div>

The same [=authorization token=] MAY be used with multiple license requests but one license request SHALL only carry one [=authorization token=], even if the license request is for multiple [=content keys=]. A DASH client SHALL NOT use [[#CPS-default_KID|content key batching features]] offered by the platform APIs to combine requests for [=content keys=] that require the use of separate [=authorization tokens=].

A DASH client SHALL NOT make license requests for [=content keys=] that are configured as requiring an [=authorization token=] but for which the DASH client has failed to acquire an [=authorization token=].

Note: A [=content key=] requires an [=authorization token=] if there is at least one `dashif:authzurl` in the MPD or if this element is added by [=solution-specific logic and configuration=].

## Problem signaling and handling ## {#CPS-lr-model-errors}

Authorization services and license servers SHOULD indicate an inability to satisfy a request by returning an HTTP response that:

1. Signals a suitable status code (4xx or 5xx).
1. Has a `Content-Type` of `application/problem+json`.
1. Contains a HTTP response body conforming to [[!rfc7807]].

<div class="example">
HTTP response from an authorization service, indicating a rejected [=authorization token=] request because the requested content is not a part of the user's subscriptions.

<xmp highlight="json">
HTTP/1.1 403 Forbidden
Content-Type: application/problem+json

{
    "type": "https://dashif.org/drm-problems/not-authorized",
    "title": "Not authorized",
    "detail": "Your active service plan does not include the channel 'EurasiaSport'.",
    "href": "https://example.com/view-available-subscriptions?channel=EurasiaSport",
    "hrefTitle": "Available subscriptions"
}
</xmp>
</div>

A problem record SHALL contain a short human-readable description of the problem in the `title` field and SHOULD contain a human-readable description, designed to help the reader solve the problem, in the `detail` field.

Note: The `detail` field is intended to be displayed to users of a DASH client, not to developers. The description should be helpful to the user whose device the DASH client is running on.

During [[#CPS-activation-workflow|DRM system activation]], it is possible that multiple failures occur. DASH clients SHOULD be capable of displaying a list of error messages to the end-user and SHOULD deduplicate multiple records with the same `type` (e.g. if an [=authorization token=] expires, this expiration may cause failures when requesting 5 [=content keys=] but should result in at most 1 error message being displayed).

Note: Merely the fact that a problem record was returned does not mean that it needs to be presented to the user or acted upon in other ways. The user may still experience successful playback in the presence of some failed requests. See [[#CPS-unavailable-keys]].

This chapter defines a set of standard problem types that SHOULD be used to indicate the nature of the failure. Implementations MAY extend this set with further problem types if the nature of the failure does not fit into the existing types.

Issue: Let's come up with a good set of useful problem types we can define here, to reduce the set of problem types that must be defined in solution-specific scope.

### Problem type: not authorized to access content ### {#CPS-lr-model-errortype-not-authorized}

Type: `https://dashif.org/drm-problems/not-authorized`

Title: Not authorized

HTTP status code: 403

Used by: authorization service

This problem record SHOULD be returned by an authorization service if the user is not authorized to access the requested [=content keys=]. The `detail` field SHOULD explain why this is so (e.g. their subscription has expired, the requested [=content keys=] are for a movie not in their list of purchases, the content is not available in their geographic region).

The authorization service MAY supply a `href` (string) field on the problem record, containing a URL using which the user can correct the problem (e.g. purchase a missing subscription). If the `href` field is present, a `hrefTitle` (string) field SHALL also be present, containing a title suitable for a hyperlink or button (e.g. "Subscribe"). DASH clients MAY expose this URL and title in their user interface to enable the user to find a quick solution to the problem.

### Problem type: insufficient proof of authorization ### {#CPS-lr-model-errortype-must-supply-usable-authorization-token}

Type: `https://dashif.org/drm-problems/insufficient-proof-of-authorization`

Title: Not authorized

HTTP status code: 403

Used by: license server

This problem record SHOULD be returned by a license server if the proof of authorization (if any) attached to a license request is not sufficient to authorize the use of any of the requested [=content keys=]. The `detail` field SHOULD explain what exactly was the expectation the caller failed to satisfy (e.g. no token provided, token has expired, token is for disabled tenant).

Note: If the authorization token authorizes only a subset of requested keys, a license server does not signal a problem and simply returns only the authorized subset of [=content keys=].

When encountering this problem, a DASH client SHOULD discard whatever authorization token was used, acquire a new [=authorization token=] and retry the license request. If no authorization service URL is available, this indicates a DASH service or client misconfiguration (as clearly, an [=authorization token=] was expected) and the problem SHOULD be escalated for operator attention.

## Possible deployment architectures ## {#CPS-lr-model-deployment}

The interoperable license request model is designed to allow for the use of different deployment architectures in common use today, including those where authorization duties are offloaded to a "license proxy". This chapter outlines some of the possible architectures and how interoperable DASH clients support them.

The baseline architecture assumes that a separate authorization service exists, implementing the logic required to determine which users have the rights to access which content.

<figure>
	<img src="Diagrams/LicenseRequestModel-BaselineArchitecture.png" />
	<figcaption>The baseline architecture with an authorization service directly exposed to the DASH client.</figcaption>
</figure>

While the baseline architecture offers several advantages, in some cases it may be desirable to have the authorization checks be transparent to the DASH client. This need may be driven by license server implementation limitations or by other system architecture decisions.

A common implementation for transparent authorization is to use a "license proxy", which acts as a license server but instead forwards the license request after authorization checks have passed. Alternatively, the license server itself may perform the authorization checks.

<figure>
	<img src="Diagrams/LicenseRequestModel-TransparentArchitecture.png" />
	<figcaption>A transparent authorization architecture performs the authorization checks at the license server, which is often hidden behind a proxy (indistinguishable from a license server to the DASH client).</figcaption>
</figure>

The two architectures can be mixed, with some [=DRM systems=] performing the authorization operations in the license server (or a "license proxy") and others using the authorization service directly. This may be relevant when integrating license servers from different vendors into the same solution.

A DASH client will attempt to contact an authorization service if an authorization service URL is provided either in the MPD or by [=solution-specific logic and configuration=]. If no such URL is provided, it will assume that all authorization checks (if any are required) are performed by the license server (in reality, often a license proxy) and will not attach any proof of authorization.

## Passing a content ID to services ## {#CPS-lr-model-contentid}

The concept of a content ID is sometimes used to identify groups of [=content keys=] based on solution-specific associations. The DRM workflows described by this document do not require this concept to be used but do support it if the solution architecture demands it.

In order to make use of a content ID in DRM workflows, the content ID SHOULD be embedded into authorization service URLs and/or license server URLs (depending on which components are used and require the use of the content ID). This may be done either directly at MPD authoring time (if the URLs and content ID are known at such time) or by [=solution-specific logic and configuration=] at runtime.

Having embedded the content ID in the URL, all DRM workflows continue to operate the same as they normally would, except now they also include knowledge of the content ID in each request to the authorization service and/or license server. The content ID is an addition to the license request workflows and does not replace any existing data.

Advisement: Embedding a content ID allows the service handling the request to use the content ID in its business logic. However, the presence of a content ID in the URL does not invalidate any requirements related to the processing of the `default_KID` values of content keys. For example, an authorization service must still constrain the set of authorized [=content keys=] to a subset of the keys listed in the `kids` parameter ([[#CPS-lr-model-authz-issuing]]).

No generic URL template for embedding the content ID is defined, as the content ID is always a proprietary concept. Recommended options include:

* Query string parameters: `https://example.com/tenants/5341/authorize?contentId=movie865343651`
* Path segments: `https://example.com/moviecatalog-license-api/movie865343651/AcquireLicense`

<div class="example">
[=DRM system configuration=] with the content ID embedded in the authorization service and license server URLs. Each service may use a different implementation-defined URL structure for carrying the content ID.

<xmp highlight="xml">
<ContentProtection
  schemeIdUri="urn:uuid:d0ee2730-09b5-459f-8452-200e52b37567"
  value="AcmeDRM 2.0">
  <cenc:pssh>YmFzZTY0IGVuY29kZWQgY29udGVudHMgb2YgkXBzc2iSIGJveCB3aXRoIHRoaXMgU3lzdGVtSUQ=</cenc:pssh>
  <dashif:authzurl>https://example.com/tenants/5341/authorize?contentId=movie865343651</dashif:authzurl>
  <dashif:laurl>https://example.com/moviecatalog-license-api/movie865343651/AcquireLicense</dashif:laurl>
</ContentProtection>
</xmp>
</div>

The content ID SHOULD NOT be embedded in DRM system specific data structures such as `pssh` boxes, as logic that depends on DRM system specific data structures is not interoperable and often leads to increased development and maintenance costs.