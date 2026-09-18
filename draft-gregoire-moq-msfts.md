---
title: "MPEG-2 Transport Stream Packaging for MOQT"
abbrev: "MOQT MPEG-2 TS Packaging"
category: info

docname: draft-gregoire-moq-msfts-latest
submissiontype: IETF
number:
date:
consensus: false
v: 3
area: "Applications and Real-Time"
workgroup: "Media Over QUIC"
keyword:
 - MOQ
 - MOQTransport
 - MPEG-2 Transport Stream
 - M2TS
 - MSF
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "mondain/msfts"
  latest: "https://mondain.github.io/msfts/draft-gregoire-moq-msfts.html"

author:
 -
    fullname: Paul Gregoire
    organization: Red5
    email: paul@red5.net
 -
    fullname: Gwendal Simon
    organization: Quortex
    email: gwendal.simon@quortex.io

normative:
  MOQTransport: I-D.draft-ietf-moq-transport
  MSF: I-D.draft-ietf-moq-msf-01
  ISO138181:
    title: "Information technology - Generic coding of moving pictures and associated audio information: Systems"
    author:
      org: ISO/IEC
    seriesinfo:
      ISO/IEC: 13818-1
    date: 2023
  BASE64: RFC4648
  SCTE35:
    title: "Digital Program Insertion Cueing Message"
    author:
      org: Society of Cable Telecommunications Engineers
    seriesinfo:
      ANSI/SCTE: 35 2023r1
    date: 2023-11

informative:
  LOC: I-D.draft-ietf-moq-loc
  SecureObjects: I-D.draft-ietf-moq-secure-objects
  C4M: I-D.draft-ietf-moq-c4m
  PrivacyPassAuth: I-D.draft-ietf-moq-privacy-pass-auth

--- abstract

This document extends the MOQT Streaming Format (MSF) catalog by defining the "mpeg2ts" packaging value for carrying MPEG-2
Transport Stream and M2TS source packets over MOQT. It defines
catalog-extension fields for transport-stream track description and specifies
subscriber behavior for joining, switching, and validating packetized streams.

--- middle

# Introduction {#introduction}

MPEG-2 Transport Stream MOQT Streaming Format (MSFTS) is an extension of the
MOQT Streaming Format (MSF) {{MSF}} that delivers MPEG-2 Transport Stream (TS)
{{ISO138181}} content over MOQT {{MOQTransport}}. MSFTS retains the scope,
capabilities, and features of MSF, including the catalog format, the timeline,
and alternate rendition switching.
MSFTS keeps the transport stream as the media container. A track carries
either 188-octet TS packets or 192-octet M2TS source packets, and the
publisher maps consecutive source packets into MOQT Objects. MSFTS is targeted
at publishers that already produce packetized transport streams, including
contribution feeds, broadcast distribution workflows, and systems that segment
transport streams for HTTP-based delivery.

This document describes version 2 of the MSFTS packaging format.

# MSF Extension {#msf-extension}

All specifications, requirements, and terminology defined in {{MSF}} apply to
implementations of this extension unless explicitly noted otherwise in this
document. MSFTS does not use the Low Overhead Media Container (LOC) {{LOC}}
packaging defined in {{MSF}}. This document defines the equivalent behavior
for mpeg2ts-packaged tracks.

This document uses two unrelated version numbers. The catalog `version` field
carries the MSF revision. The MSFTS format version given in {{introduction}}
identifies this packaging specification and never appears in a catalog. A
catalog conforming to this document MUST set `version` as {{MSF}} requires.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the following abbreviations from {{ISO138181}}: Packet
Identifier (PID), Program Association Table (PAT), Program Map Table (PMT),
Program Clock Reference (PCR), and Program Specific Information (PSI).

The following terms are used throughout this document:

TS packet:
: A 188-octet MPEG-2 Transport Stream packet as defined by {{ISO138181}}.

M2TS source packet:
: A 192-octet packet consisting of a four-octet source-packet timestamp followed
  by a 188-octet TS packet.

Source packet:
: Either a TS packet or an M2TS source packet. The catalog signals which of the
  two a track carries.

Subscriber:
: The MOQT endpoint that subscribes to a track and receives its Objects, as
  defined by {{MOQTransport}}. A subscriber operates on MOQT Objects and
  produces the reconstructed packet stream.

Receiver:
: The equipment that consumes the reconstructed packet stream, for example an
  Integrated Receiver Decoder (IRD). A receiver operates on source packets and
  needs no knowledge of MOQT. One implementation can act as both a subscriber
  and a receiver.

Random access point:
: A point in the packet stream at which a receiver can begin decoding after
  receiving the applicable transport-stream tables and decoder initialization.

Single-program transport stream:
: A transport stream whose PAT lists exactly one program.

Multi-program transport stream (MPTS):
: A transport stream whose PAT lists two or more programs.

# Scope

The purpose of MSFTS is to carry an MPEG-2 Transport Stream over
{{MOQTransport}} without changing the transport stream itself. Interoperability
implies that:

* An original publisher can map an incoming transport stream into MOQT Objects
  and Groups, describe it in an MSF catalog, and announce it to an MOQT relay.
* An MOQT relay can cache and propagate the tracks without parsing the
  transport stream.
