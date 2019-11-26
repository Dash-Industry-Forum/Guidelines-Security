# Periodic re-authorization # {#CPS-PeriodReauth}

In a live DASH presentation the rights of the user can be different for different programs included in the presentation. This chapter describes recommended mechanisms for forcing rights to be re-evaluated at program boundaries.

The user's level of access to content is governed by the issuance (or not) of [=licenses=] with [=content keys=] and the policy configuration carried by the [=licenses=]. The license server is the authority on what rights are assigned to the user. To force re-evaluation of rights, a service must force a new license request to be made. This can be accomplished by:

1. Defining an expiration time on the license.
1. Changing the [=content key=] to one that is not yet available to DASH clients, thereby triggering [[#CPS-activation-workflow|DRM system activation]] for the new [=content key=].

Not every [=DRM system=] supports real-time license expiration - some widely used implementations only check license validity at activation time. Therefore the latter option is a more universally applicable method to force re-evaluation of access rights. As changing the [=content key=] is only possible on DASH period boundaries, live DASH presentations SHOULD create a new period in which content is encrypted with new [=content keys=] to force re-evaluation of user's access rights.

Note: Changing the [=content keys=] does not increase the cryptographic security of content protection. The term *periodic re-authorization* is therefore used here instead of *key rotation*, to maintain focus on the goal and not the mechanism.

# Controlling access rights with a key hierarchy # {#CPS-KeyHierarchy}

Using a key hierarchy allows a single [=content key=] to selectively unlock only a subset of a DASH presentation and apply license policy updates without the need to perform license requests at every program boundary. This mechanism is a specialization of periodic re-authorization for scenarios where license requests at program boundaries are not always desirable or possible.

<figure>
	<img src="Diagrams/KeyHierarchy.png" />
	<figcaption>A key hierarchy establishes a [=DRM system=] specific relationship between a [=root key=] and a set of [=leaf keys=].</figcaption>
</figure>

A key hierarchy defines a multi-level structure of cryptographic keys, instead of a single [=content key=]:

* <dfn>Root keys</dfn> take the place of [=content keys=] in DASH client workflows.
* <dfn>Leaf keys</dfn> are used to encrypt the media samples.

A [=root key=] might not be an actual cryptographic key. Rather, it acts as a reference to identify the set of [=leaf keys=] that protect content. A DASH client requesting a [=license=] for a specific [=root key=] will be interpreted as requesting a [=license=] that makes available all the [=leaf keys=] associated with that [=root key=].

Note: Intermediate layers of cryptographic keys may also exist between [=root keys=] and [=leaf keys=] but such layers are [=DRM system=] specific and only processed by the [=DRM system=], being transparent to the DASH client and the [=media platform=]. To a DASH client, only the [=root keys=] have meaning. To the [=media platform=], only the [=leaf keys=] have meaning.

This layering enables the user's rights to content to be evaluated in two ways:

1. Changing the [=root key=] invokes the full re-evaluation workflow as a new license request must be made by the DASH client.
1. Changing the [=leaf key=] invokes an evaluation of the rights granted by the [=license=] for the [=root key=] and processing of any additional policy attached to the [=leaf key=]. If result of this evaluation indicates the [=leaf key=] cannot be used, the [=DRM system=] will signal playback failure to the DASH client.

Changing the [=root key=] is equivalent to changing the [=content key=] in terms of MPD signaling, requiring a new period to be started. The [=leaf key=] can be changed in any media segment and does not require modification of the MPD. [=Leaf keys=] SHOULD NOT be changed within the same program. Changing [=leaf keys=] on a regular basis does not increase cryptographic security.

Note: A DASH service with a key hierarchy is sometimes referred to as using "internal key rotation".

The mechanism by which a set of [=leaf keys=] is made available based on a request for a [=root key=] is [=DRM system=] specific. Nevertheless, different [=DRM systems=] may be interoperable as long as they can each make available the required set of [=leaf keys=] using their system-specific mechanisms, using the same [=root key=] as the identifier for the same set of [=leaf keys=].

When using a key hierarchy, the [=leaf keys=] are typically delivered in-band in the media segments, using `moof/pssh` boxes, together with additional/updated license policy constraints. The exact implementation is [=DRM system=] specific and transparent to a DASH client.

<figure>
	<img src="Images/KeyHierarchy-NightlyUpdates.png" />
	<figcaption>Different rows indicate [=root key=] changes. Color alternations indicate [=leaf key=] changes. A key hierarchy enables per-program access control even in scenarios where a license request is only performed once per day. The single license request makes available all the [=leaf keys=] that the user is authorized to use during the next epoch.</figcaption>
</figure>

A key hierarchy is useful for broadcast scenarios where license requests are not possible at arbitrary times (e.g. when the system operates by performing nightly [=license=] updates). In such a scenario, this mechanism enables user access rights to be cryptographically enforced at program boundaries, defined on the fly by the service provider, while re-evaluating the access rights during moments when license requests are possible. At the same time, it enables the service provider to supply in-band updates to license policy (when supported by the [=DRM system=]).

Similar functionality could be implemented without a key hierarchy by using a separate [=content key=] for each program and acquiring all relevant [=licenses=] in advance. The advantages of a key hierarchy are:

* Greatly reduced license acquisition traffic and required license storage size, as [=DRM systems=] are optimized for efficient handling of large numbers of [=leaf keys=].
* Ability for the service provider to adjust license policy at any time, not only during license request processing (if in-band policy updates are supported by the [=DRM system=]).

# Use of W3C Clear Key with DASH # {#CPS-AdditionalConstraints-W3C}

Clear Key is a [=DRM system=] defined by W3C in [[!encrypted-media]]. It is intended primarily for client and [=media platform=] development/test purposes and does not perform the content protection and [=content key=] protection duties ordinarily expected from a [=DRM system=]. Nevertheless, in DASH client DRM workflows, it is equivalent to a real [=DRM system=].

A DRM system specific `ContentProtection` descriptor for Clear Key SHALL use the system ID `e2719d58-a985-b3c9-781a-b030af78d30e` and `value="ClearKey1.0"`.

The `dashif:laurl` element SHOULD be used to indicate the license server URL. Legacy content MAY also use an equivalent `Laurl` element from the `http://dashif.org/guidelines/clearKey` namespace, as this was defined in previous versions of this document (the definition is now expanded to also cover non-clearkey scenarios). Clients SHOULD process the legacy element if it exists and `dashif:laurl` does not.

The license request and response format is defined in [[!encrypted-media]].

W3C describes the use of the system ID `1077efec-c0b2-4d02-ace3-3c1e52e2fb4b` in [[!eme-initdata-cenc]] section 4 to indicate that tracks are encrypted with [[!CENC|Common Encryption]]. However, the presence of this "common" `pssh` box does not imply that Clear Key is to be used for decryption. DASH clients SHALL NOT interpret a `pssh` box with the system ID `1077efec-c0b2-4d02-ace3-3c1e52e2fb4b` as an indication that the Clear Key mechanism is to be used (nor as an indication of anything else beyond the use of Common Encryption).

<div class="example">

An example of a Clear Key `ContentProtection` descriptor using `laurl` is as follows.

```xml
<MPD xmlns="urn:mpeg:dash:schema:mpd:2011" xmlns:dashif="https://dashif.org/">
	<Period>
		<AdaptationSet>
			<ContentProtection schemeIdUri="urn:uuid:e2719d58-a985-b3c9-781a-b030af78d30e" value="ClearKey1.0">
				 <dashif:laurl>https://clearKeyServer.foocompany.com</dashif:laurl>
				 <dashif:laurl>file://cache/licenseInfo.txt</dashif:laurl>
			</ContentProtection>
		</AdaptationSet>
	</Period>
</MPD>
```

Parts of the MPD structure that are not relevant for this chapter have been omitted - this is not a fully functional MPD file.
</div>

# XML Schema for DASH-IF MPD extensions # {#CPS-schema}

The namespace for the DASH-IF MPD extensions is `https://dashif.org/`. This document refers to this namespace using the `dashif` prefix. The XML schema of the extensions is:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
    xmlns:dashif="https://dashif.org/"
    targetNamespace="https://dashif.org/">

    <xs:element name="laurl" type="xs:anyURI"/>
    <xs:element name="authzurl" type="xs:anyURI"/>
</xs:schema>
```