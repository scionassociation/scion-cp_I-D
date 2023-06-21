---

title: "SCION Control Plane"
abbrev: "SCION CP"
category: info
submissiontype: independent

docname: draft-dekater-scion-controlplane-latest
v: 3
# area: "IRTF"
# workgroup: "Path Aware Networking RG"
keyword: Internet-Draft
venue:
#  group: "Path Aware Networking RG"
#  type: "Research Group"
#  mail: "panrg@irtf.org"
#  arch: "https://www.ietf.org/mail-archive/web/panrg/"
  github: "scionassociation/scion-cp_I-D"
  latest: "https://scionassociation.github.io/scion-cp_I-D/draft-dekater-scion-controlplane.html"

author:
 -   ins: C. de Kater
     name: Corine de Kater
     org: SCION Association
     email: cdk@scion.org

 -   ins: M. Frei
     name: Matthias Frei
     org: SCION Association
     email: matzf@scion.org

 -   ins: N. Rustignoli
     name: Nicola Rustignoli
     org: SCION Association
     email: nic@scion.org

normative:
  RFC1122:
  RFC2119:
  RFC4632:
  RFC5398:
  RFC5952:
  RFC6996:
  RFC8174:
  I-D.path-properties-voc:
    title: A Vocabulary of Path Properties
    date: 2023
    target: https://datatracker.ietf.org/doc/draft-irtf-panrg-path-properties/
    author:
      -
        ins: R. Enghardt
        name: Reese Enghardt
        org: Netflix
      -
        ins: C. Krähenbühl
        name: Cyrill Krähenbühl
        org: ETH Zuerich

informative:
  CHUAT22:
    title: "The Complete Guide to SCION"
    date: 2022
    target: https://doi.org/10.1007/978-3-031-05288-0
    seriesinfo:
      ISBN: 978-3-031-05287-3
    author:
      -
        ins: L. Chuat
        name: Laurent Chuat
        org: ETH Zuerich
      -
        ins: M. Legner
        name: Markus Legner
        org: ETH Zuerich
      -
        ins: D. Basin
        name: David Basin
        org: ETH Zuerich
      -
        ins: D. Hausheer
        name: David Hausheer
        org: Otto von Guericke University Magdeburg
      -
        ins: S. Hitz
        name: Samuel Hitz
        org: Anapaya Systems
      -
        ins: P. Mueller
        name: Peter Mueller
        org: ETH Zuerich
      -
        ins: A. Perrig
        name: Adrian Perrig
        org: ETH Zuerich
  I-D.scion-components:
    title: SCION Components Analysis
    date: 2023
    target: https://datatracker.ietf.org/doc/draft-rustignoli-panrg-scion-components/
    author:
      -
        ins: N. Rustignoli
        name: Nicola Rustignoli
        org: SCION Association
      -
        ins: C. de Kater
        name: Corine de Kater
        org: SCION Association
  I-D.scion-cppki:
    title: SCION Control-Plane PKI
    date: 2023
    target: https://datatracker.ietf.org/doc/draft-dekater-scion-pki/
    author:
      -
        ins: C. de Kater
        name: Corine de Kater
        org: SCION Association
      -
        ins: N. Rustignoli
        name: Nicola Rustignoli
        org: SCION Association

  I-D.scion-overview:
    title: SCION Overview
    date: 2023
    target: https://datatracker.ietf.org/doc/draft-dekater-panrg-scion-overview/
    author:
      -
        ins: C. de Kater
        name: Corine de Kater
        org: SCION Association
      -
        ins: N. Rustignoli
        name: Nicola Rustignoli
        org: SCION Association
      -
        ins: A. Perrig
        name: Adrian Perrig
        org: ETH Zuerich




--- abstract

This document describes the control plane of the path-aware, inter-domain network architecture SCION (Scalability, Control, and Isolation On Next-generation networks). One of the basic characteristics of SCION is that it gives path control to SCION-capable endpoints. In fact, endpoints can choose between multiple path options, enabling the optimization of network paths. The SCION control plane is responsible for discovering these paths and making them available to the endpoints.

