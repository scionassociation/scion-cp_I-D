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


## Introduction and Overview

**Path exploration** is the process where an AS discovers paths to other ASes. In SCION, this process is referred to as *beaconing*. This section explains the SCION beaconing process in detail.

In SCION, the *control service* of each AS is responsible for the beaconing process. The control service generates, receives, and propagates so-called *path-segment construction beacons (PCBs)* on a regular basis, to iteratively construct path segments. PCBs contain topology and authentication information, and can also include additional metadata that helps with path management and selection. The beaconing process itself is divided into routing processes on two levels, where *inter-ISD* or core beaconing is based on the (selective) sending of PCBs without a defined direction, and *intra-ISD* beaconing on top-to-bottom propagation. This division of routing levels is a key architectural decision of SCION and important for achieving a better scalability.

- *Inter-ISD or core beaconing* is the process of constructing path segments between core ASes in the same or in different ISDs. During core beaconing, the control service of a core AS either initiates PCBs or propagates PCBs received from neighboring core ASes to other neighboring core ASes. Core beaconing is periodic; PCBs are sent over policy-compliant paths to discover multiple paths between any pair of core ASes.
- *Intra-ISD beaconing* creates path segments from core ASes to non-core ASes. For this, the control service of a core AS creates PCBs and sends them to the non-core child ASes (typically customer ASes). The control service of a non-core child AS receives these PCBs and forwards them to its child ASes, and so on. This procedure continues until the PCB reaches an AS without any customer (leaf AS). As a result, all ASes within an ISD receive path segments to reach the core ASes of their ISD.

On its way, a PCB accumulates cryptographically protected path- and forwarding information per traversed AS. At every AS, metadata as well as information about the AS's ingress and egress interfaces are added to the PCB.


### Peering Links

PCBs do not traverse peering links. Instead, peering links are announced along with a regular path in a PCB. If both ASes at either end of a peering link have registered path segments that include this specific peering link, then it is possible to use this peering link during segment combination to create the end-to-end path.


### Extending a PCB

Every propagation period (as configured by the AS), the control service

- selects the best combinations of PCBs and interfaces connecting to a neighboring AS (i.e., a child AS or a core AS), and
- sends each selected PCB to the selected egress interface(s) associated with it.

For every selected PCB and egress interface combination, the AS extends the PCB by adding a so-called *AS entry* to the selected PCB. Such an AS entry includes a hop field that specifies the incoming (ingress) and outgoing (egress) interface for the packet forwarding through this AS, in the beaconing direction. The AS entry can also contain peer entries.