* A final subscriber can parse the catalog, subscribe to the tracks it needs,
  reconstruct the packet stream, and pass it to a transport-stream decoder.

A subscriber needs to know how the publisher produced each track: the source
packet size, whether the publisher modified the stream, which program or
elementary stream the track carries, and where the timing reference lives.
MSFTS defines the catalog signaling that carries those decisions from the
publisher to the subscriber.

# Media Packaging {#media-packaging}

An mpeg2ts-packaged MOQT Track carries a single ordered packet stream. Each MOQT
Object payload is a concatenation of one or more whole source packets. The
publisher MUST NOT split a source packet across MOQT Objects.

When this packaging mode is used for a track, the MSF catalog `packaging` field
MUST be present and MUST be populated with the value "mpeg2ts".

## Object Payload Format {#object-payload-format}

The payload of each media Object is:

~~~ ascii-art
+===============+===============+=====+===============+
| source packet | source packet | ... | source packet |
+===============+===============+=====+===============+
~~~

The packet size is signaled by `mpeg2tsPacketSize` ({{mpeg2ts-packet-size}}). The
payload length of every media Object on an mpeg2ts track MUST be an integer
multiple of `mpeg2tsPacketSize`.

If `mpeg2tsPacketSize` is 188, every source packet MUST begin with the MPEG-2 TS
sync byte 0x47. If `mpeg2tsPacketSize` is 192, the TS packet begins four octets
after the start of each source packet and the octet at that position MUST be
0x47. A subscriber MUST treat a packet that fails this validation as invalid
media data for that track.

The source packets from received Objects are concatenated in ascending Group ID
and Object ID order to reconstruct the packet stream. A subscriber that skips
or fails to receive an Object MUST consider the reconstructed packet stream
discontinuous at that point until it reaches a subsequent random access point.

## Object Boundaries {#object-boundaries}

Object boundaries are packaging boundaries and do not change MPEG-2 Transport
Stream semantics. Continuity counters, adaptation fields, PCR, Presentation
Time Stamp (PTS), Decoding Time Stamp (DTS), PSI, and other transport-stream
syntax remain inside the source packets.

When `mpeg2tsModified` ({{mpeg2ts-modified}}) is false, a publisher MUST NOT modify
the continuity counter of any source packet and MUST NOT remap PIDs.
{{carriage-modes}} defines the modifications a publisher may make when
`mpeg2tsModified` is true.

## Group Boundaries {#group-boundaries}

For live single-program tracks, a publisher SHOULD start a new MOQT Group at
each point where the Group content is independently decodable without reference
to prior Groups. A publisher SHOULD place a random access point at the first
media Object of each Group, and the Group then includes the PAT and PMT packets
required for program demultiplexing.

{{unmodified-carriage}} defines Group boundary placement for tracks carrying a
whole multiplex.

The `mpeg2tsRandomAccess` field ({{mpeg2ts-random-access}}) declares whether every
Group in a track starts with a random access point. When `mpeg2tsRandomAccess` is
true, the first media Object in every Group MUST provide a valid random access
starting point for that Group.

## Packetization {#packetization}

Publishers SHOULD choose Object sizes that are large enough to amortize MOQT
object overhead and small enough to avoid excessive head-of-line delay at the
application layer. The number of source packets per Object can vary, but a
publisher SHOULD keep it stable within a track unless adapting to network or
encoder conditions.

If `mpeg2tsPacketsPerObject` is present, it declares the usual number of source
packets per media Object. The final Object of a Group MAY contain fewer source
packets. Subscribers MUST use the actual Object payload length rather than
assuming every Object has the declared size.

## Source Handling and Carriage Modes {#carriage-modes}

A publisher carries a transport stream in one of three modes, signaled by the
required `mpeg2tsModified` field ({{mpeg2ts-modified}}) and the optional
`mpeg2tsEsPid` field ({{mpeg2ts-es-pid}}). When `mpeg2tsModified` is false, the
publisher forwards the source packets without modification (unmodified
carriage). When `mpeg2tsModified` is true and `mpeg2tsEsPid` is absent, the
publisher has modified the stream at the program level (modified carriage).
When `mpeg2tsModified` is true and `mpeg2tsEsPid` is present, the track carries a
single elementary stream (ES-level carriage). The `mpeg2tsMpts` field
({{mpeg2ts-mpts}}) indicates whether the track carries a single program or a
whole multiplex, and it alone controls the presence of the per-program fields.

### Unmodified Carriage {#unmodified-carriage}

When `mpeg2tsModified` is false, the publisher forwards the source packets without
modification: no program selection, no packet identifier remap, no PAT or PMT
rewrite, and no insertion or removal of null packets. A subscriber can
reconstruct the source stream byte-for-byte.

When the source is a single-program transport stream, `mpeg2tsMpts` is false. The
source Program Association Table already lists exactly one program, so the
per-program fields `mpeg2tsProgramNumber`, `mpeg2tsPmtPid`, and `mpeg2tsPcrPid` SHOULD be
present to identify the carried program.

When the source is a multi-program transport stream, `mpeg2tsMpts` is true and all
source packets are emitted as received. Because no program is selected, the
per-program fields `mpeg2tsProgramNumber`, `mpeg2tsPmtPid`, and `mpeg2tsPcrPid` MUST be
absent, and per-track program subscription and the subscriber join behavior
defined in this document do not apply. Group boundary placement depends on
whether the publisher can identify random access points across the multiplex:
if it can, it MAY align Group boundaries to those points and set
`mpeg2tsRandomAccess` to true; otherwise it SHOULD start a new Group after a fixed
number of Objects.

