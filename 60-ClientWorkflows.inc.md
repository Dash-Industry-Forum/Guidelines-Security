# DRM workflows in DASH clients # {#CPS-client-workflows}

To present encrypted content a DASH client needs to:

1. [[#CPS-selection-workflow|Select a DRM system that is capable of decrypting the content.]]
    * During selection, [[#CPS-system-capabilities|the set of desired DRM system capabilities and the supported capabilities is examined]] to identify suitable candidate systems.
1. [[#CPS-activation-workflow|Activate the selected DRM system and configure it to decrypt content.]]
    * During activation, [[#CPS-license-request-workflow|acquire any missing content keys and the licenses that govern their use]].

A client also needs to take observations at runtime to detect the need for different [=content keys=] to be used (e.g. in live services that change the [=content keys=] periodically) and to detect [=content keys=] becoming unavailable (e.g. due to expiration of access rights).

This chapter defines the recommended DASH client workflows for interacting with [=DRM systems=] in these aspects.

## Capability detection ## {#CPS-system-capabilities}

A [=DRM system=] implemented by a client platform may only support playback of encrypted content that matches certain parameters (e.g. codec type and level). A DASH client needs to detect what capabilities each [=DRM system=] has in order to understand what adaptation sets can be presented and to make an informed choice when multiple [=DRM systems=] can be used.

<div class="example">
A typical [=DRM system=] might offer the following set of capabilities:

* Playback of H.264 High profile up to level 4.0 at "low" robustness
* Playback of H.264 High profile level 4.1 at "low" robustness
* Playback of H.265 Main 10 profile up to level 5.2 at "low" robustness
* Playback of AAC at "low" robustness
* Unique user identification
* Session persistence

</div>

A typical [=media platform=] API such as EME [[!encrypted-media]] will require the DASH client to query the platform by supplying a desired capability set. The [=media platform=] will inspect the desired capabilities, possibly displaying a permissions prompt to the user (if sensitive capabilities such as unique user identification are requested), after which it will return a supported capability set that indicates which of the desired capabilities are available.

<figure>
	<img src="Images/CapabilityNegotiation.png" />
	<figcaption>The DASH client presents a set of desired capabilities for each [=DRM system=] and receives a response with the supported subset.</figcaption>
</figure>

The exact set of capabilities that can be used and the data format used to express them in capability detection APIs are defined by the [=media platform=] API. A DASH client is expected to have a full understanding of the potentially offered capabilities and how they map to parameters in the MPD. Some capabilities may have no relation to the MPD and whether they are required depends entirely on the DASH client or [=solution-specific logic and configuration=].

To detect the set of supported capabilities, a DASH client must first determine the <dfn>required capability set</dfn> for each adaptation set. This is the set of capabilities required to present all the content in a single adaptation set and can be determined based on the following:

1. Content characteristics defined in the MPD (e.g. codecs strings of the representations and the used [=protection scheme=]).
1. [=Solution-specific logic and configuration=] (e.g. what [=robustness level=] is required).

Advisement: Querying for the support of different [=protection schemes=] is currently not possible via the capability detection API of Encrypted Media Extensions [[!encrypted-media]]. To determine the supported [=protection schemes=], a DASH client must assume what the CDM supports. A bug is open on W3C EME and [a pull request exists](https://github.com/w3c/encrypted-media/pull/392) for the ISOBMFF file format bytestream. In future versions of EME, this may become possible.

Some of the capabilities (e.g. required [=robustness level=]) are [=DRM system=] specific. The [=required capability set=] contains the values for all [=DRM systems=].

During [=DRM system=] selection, the [=required capability set=] of each adaptation set is compared with the supported capability set of a [=DRM system=]. As a result of this, each candidate [=DRM system=] is associated with zero or more adaptation sets that can be successfully presented using that [=DRM system=].

It is possible that multiple [=DRM systems=] have the capabilities required to present some or all of the adaptation sets. When multiple candidates exist, the DASH client SHOULD enable [=solution-specific logic and configuration=] to make the final decision.

Note: Some sensible default behavior can be implemented in a generic way (e.g. the [=DRM system=] should be able to enable playback of both audio and video if both media types are present in the MPD). Still, there exist scenarios where the choices seem equivalent to the DASH client and an arbitrary choice needs to be made.

The workflows defined in this document contain the necessary extension points to allow DASH clients to exhibit sensible default behavior and enable [=solution-specific logic and configuration=] to drive the choices in an optimal direction.

## Selecting the DRM system ## {#CPS-selection-workflow}

The MPD describes the [=protection scheme=] used to encrypt content, with the `default_KID` values identifying the [=content keys=] required for playback, and optionally provides the default [=DRM system configuration=] for one or more [=DRM systems=] via `ContentProtection` descriptors. It also identifies the codecs used by each representation, enabling a DASH client to determine the set of required [=DRM system=] capabilities.

Neither an initialization segment nor a media segment is required to select a [=DRM system=]. The MPD is the only component of the presentation used for [=DRM system=] selection.

<div class="example">
An adaptation set encrypted with a key identified by `34e5db32-8625-47cd-ba06-68fca0655a72` using the `cenc` [=protection scheme=].

```xml
<AdaptationSet>
    <ContentProtection
        schemeIdUri="urn:mpeg:dash:mp4protection:2011"
        value="cenc"
        cenc:default_KID="34e5db32-8625-47cd-ba06-68fca0655a72" />
    <ContentProtection
        schemeIdUri="urn:uuid:d0ee2730-09b5-459f-8452-200e52b37567"
        value="FirstDrm 2.0">
        <cenc:pssh>YmFzZTY0IGVuY29kZWQgY29udGVudHMgb2YgkXBzc2iSIGJveCB3aXRoIHRoaXMgU3lzdGVtSUQ=</cenc:pssh>
        <dashif:authzurl>https://example.com/tenants/5341/authorize?mode=firstDRM</dashif:authzurl>
        <dashif:authzurl>https://alternative.example.com/tenants/5341/authorize?mode=firstDRM</dashif:authzurl>
        <dashif:laurl>https://example.com/AcquireLicense</dashif:laurl>
        <dashif:laurl>https://alternative.example.com/AcquireLicense</dashif:laurl>
    </ContentProtection>
    <ContentProtection
        schemeIdUri="urn:uuid:eb3841cf-d7e4-4ec4-a3c5-a8b7f9f4f55b"
        value="SecondDrm 8.0">
        <cenc:pssh>ZXQgb2YgcGxheWFibGUgYWRhcHRhdGlvbiBzZXRzIG1heSBjaGFuZ2Ugb3ZlciB0aW1lIChlLmcuIGR1ZSB0byBsaWNlbnNlIGV4cGlyYXRpb24gb3IgZHVl</cenc:pssh>
        <dashif:authzurl>https://example.com/tenants/5341/authorize?mode=secondDRM</dashif:authzurl>
    </ContentProtection>
    <Representation mimeType="video/mp4" codecs="avc1.64001f" width="640" height="360" />
    <Representation mimeType="video/mp4" codecs="avc1.640028" width="852" height="480" />
</AdaptationSet>
</xmp>
```

The MPD provides [=DRM system configuration=] for [=DRM systems=]:

* For `FirstDRM`, the MPD provides complete [=DRM system configuration=], including the optional `dashif:authzurl`. Two equivalent alternative URLs are provided for accessing the associated services.
* For `SecondDRM`, the MPD does not provide the license server URL. It must be supplied at runtime.

There are two encrypted representations in the adaptation set, each with a different codecs string. Both codecs strings are included in the [=required capability set=] of this adaptation set. A [=DRM system=] must support playback of both representations in order to present this adaptation set.

</div>

In addition to the MPD, a DASH client can use [=solution-specific logic and configuration=] for controlling DRM selection and configuration decisions (e.g. loading license server URLs from configuration data instead of the MPD). This is often implemented in the form of callbacks exposed by the DASH client to an "app" layer in which the client is hosted. It is assumed that when executing any such callbacks, a DASH client makes available relevant contextual data, allowing the business logic to make fully informed decisions.

The purpose of the [=DRM system=] selection workflow is to select a single [=DRM system=] that is capable of decrypting a meaningful subset of the adaptation sets selected for playback. The selected [=DRM system=] will meet the following criteria:

1. It is actually implemented by the [=media platform=].
1. It supprots a set of capabilities sufficient to present an acceptable set of adaptation sets.
1. The necessary [=DRM system configuration=] for this [=DRM system=] is available.

It may be that the selected [=DRM system=] is only able to decrypt a subset of the encrypted adaptation sets selected for playback. See also [[#CPS-unavailable-keys]].

The set of adaptation sets considered during selection does not need to be constrained to a single period, potentially enabling seamless transitions to a new period with a different set of [=content keys=].

In live services new periods may be added over time, with potentially different [=DRM system configuration=] and [=required capability sets=], making it necessary to re-execute the selection process.

Note: If a new period has significantly different requirements in terms of [=DRM system configuration=] or the [=required capability sets=], the media pipeline may need to be re-initialized to play the new period. This may result in a glitch/pause at the period boundary. The specifics are implementation-dependant.

The default [=DRM system configuration=] in the MPD of a live service can change over time. DASH clients are not expected to re-execute DRM workflows if the default [=DRM system configuration=] in the MPD changes for an adaptation set that has already been processed in the past. Such changes will only affect clients that are starting playback.

When encrypted adaptation sets are initially selected for playback or when the selected set of encrypted adaptation sets changes (e.g. because a new period was added to a live service), a DASH client SHOULD execute the following algorithm for [=DRM system=] selection:

<div algorithm="drm-selection">

1. Let <var>adaptation_sets</var> be the set of encrypted adaptation sets selected for playback.
1. Let <var>signaled_system_ids</var> be the set of DRM system IDs for which a `ContentProtection` descriptor is present in the MPD on any entries in <var>adaptation_sets</var>.
1. Let <var>candidate_system_ids</var> be an ordered list initialized with items of <var>signaled_system_ids</var> in any order.
1. Provide <var>candidate_system_ids</var> to [=solution-specific logic and configuration=] for inspection/modification.
    * This enables business logic to establish an order of preference where multiple [=DRM systems=] are present.
    * This enables business logic to filter out DRM systems known to be unsuitable.
    * This enables business logic to include DRM systems not signaled in the MPD.

1. Let <var>default_kids</var> be the set of all distinct `default_KID` values in <var>adaptation_sets</var>.
1. Let <var>system_configurations</var> be an empty map of `system ID -> map(default_kid -> configuration)`, representing the [=DRM system configuration=] of each `default_KID` for each [=DRM system=].<br><img src="Images/SelectionAlgorithm-SystemConfigurations.png" />
1. For each <var>system_id</var> in <var>candidate_system_ids</var>:
    1. Let <var>configurations</var> be a map of `default_kid -> configuration` where the keys are <var>default_kids</var> and the values are the [=DRM system configurations=] initialized with data from `ContentProtection` descriptors in the MPD (matching on `default_KID` and <var>system_id</var>).
        * If there is no matching `ContentProtection` descriptors in the MPD, the map still contains a partially initialized [=DRM system configuration=] for the `default_KID`.
        * Enhance the MPD-provided default [=DRM system configuration=] with synthesized data where appropriate (e.g. [[#CPS-AdditionalConstraints-W3C|to generate W3C Clear Key initialization data in a format supported by the platform API]]).
    1. Provide <var>configurations</var> to [=solution-specific logic and configuration=] for inspection and modification, passing <var>system_id</var> along as contextual information.
        * This enables business logic to override the default [=DRM system configuration=] provided by the MPD.
        * This enables business logic to inject values that were not embedded in the MPD.
        * This enables business logic to reject [=content keys=] that it knows cannot be used, by removing the [=DRM system configuration=] for them.
    1. Remove any entries from <var>configurations</var> that do not contain all of the following pieces of data:
        * License server URL
        * [=DRM system=] initialization data in a format accepted by the particular [=DRM system=]; this is generally a `pssh` box [[!CENC]], though some [=DRM systems=] also support other formats
    1. Add <var>configurations</var> to <var>system_configurations</var> (keyed on <var>system_id</var>).
1. Remove from <var>candidate_system_ids</var> any entries for which the map of [=DRM system configurations=] in <var>system_configurations</var> is empty.

1. Let <var>required_capability_sets</var> be a map of `adaptation set -> capability set`, providing the [=required capability set=] of every item in <var>adaptation_sets</var>.
1. Match the capabilities of [=DRM systems=] with the [=required capability sets=] of adaptation sets:
    1. Let <var>supported_adaptation_sets</var> be an empty map of `system ID -> list of adaptation set`, incidating which adaptation sets are supported by which [=DRM systems=].
    1. For each <var>system_id</var> in <var>candidate_system_ids</var>:
        1. Let <var>candidate_adaptation_sets</var> by the set of adaptation sets for which <var>system_configurations</var> contains [=DRM system configuration=] (keyed on <var>system_id</var> and then the `default_KID` of the adaptation set).
            * This excludes from further consideration any adaptation sets that could not be used due to lacking [=DRM system configuration=], even if capabilities match.
        1. Let <var>maximum_capability_set</var> be the union of all values in <var>required_capability_sets</var> keyed on items of <var>candidate_adaptation_sets</var>.
        1. Query the [=DRM system=] identified by <var>system_id</var> with the capability set <var>maximum_capability_set</var>, assigning the output to <var>supported_capability_set</var>.
            * A [=DRM system=] that is not implemented is treated as having no capabilities.
        1. For each <var>adaptation_set</var> in <var>candidate_adaptation_sets</var>:
            1. If <var>supported_capability_set</var> contains all the capabilities in the corresponding entry in <var>required_capability_sets</var> (keyed on <var>adaptation_set</var>), add <var>adaptation_set</var> to the list in <var>supported_adaptation_sets</var> (keyed on <var>system_id</var>).

1. Remove from <var>supported_adaptation_sets</var> any entries for which the value (the set of adaptation sets) meets any of the following criteria:
    * The set is empty (the [=DRM system=] does not support playback of any adaptation set).
    * The set does not contain all encrypted media types present in the MPD (e.g. the [=DRM system=] can decrypt only the audio content but not the video content).
1. If <var>supported_adaptation_sets</var> is empty, playback of encrypted content is not possible and the workflow ends.
1. If <var>supported_adaptation_sets</var> contains multiple items, request [=solution-specific logic and configuration=] to select the preferred [=DRM system=] from among them.
    * This allows [=solution-specific logic and configuration=] to make an informed choice when different [=DRM systems=] can play different adaptation sets. Contrast this to the initial order of preference that was defined near the start of the algorithm, which does not consider capabilities.
1. If [=solution-specific logic and configuration=] does not make a decision, find the first entry in <var>candidate_system_ids</var> that is among the keys of <var>supported_adaptation_sets</var>. Remove items with any other key from <var>supported_adaptation_sets</var>.
    * This falls back to the "order of preference" logic and takes care of scenarios where business logic did not make an explicit choice.
1. Let <var>selected_system_id</var> be the single remaining key in <var>supported_adaptation_sets</var>.
1. Let <var>final_adaptation_sets</var> be the single remaining value in <var>supported_adaptation_sets</var>.
1. Let <var>final_configurations</var> (map of `default_KID -> DRM system configuration`) be the value from <var>system_configurations</var> keyed on <var>selected_system_id</var>.
1. Remove from <var>final_configurations</var> any entries keyed on `default_KID` values that are not used by any adaptation set in <var>final_adaptation_sets</var>.
    * These are the configurations of adaptation sets for which configuration was present but for which the required capabilities were not offered by the [=DRM system=].
1. Prohibit playback of any encrypted adaptation sets that are not in <var>final_adaptation_sets</var>.
    * These are existing adaptation sets for which either no [=DRM system configuration=] exists or for which the required capabilities are not provided by the selected [=DRM system=].
1. Execute the [[#CPS-activation-workflow|DRM system activation workflow]], providing <var>selected_system_id</var> and <var>final_configurations</var> as inputs.

</div>

If a [=DRM system=] is successfully selected, activation and potentially one or more license requests will follow before playback can proceed. These related workflows are described in the next chapters.

## Activating the DRM system ## {#CPS-activation-workflow}

Once a suitable [=DRM system=] has been selected, it must be activated by providing it a list of [=content keys=] that the DASH client requests to be made available for content decryption, together [=DRM system=] specific initialization data for each of the [=content keys=]. The result of activation is a [=DRM system=] that is ready to decrypt zero or more encrypted adaptation sets selected for playback.

During activation, it may be necessary [[#CPS-license-request-workflow|to perform license requests]] in order to obtain some or all of the [=content keys=] and the usage policy that constrains their use. Some of the requested [=content keys=] may already be available to the [=DRM system=], in which case no license request will be triggered.

Note: The details of stored [=content key=] management and persistent DRM session management are out of scope of this document - workflows described here simply accept the fact that some [=content keys=] may already be available, regardless of why that is the case or what operations are required to establish [=content key=] persistence.

Once a suitable [=DRM system=] [[#CPS-selection-workflow|has been selected]], a DASH client SHOULD execute the following algorithm to activate it:

<div algorithm="drm-activation">

1. Let <var>configurations</var> be the input to the algorithm; it is a map with the entry keys being `default_KID` values identifying the [=content keys=] and the entry values being the [=DRM system configuration=] to use with that particular [=content key=].
1. Let <var>pending_license_requests</var> be an empty set.
1. For each <var>kid</var> and <var>config</var> pair in <var>configurations</var> invoke the platform API to activate the selected [=DRM system=] and signal it to make <var>kid</var> available for decryption, passing the [=DRM system=] the initialization data stored in <var>config</var>.
    * If the [=DRM system=] indicates that one or more license requests are needed, add any license request data provided by the [=DRM system=] and/or platform API to <var>pending_license_requests</var>, together with the associated <var>kid</var> and <var>config</var> values.
1. If <var>pending_license_requests</var> is not an empty set, execute the [[#CPS-license-request-workflow|license request workflow]] and provide this set as input to the algorithm.
1. Inspect the set of [=content keys=] the [=DRM system=] indicates are now available and deselect from playback any adaptation sets for which the [=content key=] has not become available.
1. Inspect the set of remaining adaptation sets to determine if a sufficient data set remains for successful playback. Raise error if playback cannot continue.

</div>

The default format for initialization data supplied to a [=DRM system=] is a `pssh` box. However, if the DASH client has knowledge of any special initialization requirements of a particular [=DRM system=], it MAY supply initialization data in other formats (e.g. the `keyids` JSON structure used by W3C Clear Key). Presence of initialization data in the expected format is considered during [[#CPS-selection-workflow|DRM system selection]] when determining whether a [=DRM system=] is a valid candidate.

For historical reasons, platform APIs often implement [=DRM system=] activation as a per-content-key operation. Some APIs and [=DRM system=] implementations may also support batching all the [=content keys=] into a single activation operation, for example by combining multiple "[=content key=] and DRM system configuration" data sets into a single data set in a single API call. DASH clients MAY make use of such batching where supported by the platform API. The workflow in this chapter describes the most basic scenario where activation must be performed separately for each [=content key=].

Note: The batching may, for example, be accomplished by concatenating all the `pssh` boxes for the different [=content keys=]. Support for this type of batching among DRM systems and platform APIs remains uncommon, despite the potential efficiency gains from reducing the number of license requests triggered.

### Handling unavailability of [=content keys=] ### {#CPS-unavailable-keys}

It is possible that not all of the encrypted adaptation sets selected for playback can actually be played back (e.g. because a [=content key=] for ultra-HD content is only authorized for use by implementations with a high [=robustness level=]). The unavailability of one or more [=content keys=] SHOULD NOT be considered a fatal error condition as long as at least one audio and at least one video adaptation set remains available for playback (assuming both content types are initially selected for playback). This logic MAY be overridden by solution specific business logic to better reflect end-user expectations.

The set of available [=content keys=] can change over time (e.g. due to [=license=] expiration or due to new periods in the presentation requiring different content keys). A DASH client SHALL monitor the set of `default_KID` values that are required for playback and either request the [=DRM system=] to make these [=content keys=] available or deselect the affected adaptation sets when the [=content keys=] become unavailable. Conceptually, any such change can be handled by re-executing the [[#CPS-selection-workflow|DRM system selection]] and [[#CPS-activation-workflow|activation workflows]], although platform APIs may also offer more fine-grained update capabilities.

A DASH client can request a [=DRM system=] to enable decryption using any set of [=content keys=] (if it has the necessary [=DRM system configuration=]). However, this is only a request and playback can be countermanded at multiple stages of processing by different involved entities.

<figure>
	<img src="Diagrams/ReductionOfKeys.png" />
	<figcaption>The set of [=content keys=] made available for use can be far smaller than the set requested by a DASH client. Example workflow indicating potential instances of [=content keys=] being removed from scope.</figcaption>
</figure>

The set of available [=content keys=] is only known at the end of executing the activation workflow and may decrease over time (e.g. due to [=license=] expiration). The proper handling of unavailable keys depends on the limitations imposed by the platform APIs.

Advisement: [=Media platform=] APIs often refuse to start or continue playback if the [=DRM system=] is not able to decrypt all the data already in [=media platform=] buffers.

It may be appropriate for a DASH client to avoid buffering data for encrypted adaptation sets until the required [=content key=] is known to be available. This allows the client to avoid potentially expensive buffer resets and rebuffering if unusable data needs to be removed from buffers.

Note: The DASH client should still download the data into intermediate buffers for faster startup and simply defer submitting it to the [=media platform=] API until key availability is confirmed.

If a [=content key=] expires during playback, it is common for a [=media platform=] to pause playback until the [=content key=] can be refreshed with a new [=license=] or until data encrypted with the now-unusable [=content key=] is removed from buffers. DASH clients SHOULD acquire new [=licenses=] in advance of [=license=] expiration. Alternatively, DASH clients should implement appropriate recovery/fallback behavior to ensure a minimally disrupted user experience in situations where some [=content keys=] remain available.

## Content protection policies ## {#CPS-protection-policies}

When [=content keys=] are acquired, the [=license=] that delivers them also supplies a policy for the [=DRM system=], instructing it how to protect the content that is made accessible by the [=content keys=].

<div class="example">

Protection policy may define the following example requirements:

* All connected displays must support HDCP 2.2 or newer.
* The video display area must be no more than 1280x720 pixels.
* Minimum [=DRM system=] [=robustness level=] is "800".

</div>

<b>Typical [=DRM systems=] will enforce the most restrictive protection policy from among all active [=content keys=] and will refuse to start playback if <u>any</u> of the constraints cannot be satisfied!</b> As a result, it can be the case that even though only the constraints for a UHD video stream cannot be satisfied, playback of even the lower quality levels is blocked.

In many cases, it might be more desirable to instead exclude the UHD quality level from the set of adaptation sets selected for playback and [=DRM system=] activation. Alternatively, there may be a different [=DRM system=] implementation available on the device that is capable of satisfying the constraints. It is not possible for a DASH client to resolve these constraints as it has no knowledge of what policy applies nor of the capabilities of the different [=DRM system=] implementations.

[=Solution-specific logic and configuration=] SHOULD be used to select the most suitable [=DRM system=], taking into consideration the protection policy, and to preemptively exclude adaptation sets from playback if it can be foreseen that the protection policy for their [=content keys=] cannot be satisfied. Likewise, license servers SHOULD NOT provide [=content keys=] if it can be foreseen that the recipient will be unable to satisfy their protection policy.

## Performing license requests ## {#CPS-license-request-workflow}

DASH clients performing license requests SHOULD follow the [[#CPS-lr-model|DASH-IF interoperable license request model]]. The remainder of this chapter only applies to DASH clients that follow this model. Alternative implementations are possible and in common use but are not interoperable and are not described in this document.

[=DRM systems=] generally do not perform license requests on their own. Rather, when they determine that a [=license=] is required, they generate a document that serves as the license request body and expect the DASH client to deliver it to a license server for processing. The latter returns a suitable response that, if a [=license=] is granted, encapsulates the [=content keys=] in an encrypted form only readable to the DRM system.

<figure>
	<img src="Diagrams/LicenseRequestConcept.png" />
	<figcaption>Simplified conceptual model of license request processing. Many details omitted.</figcaption>
</figure>

The request and response body are in [=DRM system=] specific formats and considered opaque to the DASH client. A DASH client SHALL NOT modify the request body or the response body.

The license request workflow defined here exists to enable the following goals to be achieved without the need to customize the DASH client with logic specific to a [=DRM system=] or license server implementation:

1. Provide proof of authorization if the license server requires the DASH client to prove that the user being served has the rights to use the requested [=content keys=].
1. Execute the license request workflow driven purely by the MPD, without any need for [=solution-specific logic and configuration=].
1. Detect common error scenarios and present an understandable message to the user.

The proof of authorization is optional and the need to attach it to a license request is indicated by the presence of at least one `dashif:authzurl` in the [=DRM system configuration=]. The proof of authorization is a [[!jwt|JSON Web Token]] in compact encoding (the `aaa.bbb.ccc` form) returned as the HTTP response body when the DASH client performs a GET request to this URL. The token is attached to a license request in the HTTP `Authorization` header with the `Bearer` type. For details, see [[#CPS-lr-model]].

Error responses from both the authorization service and the license server SHOULD be returned as [[rfc7807]] compatible responses with a 4xx or 5xx status code and `Content-Type: application/problem+json`.

DASH clients SHOULD implement retry behavior to recover from transient failures and expiration of [=authorization tokens=].

To process license requests queued during execution of the [[#CPS-activation-workflow|DRM system activation workflow]], the client SHOULD execute the following algorithm:

<div algorithm="drm-license-acquisition">

1. Let <var>pending_license_requests</var> be the set of license requests that the [=DRM system=] has requested to be performed, with at least the following data present in each entry:
    * The license request body provided by the [=DRM system=].
    * The [=DRM system configuration=].
1. Let <var>retry_requests</var> be an empty set. It will contain the set of license requests that are to be retried due to transient failure.
1. Let <var>pending_authz_requests</var> be a map of `URL -> GUID[]`, with the keys being authorization service URLs and the values being lists of `default_KIDs`. The map is initially empty.
1. For each <var>request</var> in <var>pending_license_requests</var>:
    1. If the [=DRM system configuration=] does not contain at least one value for `dashif:authzurl`, skip to the next loop iteration. This means that no [=authorization token=] is to be attached to this license request.
    1. Create/update the entry in <var>pending_authz_requests</var> with the key being the set of `dashif:authzurl` values; add the `default_KID` to the list in the map entry value.
1. Let <var>authz_tokens</var> be a map of `GUID -> string`, with the keys being `default_KIDs` and the values being the associated [=authorization tokens=]. The map is initially empty.
1. For each <var>authz_url_set</var> and <var>kids</var> pair in <var>pending_authz_requests</var>:
    1. If the DASH client has a cached [=authorization token=] previously acquired for the same <var>authz_url_set</var> and <var>kids</var> combination that still remains valid according to its `exp` "Expiration Time" claim:
        1. Let <var>authz_token</var> be the cached [=authorization token=].
    1. Else:
        1. Create a comma-separated list from <var>kids</var> in ascending alphanumeric (ASCII) order.
        1. Let <var>authz_url</var> be a random item from <var>authz_url_set</var>.
        1. Let <var>authz_url_with_kids</var> be <var>authz_url</var> with an additional query string parameter named `kids` with the value from <var>kids</var>.
            * <var>authz_url</var> may already include query string parameters, which should be preserved!
        1. Perform an HTTP GET request to <var>authz_url_with_kids</var> (following redirects).
            * Include any relevant HTTP cookies.
            * Allow [=solution-specific logic and configuration=] to intercept the request and inspect/modify it as needed (e.g. provide additional HTTP request headers to enable user identification).
        1. If the response status code [[#CPS-lr-model-errors|indicates failure]], make a note of any error information for later processing and skip to the next <var>authz_url</var>.
        1. Let <var>authz_token</var> be the HTTP response body.
        1. Submit <var>authz_token</var> into the DASH client cache, with the cache key being a combination of <var>authz_url_set</var> and <var>kids</var>, and the cache entry expiration being defined by the `exp` "Expiration Time" claim in the [=authorization token=] (defaulting to never expires).
    1. For each <var>kid</var> in <var>kids</var>, add an entry to <var>authz_tokens</var> with the key <var>kid</var> and the value being <var>authz_token</var>.
1. For each <var>request</var> in <var>pending_license_requests</var>:
    1. If the [=DRM system configuration=] from <var>request</var> contains an authorization service URL but there is no entry in <var>authz_tokens</var> keyed on the `default_KID` from <var>request</var>, skip to the next loop iteration.
        * This occurs when an [=authorization token=] is required but cannot be obtained for this license request.
    1. Execute an HTTP POST request with the following parameters:
        * Request body is the license request body from <var>request</var>.
        * Request URL is defined by [=DRM system configuration=]. If multiple license server URLs are defined, select a random URL from the set.
        * If <var>authz_tokens</var> contains an entry with the key being the `default_KID` from <var>request</var>, add the `Authorization` header with the value being the string `Bearer` concatenated with a space and the [=authorization token=] from <var>authz_tokens</var> (e.g. `Bearer aaa.bbb.ccc`).
    1. If the response status code [[#CPS-lr-model-errors|indicates failure]]:
        1. Expel the used [=authorization token=] (if any) from the DASH client cache to force a new token to be used for any future license requests.
        1. If the DASH client believes that retrying the license request might succeed (e.g. because the response indicates that the error might be transient or due to an expired [=authorization token=] that can be renewed), add <var>request</var> to <var>retry_requests</var>.
        1. Make a note of any error information for later processing and presentation to the user.
        1. Skip to the next loop iteration.
    1. Submit the HTTP response body to the [=DRM system=] for processing.
        * This may cause the [=DRM system=] to trigger additional license requests. Append any triggered request to <var>pending_license_requests</var> and copy the [=DRM system configuration=] from the current entry, processing the additional entry in a future iteration of the same loop.
        * If the [=DRM system=] indicates a failure to process the data, make a note of any error information for later processing and skip to the next loop iteration.
1. If <var>retry_requests</var> is not empty, re-execute this workflow with <var>retry_requests</var> as the input.

</div>

While the above algorithm is presented sequentially, authorization requests and license requests may be performed in a parallelized manner to minimize processing time.

At the end of this algorithm, all pending license requests have been performed. However, it is not necessary that all license requests or authorization requests succeed! For example, even if one of the requests needed to obtain an HD quality level [=content key=] fails, other requests may still make SD quality level [=content keys=] available, leading to a successful playback if the HD quality level is deselected by the DASH client. Individual failing requests therefore do not indicate a fatal error. Rather, such error information should be collected and provided to the top-level error handler of the DRM system activation workflow, which can make use of this data to present user-friendly messages if it decides that meaningful playback cannot take place with the final set of available [=content keys=]. See also [[#CPS-unavailable-keys]].

### Efficient license acquisition ### {#CPS-efficiency-in-license-requests}

In some situations a DASH client can foresee the need to make new [=content keys=] available for use or to renew the [=licenses=] that enable [=content keys=] to be used. For example:

* Live DASH services can at any time introduce new periods that use different [=content keys=]. They can also alternmate between encrypted and clear content in different periods.
* The [=license=] that enables a [=content key=] to be used can have an expiration time, after which a new [=license=] is required.

DASH clients SHOULD perform license acquisition ahead of time, activating a [=DRM system=] before it is needed or renewing [=licenses=] before they expire. This provides the following benefits:

* Playback can continue seamlessly when [=licenses=] are renewed, without pausing for license acquisition.
* New [=content keys=] are already available when content needs them, again avoiding a pause for license acquisition.

To avoid a huge number of concurrent license requests causing license server overload, a DASH client SHOULD perform a license request at a randomly selected time between the moment when it became aware of the need for the license request and the time when the [=license=] must be provided to a [=DRM system=] (minus some safety margin).

Multiple license requests to the same license server with the same [=authorization token=] SHOULD be batched into a single request if the [=media platform=] API supports this. See [[#CPS-activation-workflow]] for details.

The possibility for ahead-of-time [=DRM system=] activation, seamless [=license=] renewal and license request batching depends on the specific [=DRM system=] and [=media platform=] implementations. Some implementations might not support optimal behavior.