- For the specification of one PCB, see [](#pcbs)
- For more details on selecting PCBs, see [](#selection)
- For more details on propagating PCBs, see [](#path-segment-prop)


### One-Hop Paths

To propagate PCBs to neighboring ASes to which no forwarding path is available yet, SCION uses so-called *one-hop paths*. The One-Hop Path Type will be described in more detail in the SCION Data Plane specification (this document will be available later this year).



### PCB Propagation - Illustrated Examples

**Figure 2** below shows how intra-ISD PCB propagation works, from the ISD's core AS down to child ASes. For the sake of illustration, the interfaces of each AS are numbered with integer values. In practice, each AS can choose any encoding for its interfaces; in fact, only the AS itself needs to understand its encoding. Here, AS F receives two different PCBs via two different links from core AS Core. Moreover, AS F uses two different links to send two different PCBs to AS G, each PCB containing the respective egress interfaces. AS G extends the two PCBs and forwards both of them over a single link to a child AS.

**images/intra-isd-beaconing-paths.png**
Figure 2: *Intra-ISD PCB propagation from the ISD core to child ASes*

PCBs are used to explore paths within or between ISDs. As PCBs traverse the network, they accumulate path and forwarding information on AS-level. One could say that a PCB represents a single path segment that can be used to construct end-to-end forwarding paths. However, there is a difference between a PCB and a (registered) path segment. A PCB is a so-called "travelling path segment" that accumulates AS entries as it transits the network, as is shown in **Figure 2**. A (registered) path segment, instead, is a "snapshot" of a travelling PCB at a given time T and on a given AS interface X. This is illustrated by **Figure 3**. This figure shows several possible path segments to reach AS G. It is up to AS G to decide via which of these path segments it wants to be reached, and thus which path segments it will register.

**images/pcb-propagation.png**
Figure 3: *Possible up- or down-path segments for AS G*


## Path-Segment Construction Beacons (PCBs) {#pcbs}

This section provides a detailed specification of a single PCB and its message format.

**Note:** The SCION open-source implementation makes use of Protobuf (Protocol Buffers), a free and open-source cross-platform data format developed by Google and used to serialize structured data. The messages and remote procedure calls described below are in "proto3" language. For more information on Protobuf, see the official ["Protocol Buffers Documentation"](https://protobuf.dev/).


### Components of a PCB in Message Format {#pcb-compos}

{{figure-4}} graphically represents the PCB message format:

~~~~
PCB / PATH SEGMENT
+-------------+----------+-----------+-----------+-----------------+
|Segment Info |AS Entry 0| AS Entry 1|   ...     | AS Entry N      |
+-------------+----------+-----------+-----------+-----------------+
*- - - - # - -*          *- - -#- - -*
         |                     |
*- - - - v - - - *             |
+---------+------+             |
|Timestamp|Seg ID|             |
+---------+------+             |
                               |
                               |
                               |
*- - - - - - - - - - - - - - - v - - - - - - - - - - - - - - - - - - - - - - - *
+-----------------------+------------------------------------------------------+
|    Unsigned Ext.      |                   Signed AS Entry                    |
+-----------------------+------------------------------------------------------+
                        *- - - - - - - - - - - - # - - - - - - - - - - - - - - *
                                                 |
                                                 |
*- - - - - - - - - - - - - - - - - - - - - - - - v - - - - - - - - - - - - - - *
+--------------------+-----------------++--------------------------------------+
|     Signature      |    Header       ||                    Body              |
+--------------------+-----------------++--------------------------------------+
                     *- - - - # - - - -**- - - - - - - - - - # - - - - - - - - *
                              |                              |
*- - - - - - - - - - - - - - -v- - - - *                     |
+----------------+---------------------+                     |
| Signature Alg. | Verification Key ID |                     |
+----------------+---------------------+                     |
                 *- - - - - # - - - - -*                     |
                            |                                |
*- - - - - - - - - - - - - -v- - - - - - - - - -*            |
+---------+---------+------------+--------------+            |
| ISD-AS  |TRC Base | TRC Serial |Subject Key ID|            |
+---------+---------+------------+--------------+            |
                                                             |
                                                             |
                                                             |
*- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - v - - - - - - - - *
+------+-----------+-----------++--------------+------+--------------++---+----+
|ISD-AS|Next ISD-AS| Hop Entry || Peer Entry 0 | ...  | Peer Entry N ||MTU|Ext.|
+------+-----------+-----------++--------------+------+--------------++---+----+
                   *- - # - - -**- - - - # - - *
                        |                |
                        |                |
*- - - - - - - - - - - -v- *   *- - - - -v- - - - - - - - - - - - - - - - - - -*
+-------------+------------+   +----------+---------+-----------+--------------+
| Ingress MTU |  Hop Field |   |Hop Field | Peer MTU|Peer ISD-AS|Peer Interface|
+-------------+----+-------+   +-------+--+---------+-----------+--------------+
                   *- - - - -#- - - - -*
                             |
                             |
                             |
*- - - - - - - - - - - - - - v - - - - - - - - - - - - - - *
+------------+-------------+-------------------+-----------+
|  Ingress   |    Egress   |  Expiration Time  |   MAC     |
+------------+-------------+-------------------+-----------+
~~~~
{: #figure-4 title="Top-down composition of one PCB"}

The following sections provide detailed specifications of the PCB messages, starting with the top-level message of one PCB, and then diving deeper into each of the PCB's message components.


#### PCB Top-Level Message Format {#segment}

test

~~~~
+-------------+-------------+------------+------+------------+
|Segment Info | AS Entry 0  | AS Entry 1 |  ... | AS Entry N |
+-------------+-------------+------------+------+------------+
~~~~

Each PCB at least consists of:

- An information field with an identifier and a timestamp.
- Entries of all ASes on the path segment represented by this PCB.

The following code block defines the PCB on top level in Protobuf message format.

~~~~proto
message PathSegment {
    bytes segment_info = 1;
    repeated ASEntry as_entries = 2;
}
~~~~

- `segment_info`: This field is used as input for the PCB signature. It is the encoded version of the component `SegmentInformation`, which provides basic information about the PCB.  The `SegmentInformation` component is specified in detail in [](#seginfo).
- `as_entries`: Contains the `ASEntry` component of all ASes on the path segment represented by this PCB.
   - `ASEntry`: The `ASEntry` component contains the complete path information of a specific AS that is part of the path segment represented by the PCB. The `ASEntry` component is specified in detail in [](#ase-message).


#### Segment Information {#seginfo}

test

~~~~
+----------------------------+
|         Segment Info       |
+----------------------------+
*- - - - - - - # - - - - - - *
               |
               |
*- - - - - - - v - - - - - - *
+--------------+-------------+
|   Timestamp  |   Seg ID    |
+--------------+-------------+
~~~~

Each PCB MUST include an information component with basic information about the PCB.

In the Protobuf message format, the information component of a PCB is called the ``SegmentInformation`` message. The following code block shows the Protobuf message definition for the ``SegmentInformation`` message.

~~~~proto
message SegmentInformation {
    int64 timestamp = 1;
    uint32 segment_id = 2;
}
~~~~

- `timestamp`: The 64-bit timestamp indicates the creation time of this PCB. It is set by the originating core AS. The expiration time of the corresponding path segment is computed relative to this timestamp. The timestamp is encoded as the number of seconds elapsed since the POSIX Epoch (1970-01-01 00:00:00 UTC).
- `segment_id`: The 32-bit identifier of this PCB and the corresponding path segment. The segment ID is required for the computation of the message authentication code (MAC) of an AS's hop field. The MAC is used for hop field verification in the data plane. The `segment_id` field is an updatable field. The originating core AS MUST fill this field with a cryptographically random number. Each following AS MUST then update the segment ID correctly for the next AS to be able to verify the hop field in the data plane.

**Note:** See [](#hopfield) for more information on the hop field message format. The SCION Data Plane Specification provides a detailed description of the computation of the segment ID field and the verification of the hop field in the data plane.


#### AS Entry {#ase-message}

test

~~~~
                           +--------------+
                           |  AS Entry    |
                           +--------------+
                           *- - - -#- - - *
                                   |
                                   |
                                   |
*- - - - - - - - - - - - - - - - - v - - - - - - - - - - - - - - - *
+-----------------------+------------------------------------------+
|    Unsigned Ext.      |          Signed AS Entry                 |
+-----------------------+------------------------------------------+
~~~~

Beside the basic information component, each PCB MUST also contain the entries of all ASes included in the corresponding path segment. This means that the originating core AS MUST add its AS entry to each PCB it creates. During the beaconing process, also each traversed AS MUST attach its AS entry to the PCB.

One AS entry contains the complete hop information for this specific AS in this specific path segment. It consists of a signed and an unsigned component.

The code block below defines an AS entry `ASEntry` in Protobuf message format.

~~~~
   message ASEntry {
       SignedMessage signed = 1;
       PathSegmentUnsignedExtensions unsigned = 2;
   }
~~~~

It includes the following components:

- `SignedMessage`: The signed component of an AS entry. For the specification of this part of the AS entry, see :ref:`signed-compo` below.
- `PathSegmentUnsignedExtensions`: The unsigned and thus unprotected part of the AS entry. These are extensions with metadata that need no explicit protection.


#### AS Entry Signed Component {#signed-compo}


~~~~
        +------------------------------------------------------+
        |                   Signed AS Entry                    |
        +------------------------------------------------------+
        *- - - - - - - - - - - - -#- - - - - - - - - - - - - - *
                                  |
                                  |
*- - - - - - - - - - - - - - - - -v- - - - - - - - - - - - - - - - - -*
+--------------------+-----------------+------------------------------+
|      Header        |     Body        |            Signature         |
+--------------------+-----------------+------------------------------+
~~~~

Each AS entry of a PCB MUST include a signed component as well as a signature computed over the signed component. Each AS entry MUST be signed with a private key that corresponds to the public key certified by the AS's certificate.

This section specifies the signed component of an AS entry. The signed component of an AS entry MUST include the following elements:

- Header
- Body
- Signature

In the Protobuf message-format implementation, the signed component of an AS entry is specified by the `SignedMessage`. It consists of a header-and-body part `header_and_body`, and a raw signature `signature`. See also the code block below.


~~~~
   message SignedMessage {
       bytes header_and_body = 1;
       bytes signature = 2;
   }
~~~~

   // Low-level representation of HeaderAndBody used for signature computation
   // input. This should not be used by external code.

~~~~
   message HeaderAndBodyInternal {
       // Encoded header suitable for signature computation.
       bytes header = 1;
       // Raw payload suitable for signature computation.
       bytes body = 2;
   }
~~~~

- For the specification of the signed header, see :ref:`ase-header`.
- For the specification of the signed body, see :ref:`ase-sign`.
- For the specification of the ``signature`` field, see :ref:`sign`.


##### AS Entry Signed Header {#ase-header}

.. figure:: fig/signed-header.png
   :width: 75 %
   :figwidth: 100 %


The header part defines metadata that is relevant to (the computation and verification of) the signature. It MUST at least include the following metadata:

- The algorithm to compute the signature
- The identifier of the public key used to verify the signature (i.e., the public key certified by the AS's certificate)
- The ISD-AS number of the AS

The following code block defines the signed header of an AS entry in Protobuf message format (called the ``Header`` message).


.. code-block:: proto

   message Header {
       SignatureAlgorithm signature_algorithm = 1;
       bytes verification_key_id = 2;
       // Optional
       google.protobuf.Timestamp timestamp = 3;
       // Optional
       bytes metadata = 4;
       int32 associated_data_length = 5;
   }

   message VerificationKeyID {
       uint64 isd_as = 1;
       bytes subject_key_id = 2;
       uint64 trc_base = 3;
       uint64 trc_serial = 4;
   }


- `signature_algorithm`: Specifies the algorithm to compute the signature.
- ``verification_key_id``: Holds the serialized data defined by the ``VerificationKeyID`` message type. The ``VerificationKeyID`` message contains more information that is relevant to signing and verifying PCBs and other control-plane messages. The ``VerificationKeyID`` message type includes the following fields (see also the above code block):

  - ``isd_as``: The ISD-AS number of the current AS.
  - ``subject_key_id``: Refers to the certificate that contains the public key needed to verify this PCB's signature.
  - ``trc_base``: Defines the *base* number of the latest Trust Root Configuration (TRC) available to the signer at the time of the signature creation.
  - ``trc_serial``: Defines the *serial* number of the latest TRC available to the signer at the time of the signature creation.

  .. note::

     | For more information on signing and verifying PCBs, see :ref:`signing-verifying-cp-messages` in the chapter Certificate Specification of the SCION Control-Plane PKI Specification.
     | For more information on the TRC base and serial number, see the section :ref:`id` in the chapter Trust Root Configuration Specification of the SCION Control-Plane PKI Specification.

- ``timestamp``: Defines the signature creation timestamp. This field is optional.
- ``metadata``: Can be used to include arbitrary per-protocol metadata. This field is optional.
- ``associated_data_length``: Specifies the length of associated data that is covered by the signature, but is not included in the header and body. The value of this field is zero, if no associated data is covered by the signature.


##### AS Entry Signed Body {#ase-sign}

.. figure:: fig/signed-as-body.png
   :width: 75 %
   :figwidth: 100 %

The body of an AS entry consists of the signed component ``ASEntrySignedBody`` of all ASes in the path segment represented by the PCB, up until and including the current AS.

The following code block defines the signed body of one AS entry in Protobuf message format (called the ``ASEntrySignedBody`` message).

.. code-block:: proto

   message ASEntrySignedBody {
       uint64 isd_as = 1;
       uint64 next_isd_as = 2;
       HopEntry hop_entry = 3;
       repeated PeerEntry peer_entries = 4;
       uint32 mtu = 5;
       PathSegmentExtensions extensions = 6;
   }


- ``isd_as``: The ISD-AS number of the AS that created this AS entry.
- ``next_isd_as``: The ISD-AS number of the downstream AS to which the PCB should be forwarded.
- ``hop_entry``: The hop entry (``HopEntry``) with the information required to forward this PCB through the current AS to the next AS. This information is used in the data plane. For a specification of the hop entry, see :ref:`hopentry`.
- ``peer_entries``: The list of optional peer entries (``PeerEntry``). For a specification of one peer entry, see :ref:`peerentry`.
- ``mtu``: The size of the maximum transmission unit (MTU) within the current AS's network.
- ``extensions``: List of (signed) extensions (optional). PCB extensions defined here are part of the signed AS entry. This field should therefore only contain extensions that include important metadata for which cryptographic protection is required. For more information on PCB extensions, see :ref:`pcb-ext`.


##### AS Entry Signature {#sign}

The signature of an AS entry is computed over the AS entry's signed component. This is the input for the computation of the signature:

- The signed header and body of the current AS (``header_and_body``).
- The ``segment_info`` component of the current AS. This is the encoded version of the ``SegmentInformation`` component containing basic information about the path segment represented by the PCB. For the specification of ``SegmentInformation``, see :ref:`seginfo`.
- The signed [``header_and_body``/ ``signature``] combination of each previous AS on this specific path segment.

The following code block shows how the signature input is defined in the SCION Protobuf implementation ("ps" stands for path segment). Note that the signature has a nested, onion-like structure.

.. code-block::

   input(ps, i) = signed.header_and_body || associated_data(ps, i)

   associated_data(ps, i) = ps.segment_info ||
                            ps.as_entries[1].signed.header_and_body ||
                            ps.as_entries[1].signed.signature ||
                            ...
                            ps.as_entries[i-1].signed.header_and_body ||
                            ps.as_entries[i-1].signed.signature



#### Hop Entry {#hopentry}

.. figure:: fig/hop-entry.png
   :width: 75 %
   :figwidth: 100 %

Each body of an AS entry MUST contain exactly one hop entry component. The hop entry component specifies forwarding information for the data plane. The data plane requires this information to create the hop through the current AS (in the direction of the beaconing).

The following code block defines the hop entry component ``HopEntry`` in Protobuf message format:

.. code-block:: proto

   message HopEntry {
       HopField hop_field = 1;
       uint32 ingress_mtu = 2;
   }


- ``hop_field``: Contains the authenticated information about the ingress and egress interfaces in the direction of beaconing. The data plane needs this information to forward packets through the current AS. For further specifications, see :ref:`hopfield`.
- ``ingress_mtu``: Specifies the maximum transmission unit (MTU) of the ingress interface of the current AS.



#### Hop Field {#hopfield}

.. figure:: fig/hop-field.png
   :width: 75 %
   :figwidth: 100 %

The hop field, part of both hop entries and peer entries, is used directly in the data plane for packet forwarding: It specifies the incoming and outgoing interfaces of the ASes on the forwarding path. To prevent forgery, this information is authenticated with a message authentication code (MAC).

The following code block defines the hop field component ``HopField`` in Protobuf message format:

.. code-block:: proto

   message HopField {
       uint64 ingress = 1;
       uint64 egress = 2;
       uint32 exp_time = 3;
       bytes mac = 4;
   }

- ``ingress``: The 16-bit ingress interface identifier (in the direction of the path construction, that is, in the direction of beaconing through the current AS).

  .. note::

   For the AS that initiates the PCB, the ingress interface identifier MUST NOT be specified. This initiating AS is a core AS.

- ``egress``: The 16-bit egress interface identifier (in the direction of beaconing).
- ``exp_time``: The 8-bit encoded expiration time of the hop field, indicating how long the hop field is valid. This value is an offset relative to the PCB creation timestamp set in the PCB's segment information component (see also :ref:`seginfo`). By combining these two values, the AS can compute the absolute expiration time of the hop field. Data-plane packets that use the hop field after the expiration time MUST be dropped.
- ``mac``: The message authentication code (MAC) used in the data plane to verify the hop field.


#### Peer Entry {#peerentry}

.. figure:: fig/peer-entry-pcb.png
   :width: 75 %
   :figwidth: 100 %


By means of a peer entry, an AS can announce that it has a peering link to another AS. A peer entry is an optional component of a PCB - it is only included if there is a peering link to a peer AS.

The following code block defines the peer entry component ``PeerEntry`` in Protobuf message format:

.. code-block:: proto

   message PeerEntry {
       uint64 peer_isd_as = 1;
       uint64 peer_interface = 2;
       uint32 peer_mtu = 3;
       HopField hop_field = 4;
   }


- ``peer_isd_as``: The ISD-AS number of the peer AS. This number is used to match peering segments during path construction.
- ``peer_interface``: The 16-bit interface identifier of the peering link on the peer AS side. This identifier is used to match peering segments during path construction.
- ``peer_mtu``: Specifies the maximum transmission unit MTU on the peering link.
- ``hop_field``: Contains the authenticated information about the ingress and egress interfaces in the current AS (coming from the peering link, in the direction of beaconing - see also the next figure). The data plane needs this information to forward packets through the current AS. For further specifications, see :ref:`hopfield`.

.. figure:: fig/peer-entry.png
   :width: 50 %
   :figwidth: 100 %

   *Peer entry information, in the direction of beaconing*



### PCB Extensions {#pcb-ext}

In addition to basic routing information like hop entries and peer entries, PCBs can be used to communicate additional metadata, in its extensions. Extensions can be signed and unsigned. Signed extensions are protected by the AS signature, whereas unsigned extensions are not.

On code-level and in Protobuf message format, extensions are specified as follows:

- Unsigned extensions ``PathSegmentUnsignedExtensions`` are part of the AS entry component (the ``ASEntry`` message, see also :ref:`ase-message`).
- Signed extensions ``PathSegmentExtensions`` are part of the signed body component of an AS entry (the ``ASEntrySignedBody`` message, see also :ref:`ase-sign`).

.. note::

   SCION also supports so-called "detachable extensions". The detachable extension itself is part of a PCB's unsigned extensions, but a cryptographic hash of the detachable extension data is added to the signed extensions. Thus, a PCB with a detachable extension can be signed and verified without actually including the detachable extension in the signature. This prevents a possible processing overhead caused by large cryptographically-protected extensions.



## Propagation of PCBs {#path-prop}

This section describes how PCBs are selected and propagated in the path exploration process.


### Selection of PCBs to Propagate {#selection}

As an AS receives a series of intra-ISD or core PCBs, it must select the PCBs it will use to continue beaconing. Each AS must specify a local policy on the basis of which PCBs are evaluated, selected or eliminated. The selection process can be based on *path* properties (e.g., length, disjointness across different paths) as well as on *PCB* properties (e.g., age, remaining lifetime of sent instances) - each AS is free to use those properties that suit the AS best. The control service can then compute the overall quality of each candidate PCB based on these properties. For this, the AS should use a selection algorithm or metric that reflects its needs and requirements, and identifies the best PCBs or paths segments for this AS.


#### Storing and Selecting Candidate PCBs

When receiving a PCB, an AS first stores the PCB in a temporary storage for candidate PCBs, called the *beacon store*.

PCBs are propagated in batches to each connected downstream AS at a fixed frequency, the *propagation interval*. At each propagation event, each AS selects a set of the best PCBs from the candidates in the beacon store, according to the AS's selection policy. This set should have a fixed size, the *best PCBs set size*.

- The *best PCBs set size* SHOULD be at most "50" (PCBs) for intra-ISD beaconing and at most "5" (PCBs) for core beaconing.

.. note::

   Depending on the selection criteria, it may be necessary to keep more candidate PCBs than the *best PCBs set size* in the beacon store, to be able to determine the best set of PCBs. If this is the case, an AS should have a suitable pre-selection of candidate PCBs in place, in order to keep the beacon store capacity limited.

- The *propagation interval* SHOULD be at least "5" (seconds) for intra-ISD beaconing and at least "60" (seconds) for core beaconing.

.. note::

   Note that during bootstrapping and if the AS obtains a PCB containing a previously unknown path, the AS should forward the PCB immediately, to ensure quick connectivity establishment.


#### Selection Policy Example


The following figure illustrates the selection of path segments in three networks. Each network uses a different path property to select path segments. The selected path segments are represented by the bold or colored lines.

- The network on the left considers the *path length*, which is here defined as the number of hops from the originator core AS to the local AS. This number can give an indication of the path's latency.
- The network in the middle uses *peering links* as the selection criterion, that is, the number of different peering ASes from all non-core ASes on the PCB or path segment: A greater number of peering ASes increases the likelihood of finding a shortcut on the path segment.
- The network on the right selects PCBs based on *disjointness*. The disjointness of a PCB is calculated relative to the PCBs that have been previously sent. Paths can be either AS-disjoint or link-disjoint. AS-disjoint paths have no common upstream/core AS for the current AS, whereas link-disjoint paths do not share any AS-to-AS link. Depending on the objective of the AS, both criteria can be used: AS-disjointness allows path diversity in the event that an AS becomes unresponsive, and link-disjointness provides resilience in case of link failure.


.. figure:: fig/path-segment-selection.png
   :width: 75 %
   :figwidth: 100 %

   *Example networks to illustrate path-segment selection based on different path properties. The selected path segments are represented by the bold or colored lines.* \ :sup:`© ETH Zürich/Springer Verlag`



### Propagation of Selected PCBs {#path-segment-prop}

Once per *propagation period* (determined by each AS), an AS propagates selected PCBs to its neighboring ASes. This happens on the level of both intra-ISD beaconing and core beaconing. Both processes are described here.

To bootstrap the initial communication with a neighboring beacon service, ASes use so-called one-hop paths. This special
kind of path handles beaconing between neighboring ASes for which no forwarding path may be available yet. In fact, it
is the task of beaconing to discover such forwarding paths. The purpose of one-hop paths is thus to break this circular
dependency. The One-Hop Path Type is described in more detail in the SCION Data Plane specification.


#### Propagation - First Steps

The following first steps of the propagation procedure are the same for both intra-ISD and core beaconing:

1. Upon receiving a PCB, the control service of an AS verifies the structure and all signatures on the PCB.

   .. note::

      The PCB contains the version numbers of the trust root configuration(s) (TRC) and certificate(s) that must be used to verify its signatures. This enables the control service to check whether it has the relevant TRC(s) and certificate(s); if not, they can be requested from the control service of the sending AS.

2. As core beaconing is based on sending PCBs without a defined direction, it is necessary to avoid loops during path creation. The control service of core ASes therefore performs the following additional checks:
   - Does the PCB include an AS entry created by the core AS itself?

   If the answer to this question is yes, the PCB is discarded in order to avoid loops.

  .. note::

     It can be legitimate to cross the same ISD multiple times: For example, if the ISD spans a large geographical area, a path transiting another ISD may constitute a shortcut. However, it is up to each core AS to decide whether it wants to allow this.


3. If the PCB verification is successful, the control service decides whether to store the PCB as a candidate for propagation based on selection criteria and polices specific for each AS. For more information on the selection process, see :ref:`selection`.


#### Propagation of PCBs in Intra-ISD Beaconing {#intra-prop}

The propagation process in intra-ISD beaconing includes the following steps:

1. From the candidate PCBs stored in the beacon store, the control service of an AS selects the best PCBs to propagate to its downstream neighboring ASes, based on a selection algorithm specific for this AS.
2. The control service adds a new AS entry to every selected PCB. This AS entry MUST at least include:
   a. The ingress interface to this AS, in the hop field component (part of the AS entry's signed body component).
   b. The egress interface to the neighboring AS, also in the hop field component.
   c. The ISD_AS number of the next AS, in the signed body component of the AS entry.
   d. If the AS has peering links, the control service should add corresponding peer entry components to the signed body of the AS entry - one peer entry component for each peering link that the AS wants to advertise. The hop field component of each added peer entry must have a specified egress interface.
3. The control service MUST now sign each selected, extended PCB and append the computed signature.
4. As a final step, the control service propagates each extended PCB to the correct neighboring ASes, by invoking the ``SegmentCreationService.Beacon`` remote procedure call (RPC) in the control services of the neighboring ASes (see also :ref:`prop-proto`).


.. note::

   - For more information on the signed body component of an AS entry, see :ref:`ase-sign`.
   - For more information on a peer entry, see :ref:`peerentry`.
   - For more information on the hop field component, see :ref:`hopfield`.
   - For more information on signing an AS entry, see :ref:`sign`.


#### Propagation of PCBs in Core Beaconing


The propagation process in core beaconing includes the following steps:

1. The core control service selects the best PCBs to forward to neighboring core ASes observed so far.
2. The service adds a new AS entry to every selected PCB. This AS entry MUST at least include:
   a. The egress interface to the neighboring core AS, in the hop field component (part of the AS entry's signed body component).
   b. The ISD_AS number of the neighboring core AS, in the signed body component of the AS entry.
3. The core control service MUST now sign the extended PCBs.
4. As a final step, the service propagates the extended PCBs to the neighboring core ASes, by invoking the ``SegmentCreationService.Beacon`` remote procedure call (RPC) in the control services of the neighboring core ASes (see also :ref:`prop-proto`).


.. note::

   - For more information on the signed body component of an AS entry, see :ref:`ase-sign`.
   - For more information on signing an AS entry, see :ref:`sign`.
   - For more information on the hop field component, see :ref:`hopfield`.
   - For more information on signing an AS entry, see :ref:`sign`.



#### Propagation of PCBs in Protobuf Message Format {#prop-proto}

Step 4 of the above described propagation procedures is implemented as follows in Protobuf message format:


.. code-block:: proto

   service SegmentCreationService {
       rpc Beacon(BeaconRequest) returns (BeaconResponse) {}
   }

   message BeaconRequest {
       PathSegment segment = 1;
   }

   message BeaconResponse {}


The propagation procedure includes the following elements:


- ``SegmentCreationService``: Specifies the service via which the extended PCB is propagated to the control service of the neighboring AS.
  - ``Beacon``: Specifies the method that calls the control service at the neighboring AS in order to propagate the extended PCB.
- ``BeaconRequest``: Specifies the request message sent by the ``Beacon`` method to the control service of the neighboring AS. It contains the following element:
  - ``PathSegment``: Specifies the path segment to propagate to the neighboring AS. For more information on the Protobuf message type ``PathSegment``, see :ref:`segment`.
- ``BeaconResponse``: Specifies the response message from the neighboring AS.





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