### Modified Carriage {#modified-carriage}

When `mpeg2tsModified` is true, the publisher has changed the source stream, for
example by selecting a program, filtering packets, rewriting the PAT or PMT, or
adding or removing null packets. A publisher that makes any of these changes
MUST set `mpeg2tsModified` to true.

A publisher deriving a per-program track SHOULD filter the source packets so
that each track contains only:

* Null packets with PID 0x1FFF, which MAY be removed or
  retained at the publisher's discretion.
* Program Association Table packets (PID 0x0000), rewritten to list only the
  program present in this track.
* Program Map Table packets for the selected program (whose PID is listed in
  the Program Association Table entry for that program).
* All packets whose PID is listed in the Program Map Table of the selected
  program, including the PCR_PID and the PIDs of all elementary streams.

Removing null packets changes the inter-packet byte spacing that
constant-bit-rate receivers use to recover the mux clock. A subscriber
wishing to reconstruct a constant-bit-rate output stream cannot derive the
original rate from the stream alone; publishers that remove null packets
SHOULD declare the source mux rate using `mpeg2tsMuxRate` ({{mpeg2ts-mux-rate}}).

These rules apply to unscrambled transport stream sources. Publishers filtering
scrambled transport streams MUST also retain the conditional access packets
required for descrambling; conditional access integration is application-specific
and outside the scope of this document.

Filtering to a single program drops the service information (SI) tables,
because the PMT references only the elementary stream PIDs and the PCR PID of
the selected program. Digital Video Broadcasting (DVB) carries those tables on
fixed PIDs that no PMT lists:

| Table                                                | PID    |
|:=====================================================|:=======|
| Network Information Table (NIT)                      | 0x0010 |
| Service Description Table and Bouquet Association Table (SDT/BAT) | 0x0011 |
| Event Information Table (EIT)                        | 0x0012 |
| Time and Date Table and Time Offset Table (TDT/TOT)  | 0x0014 |
{: #si-tables title="DVB service information tables"}

The Advanced Television Systems Committee (ATSC) carries equivalent information
in its Program and System Information Protocol (PSIP) tables. A track filtered
without these tables has no service identity, no Electronic Program Guide
(EPG), and no broadcast time. A publisher producing tracks for broadcast or IRD
reception SHOULD retain the SI tables that the target standard requires, and
SHOULD declare their PIDs using `mpeg2tsSiPids` ({{mpeg2ts-si-pids}}) so that
subscribers can verify which tables are present.

SDT and EIT carried from an MPTS describe every program in the multiplex, so a
publisher retaining them SHOULD filter or rewrite them to leave only the
service and schedule entries for the carried program. NIT, TDT, and TOT are
broadcast-wide and need no per-program rewriting.

The `mpeg2tsProgramNumber` field ({{mpeg2ts-program-number}}) SHOULD be present on
per-program tracks to identify the program carried. When multiple per-program
tracks are derived from the same MPTS source, the publisher SHOULD use the MSF
`altGroup` field if the programs are alternate renditions of the same content;
programs that are independent services SHOULD be published as separate tracks.

### Modified ES-Level Carriage {#es-level-carriage}

When `mpeg2tsEsPid` ({{mpeg2ts-es-pid}}) is present, the track carries a single
elementary stream or signaling table. The track payload contains only TS
packets for the PID identified by `mpeg2tsEsPid`; PAT, PMT, and null packets are
not included. Publishers SHOULD use the MSF `initDataList` field to carry the
PAT and PMT of the originating program so that subscribers can identify the
program structure before processing elementary-stream packets.
`mpeg2tsPsiInterval` MUST be absent, because the track payload contains no PSI.

When `mpeg2tsPcrPid` equals `mpeg2tsEsPid`, the track embeds the Program Clock
Reference and provides the timing reference for the program. When `mpeg2tsPcrPid`
identifies a different PID, that PID is carried by another track; a subscriber
requiring PCR-based timing MUST subscribe to the track carrying that PID.

A publisher producing multiple ES-level tracks for the same program MUST align
Group boundaries across all those tracks so that matching Group numbers
correspond to the same presentation position. This alignment allows a
subscriber to combine ES-level tracks reliably. A subscriber combining
multiple ES-level tracks into a single TS output MUST construct a PAT listing
the carried program and a PMT listing the PIDs of all subscribed ES-level
tracks, and MUST interleave packets from all tracks. The subscriber sources
PCR from the track where `mpeg2tsPcrPid` equals `mpeg2tsEsPid`.


## PCR and Timing {#pcr-timing}

The PCR is carried inside adaptation fields of
transport-stream packets as defined by {{ISO138181}}. MOQT Object and Group
boundaries are packaging boundaries and do not alter PCR continuity within a
track.

A publisher MUST NOT introduce a PCR discontinuity within a single MOQT Group.
A publisher that introduces a PCR discontinuity between consecutive MOQT Groups
MUST signal it by setting the discontinuity_indicator bit (ISO 13818-1
Section 2.4.3.5) in the adaptation field of the first TS packet carrying PCR
in the new Group.

Note: Hardware IRDs recover the mux clock from the rate at which PCR-bearing
packets arrive, not only from their encoded values. MOQT does not guarantee
that Object delivery preserves the inter-packet timing of the source stream.
Deployments targeting such receivers should account for this constraint and may
require a rate-controlled egress that re-paces packets according to the source
mux rate.

Note: The 33-bit PCR base field wraps around after approximately 26.5 hours of
continuous stream time. For long-running live streams this is a normal event;
receivers should handle it as a continuous timeline continuation rather than a
discontinuity. Subscribers that use the MSF Media Timeline {{MSF}} for playout
timing can rely on its monotonic wall-clock abstraction independently of PCR
wrap-around.

## Splice Signaling {#splice-signaling}

SCTE-35 {{SCTE35}} splice information is carried transparently in the TS
stream as splice_info_section() messages on their designated PID. Publishers
MAY surface splice events via the MSF Event Timeline {{MSF}}. This document
does not specify SCTE-35 processing.

# Catalog {#catalog}

An mpeg2ts track is described by the MSF catalog {{MSF}}. This document extends
that catalog by defining the `mpeg2ts` value for the inherited `packaging` field
and additional fields for track objects that use that value. The catalog track
name, root catalog fields, common track fields, delta update rules, variable
substitution rules, and authorization signaling are inherited unchanged from
MSF unless this document explicitly states otherwise. A parser MUST ignore
fields it does not understand.

## Track Object Fields {#track-fields}

{{track-fields-table}} lists the mpeg2ts-specific fields defined within a track
object.

| Field                         | Name                    | Definition |
|:==============================|:========================|:===========|
| Packet size                   | mpeg2tsPacketSize           | {{mpeg2ts-packet-size}} |
| Stream modified               | mpeg2tsModified             | {{mpeg2ts-modified}} |
| Packets per Object            | mpeg2tsPacketsPerObject     | {{mpeg2ts-packets-per-object}} |
| Program number                | mpeg2tsProgramNumber        | {{mpeg2ts-program-number}} |
| PMT PID                       | mpeg2tsPmtPid               | {{mpeg2ts-pmt-pid}} |
| PCR PID                       | mpeg2tsPcrPid               | {{mpeg2ts-pcr-pid}} |
| PSI interval                  | mpeg2tsPsiInterval          | {{mpeg2ts-psi-interval}} |
| Mux rate                      | mpeg2tsMuxRate              | {{mpeg2ts-mux-rate}} |
| SI PIDs                       | mpeg2tsSiPids               | {{mpeg2ts-si-pids}} |
| Random access                 | mpeg2tsRandomAccess         | {{mpeg2ts-random-access}} |
| Timestamp mode                | mpeg2tsTimestampMode        | {{mpeg2ts-timestamp-mode}} |
| SCTE-35 PID                   | mpeg2tsScte35Pid            | {{mpeg2ts-scte35-pid}} |
| ES PID                        | mpeg2tsEsPid                | {{mpeg2ts-es-pid}} |
| MPTS                          | mpeg2tsMpts                 | {{mpeg2ts-mpts}} |
{: #track-fields-table title="Track object fields defined by this document"}

Use of the MSF `initRef` and `initDataList` fields by mpeg2ts tracks is described
in {{init-data}}.

## Packet Size {#mpeg2ts-packet-size}

Required: Yes    JSON Type: Number    Location: Track Object

The source-packet size in octets. The value MUST be either 188 or 192. A value
of 188 identifies ordinary MPEG-2 TS packets. A value of 192 identifies M2TS
source packets with a four-octet timestamp prefix followed by a 188-octet TS
packet.

## Stream Modified {#mpeg2ts-modified}

Required: Yes    JSON Type: Boolean    Location: Track Object

When true, the published packet stream is not a byte-for-byte copy of the
source. The publisher has changed it, for example by selecting a program,
filtering packets, rewriting the PAT or PMT, or adding or removing null packets.
When false, the publisher MUST forward the source packets without modification,
so a subscriber can reconstruct the source stream byte-for-byte.

## Packets per Object {#mpeg2ts-packets-per-object}

Required: Optional    JSON Type: Number    Location: Track Object

The usual number of source packets carried by each media Object. This field is
advisory. Subscribers MUST validate each Object using its actual payload length.

## Program Number {#mpeg2ts-program-number}

Required: Optional    JSON Type: Number    Location: Track Object

The MPEG-2 Transport Stream program number carried by this track. When
present, the track SHOULD carry packets from only that program. When absent
and `mpeg2tsMpts` is not true, subscribers MAY select a program using local
policy or transport-stream signaling. This field MUST be absent when
`mpeg2tsMpts` is true.

## PMT PID {#mpeg2ts-pmt-pid}

Required: Optional    JSON Type: Number    Location: Track Object

The packet identifier carrying the Program Map Table for `mpeg2tsProgramNumber`.
This field is advisory and does not replace the Program Association Table or
Program Map Table carried in the transport stream. It MUST be absent when
`mpeg2tsMpts` is true. It MUST be absent when `mpeg2tsEsPid` is present, because
an ES-level track does not carry a PMT in its payload.

## PCR PID {#mpeg2ts-pcr-pid}

Required: Optional    JSON Type: Number    Location: Track Object

The packet identifier carrying the Program Clock Reference for the program
identified by `mpeg2tsProgramNumber`. This field is advisory and does not
replace PCR signaling in the transport stream. It MUST be absent when
`mpeg2tsMpts` is true. When `mpeg2tsEsPid` is present, this PID MAY be carried by
a different track in the same session; a subscriber requiring PCR-based timing
MUST subscribe to the track where `mpeg2tsPcrPid` equals `mpeg2tsEsPid`.

## PSI Interval {#mpeg2ts-psi-interval}

Required: Optional    JSON Type: Number    Location: Track Object

The maximum interval, in milliseconds, at which the publisher expects the
Program Association Table and Program Map Table to repeat in the packet stream.
For single-program tracks, publishers SHOULD repeat PSI at an interval no
larger than this value for live content. For `mpeg2tsMpts` tracks, the publisher
does not control PSI injection; when present, this field describes the source
multiplex PSI repetition rate and is advisory only. Subscribers MAY use this
value to estimate join latency in both modes. This field MUST be absent when
`mpeg2tsEsPid` is present, because ES-level tracks carry no PSI in their payload.

## Mux Rate {#mpeg2ts-mux-rate}

Required: Optional    JSON Type: Number    Location: Track Object

The nominal source mux rate of the transport stream in bits per second.
This field is advisory. A subscriber reconstructing a constant-bit-rate
output stream MAY use this value to restore the original mux rate when null
packets have been removed. This field MUST be absent when `mpeg2tsEsPid` is
present.

## SI PIDs {#mpeg2ts-si-pids}

Required: Optional    JSON Type: Array    Location: Track Object

The packet identifiers of SI tables retained in the filtered track, in addition
to those listed in the Program Map Table. This field is advisory. Publishers
SHOULD include this field when they retain DVB or ATSC SI tables. Subscribers
MAY use this list to verify which service information tables are present without
inspecting the packet stream. This field MUST be absent when `mpeg2tsEsPid` is
present; at ES-level granularity, each SI table is published as a separate
track identified by `mpeg2tsEsPid` and the MSF `role` field.

## Random Access {#mpeg2ts-random-access}

Required: Optional    JSON Type: Boolean    Location: Track Object

When true, every MOQT Group starts with a random access point, as defined in
{{group-boundaries}}. When absent or false, this document makes no guarantee
about where in a Group decoding can begin.

## Timestamp Mode {#mpeg2ts-timestamp-mode}

Required: Optional    JSON Type: String    Location: Track Object

For 192-octet source packets, this field identifies the interpretation of the
four-octet source-packet timestamp. The value "arrival-time" indicates an
arrival-time or emission-time stamp associated with the following TS packet. The
value "opaque" indicates that the timestamp prefix is carried without specified
semantics. This field MUST NOT be present when `mpeg2tsPacketSize` is 188.

## SCTE-35 PID {#mpeg2ts-scte35-pid}

Required: Optional    JSON Type: Number    Location: Track Object

The PID carrying SCTE-35 splice_info_section() messages for this track. This
field is advisory; SCTE-35 messages are also discoverable via the PMT
conditional access or registration descriptor. When present, receivers MAY
use this value to locate splice events without parsing the PMT. Publishers
SHOULD include this field when the track carries SCTE-35 splice signaling.
This field MUST be absent when `mpeg2tsEsPid` is present; when SCTE-35 is
published as an ES-level track, the track's `mpeg2tsEsPid` and `role` fields
identify it.

## ES PID {#mpeg2ts-es-pid}

Required: Optional    JSON Type: Number    Location: Track Object

The Packet Identifier of the single elementary stream or signaling table
carried by this track. When present, the track carries only TS packets for
this PID; it does not carry PAT, PMT, or null packets. This field MUST be
absent when `mpeg2tsMpts` is true. When `mpeg2tsEsPid` is present, `mpeg2tsModified`
MUST be true, `mpeg2tsPmtPid` MUST be absent, `mpeg2tsSiPids` MUST be absent, and
`mpeg2tsScte35Pid` MUST be absent.

For tracks carrying DVB or ATSC service information tables, publishers SHOULD
set the MSF `role` field to one of the following values: `"nit"` for the
Network Information Table (PID 0x0010), `"sdt"` for the Service Description
Table and Bouquet Association Table (PID 0x0011), `"eit"` for the Event
Information Table (PID 0x0012), and `"tdt"` for the Time and Date Table and
Time Offset Table (PID 0x0014). For tracks carrying SCTE-35 splice
information, publishers SHOULD set `role` to `"scte35"`. For media
elementary streams, publishers SHOULD set `role` to the MSF-defined value for
the stream type, for example `"video"` or `"audio"`.

## MPTS {#mpeg2ts-mpts}

Required: Optional    JSON Type: Boolean    Location: Track Object

When true, this track carries a multi-program transport stream without program
selection or PID filtering. `mpeg2tsProgramNumber`, `mpeg2tsPmtPid`, and
`mpeg2tsPcrPid` MUST be absent when this field is true.

## Use of MSF Initialization Data {#init-data}

The `initRef` track field and the root `initDataList` field are defined by MSF;
they are not fields defined by this extension. An mpeg2ts track MAY use those
fields to carry initialization data. The track sets `initRef` to the `id` of
an `initDataList` entry whose `type` MUST be "inline". The Base64 {{BASE64}}
decoded value of the entry's `data` field MUST be a sequence of whole source
packets using the packet size declared by `mpeg2tsPacketSize`.

Codec-level initialization data does not need an `initDataList` entry.
Parameter sets travel in band within the elementary stream and are therefore
present whenever a random access point is included.

Publishers SHOULD include current PAT and PMT packets in the referenced
initialization data when those tables are not guaranteed to be available at the
first Object of each Group. When PSI changes within a live track, the
publisher SHOULD publish an updated initialization data entry in a new
independent catalog before publishing media Objects that rely on the changed
PSI. An update to the root `initDataList` MUST NOT be expressed as an MSF delta
update. Subscribers MUST NOT assume that referenced initialization data remains
valid after the MPEG-2 PSI `version_number` changes; updated PSI in media
Objects takes precedence.

For `mpeg2tsMpts` tracks, producing referenced initialization data requires
extracting the PAT and all program PMTs from the source multiplex. Publishers
that do not inspect the source stream typically omit `initRef` and the
corresponding `initDataList` entry; subscribers will encounter PSI within one
PSI repetition cycle regardless of Group boundaries.

# Catalog Examples {#catalog-examples}

The following examples are non-normative.

## Live 188-octet Transport Stream {#example-live-ts}

~~~ json
{
  "version": "draft-01",
  "generatedAt": 1746104606044,
  "tracks": [
    {
      "name": "program-1-ts",
      "namespace": "live.example.com/channel/1",
      "packaging": "mpeg2ts",
      "mpeg2tsModified": false,
      "isLive": true,
      "targetLatency": 1000,
      "role": "video",
      "mimeType": "video/mp2t",
      "bitrate": 6000000,
      "mpeg2tsPacketSize": 188,
      "mpeg2tsPacketsPerObject": 64,
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPmtPid": 256,
      "mpeg2tsPcrPid": 257,
      "mpeg2tsPsiInterval": 100,
      "mpeg2tsRandomAccess": true
    }
  ]
}
~~~

## Live 192-octet M2TS Source Packets {#example-live-m2ts}

~~~ json
{
  "version": "draft-01",
  "generatedAt": 1746104606044,
  "tracks": [
    {
      "name": "program-1-mpeg2ts",
      "namespace": "contribution.example.net/feed/a",
      "packaging": "mpeg2ts",
      "mpeg2tsModified": false,
      "isLive": true,
      "targetLatency": 500,
      "role": "video",
      "mimeType": "video/mp2t",
      "bitrate": 12000000,
      "mpeg2tsPacketSize": 192,
      "mpeg2tsPacketsPerObject": 32,
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsTimestampMode": "arrival-time",
      "mpeg2tsRandomAccess": true
    }
  ]
}
~~~

## Video-on-Demand Transport Stream {#example-vod}

~~~ json
{
  "version": "draft-01",
  "tracks": [
    {
      "name": "asset-main",
      "namespace": "vod.example.com/assets/1000",
      "packaging": "mpeg2ts",
      "mpeg2tsModified": false,
      "isLive": false,
      "trackDuration": 632000,
      "role": "video",
      "mimeType": "video/mp2t",
      "bitrate": 4500000,
      "mpeg2tsPacketSize": 188,
      "mpeg2tsPacketsPerObject": 96,
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsRandomAccess": true
    }
  ]
}
~~~

## Multi-Program Source - Per-Program Tracks {#example-mpts}

This example shows a catalog for a publisher that receives a 2-program
transport stream and publishes each program as a separate mpeg2ts track. The
two tracks share a namespace but are independent services; `altGroup` is not
used because the programs carry different content.

~~~ json
{
  "version": "draft-01",
  "generatedAt": 1746104606044,
  "tracks": [
    {
      "name": "program-1",
      "namespace": "live.example.com/mux/1",
      "packaging": "mpeg2ts",
      "mpeg2tsModified": true,
      "isLive": true,
      "targetLatency": 1000,
      "role": "video",
      "mimeType": "video/mp2t",
      "bitrate": 6000000,
      "mpeg2tsPacketSize": 188,
      "mpeg2tsPacketsPerObject": 64,
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPmtPid": 256,
      "mpeg2tsPcrPid": 257,
      "mpeg2tsPsiInterval": 100,
      "mpeg2tsRandomAccess": true
    },
    {
      "name": "program-2",
      "namespace": "live.example.com/mux/1",
      "packaging": "mpeg2ts",
      "mpeg2tsModified": true,
      "isLive": true,
      "targetLatency": 1000,
      "role": "video",
      "mimeType": "video/mp2t",
      "bitrate": 4000000,
      "mpeg2tsPacketSize": 188,
      "mpeg2tsPacketsPerObject": 64,
      "mpeg2tsProgramNumber": 2,
      "mpeg2tsPmtPid": 512,
      "mpeg2tsPcrPid": 513,
      "mpeg2tsPsiInterval": 100,
      "mpeg2tsRandomAccess": true
    }
  ]
}
~~~

## Transparent MPTS Carriage {#example-mpts-transparent}

This example shows a catalog for a publisher that carries a complete
multi-program transport stream without program selection, so no per-program
catalog fields are present. The `mpeg2tsPsiInterval` field is included as an
advisory hint; its value is not normative for MPTS tracks.

~~~ json
{
  "version": "draft-01",
  "generatedAt": 1746104606044,
  "tracks": [
    {
      "name": "mux-1",
      "namespace": "live.example.com/mux/1",
      "packaging": "mpeg2ts",
      "mpeg2tsModified": false,
      "isLive": true,
      "targetLatency": 1000,
      "mimeType": "video/mp2t",
      "bitrate": 20000000,
      "mpeg2tsPacketSize": 188,
      "mpeg2tsPacketsPerObject": 64,
      "mpeg2tsMpts": true,
      "mpeg2tsPsiInterval": 100
    }
  ]
}
~~~

## Alternate Renditions - Two Bitrate Tracks {#example-abr}

This example shows a catalog for a live channel published at two bitrates as
alternate renditions. Both tracks are in the same `altGroup`; video tracks
MUST align Group boundaries at identical presentation positions. The tracks
use different PID assignments: a subscriber switching between them MUST re-parse
PAT and PMT on the new track before routing packets to a decoder.

~~~ json
{
  "version": "draft-01",
  "generatedAt": 1746104606044,
  "tracks": [
    {
      "name": "video-high",
      "namespace": "live.example.com/channel/1",
      "packaging": "mpeg2ts",
      "mpeg2tsModified": false,
      "isLive": true,
      "targetLatency": 1000,
      "role": "video",
      "mimeType": "video/mp2t",
      "bitrate": 6000000,
      "altGroup": 1,
      "mpeg2tsPacketSize": 188,
      "mpeg2tsPacketsPerObject": 64,
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPmtPid": 256,
      "mpeg2tsPcrPid": 257,
      "mpeg2tsPsiInterval": 100,
      "mpeg2tsRandomAccess": true
    },
    {
      "name": "video-low",
      "namespace": "live.example.com/channel/1",
      "packaging": "mpeg2ts",
      "mpeg2tsModified": false,
      "isLive": true,
      "targetLatency": 1000,
      "role": "video",
      "mimeType": "video/mp2t",
      "bitrate": 2000000,
      "altGroup": 1,
      "mpeg2tsPacketSize": 188,
      "mpeg2tsPacketsPerObject": 64,
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPmtPid": 512,
      "mpeg2tsPcrPid": 513,
      "mpeg2tsPsiInterval": 100,
      "mpeg2tsRandomAccess": true
    }
  ]
}
~~~

## ES-Level Tracks - Per-Elementary-Stream Publishing {#example-es-level}

This example shows a live program published as separate ES-level tracks: one
video track carrying the PCR, two audio tracks for different languages (English
and Spanish), and one Event Information Table track. The video and audio
tracks MUST have synchronized Group boundaries. A subscriber combines the
video track and the audio track of its choice by constructing a PAT and PMT
listing the subscribed PIDs and sourcing PCR from the video track (PID 257).

~~~ json
{
  "version": "draft-01",
  "generatedAt": 1746104606044,
  "tracks": [
    {
      "name": "program-1-video",
      "namespace": "live.example.com/channel/1",
      "packaging": "mpeg2ts",
      "mpeg2tsModified": true,
      "isLive": true,
      "targetLatency": 1000,
      "role": "video",
      "mimeType": "video/mp2t",
      "bitrate": 5000000,
      "mpeg2tsPacketSize": 188,
      "mpeg2tsPacketsPerObject": 64,
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPcrPid": 257,
      "mpeg2tsEsPid": 257,
      "mpeg2tsRandomAccess": true
    },
    {
      "name": "program-1-audio-en",
      "namespace": "live.example.com/channel/1",
      "packaging": "mpeg2ts",
      "mpeg2tsModified": true,
      "isLive": true,
      "targetLatency": 1000,
      "role": "audio",
      "mimeType": "video/mp2t",
      "bitrate": 128000,
      "mpeg2tsPacketSize": 188,
      "mpeg2tsPacketsPerObject": 32,
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPcrPid": 257,
      "mpeg2tsEsPid": 258,
      "mpeg2tsRandomAccess": true
    },
    {
      "name": "program-1-audio-es",
      "namespace": "live.example.com/channel/1",
      "packaging": "mpeg2ts",
      "mpeg2tsModified": true,
      "isLive": true,
      "targetLatency": 1000,
      "role": "audio",
      "mimeType": "video/mp2t",
      "bitrate": 128000,
      "mpeg2tsPacketSize": 188,
      "mpeg2tsPacketsPerObject": 32,
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPcrPid": 257,
      "mpeg2tsEsPid": 259,
      "mpeg2tsRandomAccess": true
    },
    {
      "name": "program-1-eit",
      "namespace": "live.example.com/channel/1",
      "packaging": "mpeg2ts",
      "mpeg2tsModified": true,
      "isLive": true,
      "role": "eit",
      "mimeType": "video/mp2t",
      "mpeg2tsPacketSize": 188,
      "mpeg2tsPacketsPerObject": 16,
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsEsPid": 18
    }
  ]
}
~~~

# Subscriber Processing {#subscriber-processing}

A subscriber obtains the catalog using the MSF catalog workflow and subscribes
to one or more mpeg2ts tracks. For each received media Object, the subscriber:

1. Validates that the payload length is a non-zero integer multiple of
   `mpeg2tsPacketSize`.
2. Validates the TS sync byte position for each source packet.
3. Reconstructs the packet stream by appending the source packets in MOQT object
   order.
4. Applies normal MPEG-2 Transport Stream demultiplexing, timing recovery, and
   decoder initialization.

If validation fails, the subscriber SHOULD discard the invalid Object and treat
the reconstructed packet stream as discontinuous. A subscriber MAY continue
processing at the next Object, but it SHOULD wait for a random access point
before presenting decoded media.

When joining a live track, a subscriber SHOULD start at the newest Group whose
first Object is available when `mpeg2tsRandomAccess` is true. Otherwise, a
subscriber SHOULD select a starting Group far enough back to encompass at least
one complete PSI repetition cycle before its target presentation time; when
`mpeg2tsPsiInterval` is declared, that value bounds the maximum look-back interval
needed. A subscriber MAY use the MSF Media Timeline {{MSF}} to resolve this
time bound to a concrete MOQT Group location for use with a Joining FETCH
{{MOQTransport}}. A subscriber MUST NOT begin media presentation until it has
received a valid PAT and PMT for the program to be decoded.

When a subscriber receives ES-level tracks ({{es-level-carriage}}), it MUST
align on the same starting Group number across all subscribed ES-level tracks
for the same program before combining them. The subscriber constructs the combined TS
output by building a PAT listing the carried program and a PMT listing the PIDs
of all subscribed ES-level tracks, then interleaving packets from all tracks.
PCR is sourced from the track where `mpeg2tsPcrPid` equals `mpeg2tsEsPid`. A
subscriber MUST NOT begin media presentation until it has received at least
one Group from each subscribed ES-level track and has obtained the originating
program's PAT and PMT, either from `initDataList` or from the packet stream.

# Switching and Alternate Renditions {#switching}

Tracks with `mpeg2tsMpts` set to true MUST NOT be included in an `altGroup`,
because Adaptive Bitrate (ABR) switching semantics require per-program Group
alignment and PCR continuity that transparent carriage does not guarantee.

Multiple mpeg2ts tracks can be advertised as alternatives using the MSF `altGroup`
field. Video tracks in the same alternate group MUST place Group boundaries at
identical presentation positions; other tracks SHOULD align their Group
boundaries to the same positions where possible. All tracks in the alternate
group SHOULD set `mpeg2tsRandomAccess` to true. This ensures that a subscriber
can switch between alternate video tracks at any Group boundary without
encountering a misaligned access point.
A subscriber SHOULD switch between alternate mpeg2ts tracks only at Group
boundaries or at transport-stream random access points that it can
independently decode.

This document does not require continuity counter values or PID assignments to
match across alternate tracks. Subscribers MUST treat a switch between tracks as
a packet-stream discontinuity unless application-specific signaling establishes
stronger continuity.

A receiver MUST treat a switch between alternate tracks as a PCR discontinuity
and MUST re-initialize its system time clock (STC) recovery using the first PCR
value received on the new track as the initial reference. In addition to the
Group boundary alignment requirements above, publishers providing alternate
tracks SHOULD align presentation timestamps at Group boundaries across tracks
to enable seamless presentation switching at the application layer.
Because PID assignments need not match across alternate tracks, a receiver
MUST re-parse the PAT and PMT of the new track after every track switch before
routing elementary-stream packets to a decoder.

# Content Protection {#content-protection}

This packaging format preserves any scrambling or conditional access information
present in the MPEG-2 Transport Stream. Transport-stream scrambling is opaque
to MOQT relays and to this specification.

Object-level encryption MAY be applied using a mechanism such as MOQ Secure
Objects {{SecureObjects}} when signaled by the catalog. When object-level
encryption is used, source packet validation is performed after successful
decryption.

# Authorization {#authorization}

Authorization requirements can be advertised using MSF catalog authorization
fields. For example, a publisher can use Common Access Token signaling
{{C4M}}, Privacy Pass authorization {{PrivacyPassAuth}}, or an application
defined authorization scheme.

# Security Considerations {#security-considerations}

The security considerations of MOQT {{MOQTransport}}, MSF {{MSF}}, MPEG-2
Transport Stream {{ISO138181}}, and any object encryption scheme apply.

Receivers need to treat transport-stream syntax as untrusted input. Invalid
packet sizes, invalid sync bytes, malformed PSI, inconsistent continuity
counters, excessive table repetition, and timestamp discontinuities can cause
decoder failures or resource exhaustion if not bounded by implementation policy.

Catalog metadata is also untrusted input. Subscribers MUST validate packet
sizes, payload lengths, Base64 values, PIDs, program numbers, and object
ordering before using the values to allocate memory or configure decoders.

Object-level encryption protects MOQT Object payloads but does not hide MOQT
namespace, track name, Group ID, Object ID, object size, or delivery timing from
authorized relays. Applications that require confidentiality for media payloads
SHOULD use an object encryption scheme in addition to transport security.

# IANA Considerations {#iana-considerations}

This document requests that, once MSF establishes an IANA registry for packaging
values, IANA register the value "mpeg2ts" with this document as the reference.

--- back

# Acknowledgments

This document follows the repository and draft structure used by the MOQT
Streaming Format work.