The main goal of SCION's control plane is to create and manage path segments, which can then be combined into forwarding paths to transmit packets in the data plane. This document first discusses how path exploration is realized through beaconing and how path segments are created and registered. Each SCION autonomous system (AS) can register segments according to its own policy - it is free to specify which path properties and algorithm to use in the selection procedure. The document then describes the path lookup process, where endpoints obtain path segments - a fundamental building block for the construction of end-to-end paths.


--- middle

# Introduction

The SCION control plane is responsible for discovering path segments and making them available to endpoints. This process includes path exploration, registration, and lookup. This section explains the basic concepts of the control plane in SCION and introduces SCION's routing concept.

As SCION is an *inter-domain* network architecture, it only deals with *inter*-domain routing. One feature of SCION is the decoupling of inter-domain routing from endpoint addressing. This Introduction section provides a description of the SCION addressing system in more detail.

**Note:** It is assumed that readers of this draft are familiar with the basic concept of the SCION next-generation inter-domain network architecture. If not, please find more detailed information in the IETF Internet Drafts {{I-D.scion-overview}}, {{I-D.scion-components}}, and {{I-D.scion-cppki}}, as well as in {{CHUAT22}}, especially Chapter 2. A short description of the SCION basic terms and elements can be found in [](#terms) below.


## Terminology {#terms}

**Autonomous System (AS)**: An autonomous system is a network under a common administrative control. For example, the network of an Internet service provider, company, or university can constitute an AS.

**Beaconing**: The control-plane process where an AS discovers paths to other ASes.

**Control Plane**: The SCION control plane is responsible for the propagation and discovery of network paths, i.e., for the exchange of routing information between network nodes. The control plane thus determines where traffic can be sent and deals with questions such as how paths are discovered, which paths exist, what quality individual links offer, etc. Within a SCION AS, such functionalities are carried out by the control service. Packet forwarding is instead a task pertaining to the data plane.

**Control Service**: The control service is the main control-plane infrastructure component within a SCION AS. It is responsible for the path exploration and registration processes that take place within the control plane.

**Core AS**: Each isolation domain (ISD) is administered by a set of distinguished autonomous systems (ASes) called core ASes, which are responsible for initiating the path-discovery and -construction process (in SCION called "beaconing").

**Endpoint**: An endpoint is the start- or the endpoint of a SCION path. For example, an endpoint can be a host as defined in {{RFC1122}}, or a gateway bridging a SCION and an IP domain. This definition is based on the definition in {{I-D.path-properties-voc}}.

**Forwarding Path**: A forwarding path is a complete end-to-end path between two SCION hosts, which is used to transmit packets in the data plane and can be created with a combination of up to three path segments (an up-segment, a core-segment, and a down-segment).

**Hop Field (HF)**: As they traverse the network, path-segment construction beacons (PCBs) accumulate cryptographically protected AS-level path information in the form of hop fields. In the data plane, hop fields are used for packet forwarding: they contain the incoming and outgoing interface IDs of the ASes on the forwarding path.

**Info Field (INF)**: Each path-segment construction beacon (PCB) contains a single info field, which provides basic information about the PCB. Together with hop fields (HFs), info fields are used to create forwarding paths.

**Isolation Domain (ISD)**: In SCION, autonomous systems (ASes) are organized into logical groups called isolation domains or ISDs. Each ISD consists of ASes that span an area with a uniform trust environment (i.e., a common jurisdiction). A possible model is for ISDs to be formed along national boundaries or federations of nations.

**Leaf AS**: An AS at the "edge" of an ISD, with no other downstream ASes.

**Packet-Carried Forwarding State (PCFS)**: Rather than relying on costly inter-domain forwarding tables, SCION data packets contain all the necessary path information. We refer to this property as packet-carried forwarding state or PCFS.

**Path Segment**: Path segments are derived from path-segment construction beacons (PCBs) and registered at control services. A path segment can be (1) an up-segment (i.e., a path between a non-core AS and a core AS in the same ISD), (2) a down-segment (i.e., the same as an up-segment, but in the opposite direction), or (3) a core-segment (i.e., a path between core ASes). Up to three path segments can be used to create a forwarding path.

**Path-Segment Construction Beacon (PCB)**: Core ASes generate PCBs to explore paths within their isolation domain (ISD) and among different ISDs. ASes further propagate selected PCBs to their neighboring ASes. As a PCB traverses the network, it carries path segments, which can subsequently be used for traffic forwarding.


## Paths and Links

SCION routers and endpoints connect to each other via links. A SCION path between two endpoints essentially traverses one or more links.

In SCION, autonomous systems (ASes) are organized into logical groups called isolation domains or ISDs. Each ISD consists of ASes that span an area with a uniform trust environment (i.e., a common jurisdiction). An ISD is administered by a set of distinguished ASes called core ASes. Within and between ISDs, SCION supports three types of links: (1) core links, (2) parent-child links, and (3) peering links.

- A *core* link always connects two core ASes, which are either within the same or in a different ISD. Core links can exist for various underlying business relationships, including provider-customer (where the customer pays the provider for traffic) and peering relationships.
- *Parent-child* links create a hierarchy between the parent and the child. ASes with a parent-child link typically have a provider-customer relationship.
- *Peering* links exist between ASes with a (settlement-free or paid) peering relationship. Peering links can only be used to reach destinations within or downstream of the peering AS. They can be established between any two core or non-core ASes, and between core and non-core ASes. Peering links can also cross ISD boundaries.

Each link connecting SCION routers is bidirectional and identified by its corresponding egress and ingress interface IDs. These IDs only need to be unique within each AS. Therefore, they can be chosen and encoded by each AS independently and without any need for coordination.


## Routing

SCION provides path-aware inter-domain routing between ASes across the Internet. The SCION control plane is responsible for discovering these inter-domain paths and making them available to the endpoints within the ASes. SCION inter-domain routing operates on two levels: Within a SCION isolation domain (ISD), which is called *intra*-ISD routing, and between ISDs, called *inter*-ISD routing. Both levels use the so-called *path-segment construction beacons (PCBs)* to explore network paths. A PCB is initiated by a core AS and then disseminated either within an ISD to explore intra-ISD paths, or among core ASes, to explore core paths across different ISDs.

The PCBs accumulate cryptographically protected path and forwarding information on AS-level, and store this information in the form of *hop fields*. Endpoints use information from these hop fields to create end-to-end forwarding paths for data packets, who carry this information in their packet headers. This concept is called *packet-carried forwarding state*. The concept also supports multi-path communication among endpoints.

The creation of an end-to-end forwarding path consists of the following processes:

1. *Path exploration (or beaconing)*: This is the process where an AS discovers paths to other ASes. See also [](#beaconing).
2. *Path registration*: This is the process where an AS selects a few PCBs, according to defined policies, turns the selected PCBs into path segments, and adds these path segments to the relevant path infrastructure, thus making them available to other ASes. See also [](#path-segment-reg).
3. *Path resolution*: This is the process of actually creating an end-to-end forwarding path from the source endpoint to the destination. For this, an endpoint performs (a) a path lookup step, to obtain path segments, and (b) a path combination step, to combine the forwarding path from the segments. This last step takes place in the data plane. See also [](#lookup).

All processes operate concurrently.

{{figure-1}} below shows the SCION routing processes and their relation to each other.

~~~~
+-------------------------+       +-------------------------+
| Exploration (Beaconing) |------>|      Registration       |
+-------------------------+       +-----------+-------------+
                                              |
                              +---------------+
                              |
     +------------------------v------------------------+
     |                 Path Resolution                 |
     |                                                 |
     |   +----------------+       +----------------+   |
     |   |     Lookup     +------>|  Combination   |   |
     |   |                |       |    (Data Plane)|   |
     |   +----------------+       +----------------+   |
     +-------------------------------------------------+
~~~~
{: #figure-1 title="SCION routing processes and their relation to each other. All processes operate concurrently"}


The **control service** is responsible for the path exploration and registration processes in the control plane. It is the main control-plane infrastructure component within each SCION AS. The control service of an AS has the following tasks:

- Generating, receiving, and propagating PCBs. Periodically, the control service of a core AS generates a set of PCBs, which are forwarded to the child ASes or neighboring core ASes. In the latter case, the PCBs are sent over policy-compliant paths to discover multiple paths between any pair of core ASes.
- Selecting and registering the set of path segments via which the AS wants to be reached.
- Managing certificates and keys to secure inter-AS communication. Each PCB contains signatures of all on-path ASes. Every time the control service of an AS receives a PCB, it validates the PCB's authenticity. When the control service lacks an intermediate certificate, it can query the control service of the neighboring AS that sent the PCB.

**Note:** The control service of an AS must not be confused with a border router. The control service of a specific AS is part of the control plane and responsible for *finding and registering suitable paths*. It can be deployed anywhere inside the AS. A border router belongs to the data plane; its main task is to *forward data packets*. Border routers are deployed at the edge of an AS.


### Path Segments

As described previously, the main goal of SCION's control plane is to create and manage path segments, which can then be combined into forwarding paths to transmit packets in the data plane. SCION distinguishes the following types of path segments:

- A path segment from a non-core AS to a core AS is an *up-segment*.
- A path segment from a core AS to a non-core AS is a *down-segment*.
- A path segment between core ASes is a *core-segment*.

So each path segment either ends at a core AS, or starts at a core AS, or both.

**Note:** There are no SCION path segments that start and end at a non-core AS. However, when combining path segments into an end-to-end SCION path, it is possible to use peering links. For more information on SCION and peering links, see [](#beaconing).

All path segments are invertible: A core-segment can be used bidirectionally, and an up-segment can be converted into a down-segment, or vice versa, depending on the direction of the end-to-end path. This means that all path segments can be used to send data traffic in both directions.


## Addressing {#numbering}

The inter-domain SCION routing is based on the <ISD, AS> tuple. Although a complete SCION address is composed of the <ISD, AS, endpoint address> 3-tuple, the endpoint address is not used for inter-domain routing or forwarding. The endpoint address can be of variable length, does not need to be globally unique, and can thus be an IPv4, IPv6, or MAC address, for example - in fact, the endpoint address is the "normal", currently used, non-SCION-specific endpoint address.

However, the ISD-AS number is a SCION-specific number. It consists of 64-bits, with the top 16 bits indicating the ISD, and the bottom 48 bits indicating the AS. The text representation uses a `-`(dash) separator between the ISD and AS numbers, for example: `4-ff00:1:f`. This section provides more details about the numbering scheme for SCION ISD and AS numbers.

**Note:** As a consequence of the fact that SCION relies on existing routing protocols (e.g., IS-IS, OSPF, SR) and communication fabric (e.g., IP, MPLS) for intra-domain forwarding, existing internal routers do not need to be changed to support SCION.


### ISD numbers

An ISD number is the 16-bit global identifier for an ISD. It MUST be globally unique. The following table gives an overview of the ISD number allocation.

| ISD          | Description                                                                             |
|--------------+-----------------------------------------------------------------------------------------|
| 0            | The wildcard ISD.                                                                       |
| 1 - 15       | Reserved for documentation and sample code (analogous to {{RFC5398}}.                   |
| 16 - 63      | Private use (analogous to {{RFC6996}}. Can be used for testing and private deployments. |
| 64 - 4094    | Public ISDs. Should be allocated in ascending order, without gaps and "vanity" numbers. |
| 4095&nbsp;-&nbsp;65535 | Reserved for future use.                                                                |
{: #table-1 title="ISD number allocations"}

A suitable mechanism to globally coordinate the assignation of ISD numbers does not yet exist. However, we hope that in the future an organization such as ICANN or a regional Internet registry (e.g., RIPE NCC) will take on the responsibility of assigning ISD and AS numbers.

Currently, ISD numbers are allocated by Anapaya, the Swiss-based provider of SCION-based networking software and solutions.


### AS numbers

An AS number is the 48-bit identifier for an AS.  SCION inherits the existing 32-bit AS numbers from RFC4893, but provides an extended 48-bit space, allowing for additional SCION-only AS numbers beyond the 32-bit space in use today.


#### Formatting

The default formatting for AS numbers in SCION is very similar to IPv6 (see {{RFC5952}}). It uses a 16-bit `:`-separated lower-case hex encoding with leading 0's ommitted: `0:0:0` to `ffff:ffff:ffff`.

In SCION, the following rules apply:

- The `::` zero-compression feature of IPv6 is NOT allowed. The feature has very limited use in a 48-bit address space and would only add more complexity.
- In order to provide easy comparison with BGP AS numbers, any AS number in the BGP AS range (0 - 2<sup>32-1</sup>) SHOULD be represented as decimal. It is allowed to write a BGP AS number in the SCION AS syntax. However, programs SHOULD always use the decimal representation for display. For example, if a program receives the AS number `0:1:f`, it should display the number as "65551".
- A range of AS numbers can be shortened with a notation similar to the one used for CIDR IP ranges ({{RFC4632}}). For example, the range of the lowest 32-bit ASNs (0-4294967295) can be represented as `0:0:0/16`.

The next table gives an overview of the AS number allocation.

| AS               | Size        | Description                                                                 |
|------------------+-------------+-----------------------------------------------------------------------------|
| 0                | 1           | The wildcard AS                                                             |
| 1-4294967295     | ~4.3&nbsp;bill.  | 32-bit BGP AS numbers, formatted as decimal. If a BGP AS deploys SCION, it has the same AS number for both BGP and SCION.<sup>1</sup> |
| `1:0:0`          | 1           | Reserved                                                                    |
| `2:0:0/16`       | ~4.3&nbsp;bill.  | Public SCION-only ASes (i.e., ASes that are created for SCION, and are no existing BGP ASes). They should be allocated in ascending order, without gaps and "vanity" numbers. |
| `ff00:0:0/32`    | 65535       | Reserved for documentation and test/sample code (analogous to {{RFC5398}}). |
| `ff00:0:0/24`    | ~16.8&nbsp;mill. | Reserved for private use (analogous to {{RFC6996}}). These numbers can be used for testing/private deployments. |
| `ffff:ffff:ffff` | 1           | Reserved                                                                    |
{: #table-2 title="AS number allocations"}

1) Some 32-bit AS numbers are reserved for special purposes. For more details, see ["IANA: Special-Purpose Autonomous System (AS) Numbers"](https://www.iana.org/assignments/iana-as-numbers-special-registry/iana-as-numbers-special-registry.xhtml).

The rest of the space is currently unallocated.


### Wildcard Addressing {#serv-disc}

SCION allows endpoints to use wildcard addresses in the control-plane routing, to designate any core AS, e.g., to place requests for core- or down path segments during path lookup. These wildcard addresses are of the form I-0, to designate any AS in ISD I. Here, "0" is the wildcard for the AS. For more information, see [](#wildcard).


## Communication Protocol

All communication between the control services in different ASes is expressed in terms of gRPC remote procedure calls (for details, see [](https://grpc.io)). Service interfaces and messages are defined in the Protocol Buffer "proto3" interface definition language (for details, see [](https://protobuf.dev)).

**Note:** The details for how gRPC is mapped to the SCION data plane will be described in a separate document.


# Path Exploration (Beaconing) {#beaconing}

dsölögökdö


# Registration of Path Segments {#path-segment-reg}

kyxsnmcs


# Path Lookup {#lookup}

yxcxyc

## Lookup Process

## Behavior of Actors in the Lookup Process

### Use of Wildcard Addresses in the Lookup Process {#wildcard}


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

One of the fundamental objectives that guided the design of SCION is security, in particular network security. See chapter 7 of the SCION book (Security Analysis), which states the precise security goals of various network participants and how SCION achieves these goals in the presence of different types of adversaries {{CHUAT22}}.


# IANA Considerations

TODO IANA considerations.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
