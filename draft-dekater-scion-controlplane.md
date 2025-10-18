- Managing certificates and keys to secure inter‑AS communication. Each PCB contains signatures from all on‑path ASes, and each time a Control Service receives a PCB it validates the PCB's authenticity. If the Control Service lacks an intermediate certificate, it can request it from the Control Service of the neighboring AS that sent the PCB via the API described in [](#crypto-api).

In case of failure, RPC calls return an error as specified by the RPC framework: a non‑zero status code and an explanatory string.

 Possible types are defined by the `SignatureAlgorithm` definition. An unspecified signature algorithm is never valid. Other algorithms or curves MAY be used in the future. Signature algorithms are discussed further in {{I-D.dekater-scion-pki}}.

Control Services distribute cryptographic material for the PKI (see {{I-D.dekater-scion-pki}}) using the following protobuf messages via the `TrustMaterialService` RPCs:

A `ChainsRequest` message includes the following fields:

- `isd_as`: The ISD‑AS of the subject in the AS certificate.
- `subject_key_id`: The SubjectKeyID from the AS certificate.
- `at_least_valid_until`: The point in time (in seconds since the UNIX epoch) at which the AS certificate must still be valid.
- `at_least_valid_since`: The point in time (in seconds since the UNIX epoch) at which the AS certificate must be, or must have been, valid.

A `ChainsResponse` includes the following fields:

- `chains`: The certificate chains that match the request. A `Chain` contains:
  - `as_cert`: The AS certificate in the chain.
  - `ca_cert`: The CA certificate in the chain.

A `TRCRequest` includes the following fields:

- `isd`: The ISD number of the TRC.
- `base`: The base number of the TRC.
- `serial`: The serial number of the TRC.

The response contains the raw TRC in the `trc` field.

Control Services provide paths to endpoints through the `SegmentLookupService` RPC. This API is exposed on the SCION dataplane by the control services of core ASes and on the intra‑domain protocol network.

~~~~
service SegmentLookupService {
    rpc Segments(SegmentsRequest) returns (SegmentsResponse) {}
}
~~~~

The API uses the following protobuf messages. A `SegmentsRequest` includes:

- `segments`: A list of `PathSegment` messages matching the request.
- `segments` (map): A mapping from path segment type (the integer representation of the `SegmentType` enum defined in [](#reg-proto)) to lists of `PathSegment` messages.