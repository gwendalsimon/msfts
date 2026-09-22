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
  ISO138189:
    title: "Information technology - Generic coding of moving pictures and
            associated audio information - Part 9: Extension for real time
            interface for systems decoders"
    author:
      org: ISO/IEC
    seriesinfo:
      ISO/IEC: 13818-9
    date: 1996
  TR101290:
    title: "Digital Video Broadcasting (DVB); Measurement guidelines for DVB
            systems"
    author:
      org: European Telecommunications Standards Institute
    seriesinfo:
      ETSI TR: 101 290 V1.4.1
    date: 2020-06
  SCTE35Timeline: I-D.draft-wilaw-moq-scte35-event-timeline
  DVBSI:
    title: "Digital Video Broadcasting (DVB); Specification for Service
            Information (SI) in DVB systems"
    author:
      org: European Telecommunications Standards Institute
    seriesinfo:
      ETSI EN: 300 468 V1.19.1
    date: 2025-02
  ATSCPSIP:
    title: "ATSC Standard: Program and System Information Protocol for
            Terrestrial Broadcast and Cable"
    author:
      org: Advanced Television Systems Committee
    seriesinfo:
      ATSC: A/65:2013
    date: 2013
  SecureObjects: I-D.draft-ietf-moq-secure-objects

--- abstract

This document extends the MOQT Streaming Format (MSF) catalog by defining the
"mpeg2ts" packaging value for carrying MPEG-2 Transport Stream and M2TS source
packets over MOQT. It defines catalog-extension fields for transport-stream
track description and specifies subscriber behavior for joining, switching,
and validating packetized streams.

--- middle

# Introduction {#introduction}

MPEG-2 Transport Stream MOQT Streaming Format (MSFTS) is an extension of the
MOQT Streaming Format (MSF) {{MSF}} that delivers MPEG-2 Transport Stream (TS)
{{ISO138181}} content over MOQT {{MOQTransport}}. MSFTS retains the scope,
capabilities, and features of MSF, including the catalog format, the timeline,
and alternate rendition switching. A track described by the MSFTS catalog
fields carries either 188-octet TS packets or 192-octet M2TS source packets,
and the publisher maps consecutive source packets into MOQT Objects. MSFTS is
targeted at publishers that already produce packetized transport streams,
including contribution feeds, broadcast distribution workflows, and systems
that segment transport streams for HTTP-based delivery.

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
: A 192-octet packet consisting of a four-octet source-packet timestamp
  followed by a 188-octet TS packet.

Source packet:
: Either a TS packet or an M2TS source packet. The catalog signals which of
  the two a track carries.

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
{{MOQTransport}} without changing the transport stream itself.
Interoperability implies that:

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

An mpeg2ts-packaged MOQT Track carries a single ordered packet stream. A track
using this packaging MUST set the MSF catalog `packaging` field to "mpeg2ts".

## Object Payload Format {#object-payload-format}

The payload of each MOQT Object is a sequence of whole source packets:

~~~ ascii-art
+===============+===============+=====+===============+
| source packet | source packet | ... | source packet |
+===============+===============+=====+===============+
~~~
{: title="Object payload of an mpeg2ts track"}

Every source packet on a track has the same size, either 188 or 192 octets, as
`mpeg2tsPacketSize` ({{mpeg2ts-packet-size}}) declares. An Object payload MUST
contain only whole source packets, so its length is always a multiple of
`mpeg2tsPacketSize`. A subscriber MUST reject an Object that breaks this rule.

A subscriber reconstructs the packet stream by concatenating the source
packets from received Objects in ascending Group ID and Object ID order. A
subscriber that skips or fails to receive an Object MUST consider the
reconstructed packet stream discontinuous at that point until it reaches a
subsequent random access point.

## Object Boundaries {#object-boundaries}

Object boundaries are packaging boundaries and do not change MPEG-2 Transport
Stream semantics. Continuity counters, adaptation fields, PCR, Presentation
Time Stamp (PTS), Decoding Time Stamp (DTS), PSI, and other transport-stream
syntax remain inside the source packets.

MPEG-2 Transport Stream semantics cover a delivery schedule as well as syntax.
The PCR values in a stream state when each transport-stream byte is meant to
reach a decoder, and {{ISO138181}}, Section 2.4.2 expresses the buffer
constraints of the Transport Stream System Target Decoder against that
schedule. {{ISO138189}} gives the tolerance within which a delivered stream
matches the schedule, and {{TR101290}} defines the limits that a DVB
deployment must meet. Object boundaries do not alter the schedule that a
stream describes, and {{pcr-timing}} covers how MOQT delivery relates to it.

When `mpeg2tsModified` ({{mpeg2ts-modified}}) is false, a publisher MUST NOT
modify the continuity counter of any source packet and MUST NOT remap PIDs.
{{carriage-modes}} defines the modifications a publisher may make when
`mpeg2tsModified` is true.

## Group Boundaries {#group-boundaries}

For live single-program tracks, a publisher SHOULD start a new MOQT Group at
each point where the Group content is independently decodable without
reference to prior Groups. A publisher SHOULD place a random access point at
the first Object of each Group, and the Group then includes the PAT and PMT
packets required for program demultiplexing.

For a track carrying a whole multiplex, Group boundary placement depends on
whether the publisher can identify random access points across the multiplex.
A publisher that can identify them MAY align Group boundaries to those points
and set `mpeg2tsRandomAccess` to true. A publisher that cannot SHOULD start a
new Group after a fixed number of Objects.

When `mpeg2tsRandomAccess` ({{mpeg2ts-random-access}}) is true, the first
Object in every Group MUST provide a valid random access starting point for
that Group.

## Source Handling and Carriage Modes {#carriage-modes}

The required `mpeg2tsModified` field ({{mpeg2ts-modified}}) selects between
two carriage modes. When `mpeg2tsModified` is false, the publisher forwards
the source packets without modification ({{unmodified-carriage}}). When
`mpeg2tsModified` is true, the publisher has changed the source stream
({{modified-carriage}}).

### Unmodified Carriage {#unmodified-carriage}

When `mpeg2tsModified` is false, the publisher forwards the source packets
without modification: no program selection, no packet identifier remap, no PAT
or PMT rewrite, and no insertion or removal of null packets. A subscriber can
reconstruct the source stream byte-for-byte.

When the source is a single-program transport stream, `mpeg2tsMpts`
({{mpeg2ts-mpts}}) is false. The catalog does not need to describe the
program, because the PAT and PMT reach the subscriber unaltered within one PSI
repetition cycle.

When the source is a multi-program transport stream, `mpeg2tsMpts` is true and
the publisher emits all source packets as received. Because the publisher
selects no program, `mpeg2tsProgramNumber` and `mpeg2tsPcrPid` MUST be absent.

### Modified Carriage {#modified-carriage}

When `mpeg2tsModified` is true, the publisher has changed the source stream,
for example by selecting a program, filtering packets, rewriting the PAT or
PMT, or adding or removing null packets. A publisher that makes any of these
changes MUST set `mpeg2tsModified` to true. The optional `mpeg2tsEsPid` field
({{mpeg2ts-es-pid}}) distinguishes the two forms of modified carriage: it is
absent for per-program carriage and present for carriage of a single
elementary stream (ES-level carriage).

#### Per-Program Carriage {#per-program-carriage}

A publisher deriving a per-program track SHOULD drop every source packet
except:

* PAT packets (PID 0x0000), rewritten to list only the program present in
  this track.
* PMT packets for the selected program, on the PID that the rewritten PAT
  lists.
* Packets on any PID that the selected program's PMT lists, including the PCR
  PID, the PIDs of all elementary streams, and the PIDs that any
  CA_descriptor references.
* Packets carrying the service information (SI) tables that the publisher
  retains, if any.
* Conditional access packets, including the Conditional Access Table on PID
  0x0001, which no PMT lists.
* Null packets (PID 0x1FFF), which the publisher MAY drop or retain.

A publisher that rewrites the PAT and the PMT SHOULD emit them at least as
often as the source stream did.

A publisher filtering a scrambled transport stream MUST retain the conditional
access packets required for descrambling. Conditional access integration is
application-specific and outside the scope of this document. A CAT carried
from a multi-program source references the entitlement management streams of
every program in the multiplex, so a publisher SHOULD rewrite it to leave only
the entries for the carried program.

The `mpeg2tsProgramNumber` field ({{mpeg2ts-program-number}}) SHOULD be
present on per-program tracks to identify the program carried. When multiple
per-program tracks are derived from the same MPTS source, the publisher SHOULD
use the MSF `altGroup` field if the programs are alternate renditions of the
same content, and SHOULD publish programs that are independent services as
separate tracks.

Removing null packets changes the inter-packet byte spacing that
constant-bit-rate receivers use to recover the mux clock. A subscriber wishing
to reconstruct a constant-bit-rate output stream cannot derive the original
rate from the stream alone, so a publisher declares it with `mpeg2tsMuxRate`
({{mpeg2ts-mux-rate}}).

A publisher that retains SI tables SHOULD declare their PIDs using
`mpeg2tsSiPids` ({{mpeg2ts-si-pids}}), so that a subscriber can tell which
tables are present without inspecting the packet stream. The declaration is
needed because no PMT lists the SI PIDs, so the packet filter defined at the
start of this section drops these tables unless the publisher retains them
deliberately. A track without them has no service identity, no event schedule,
and no broadcast time, which a publisher targeting broadcast or IRD reception
SHOULD preserve.

Digital Video Broadcasting (DVB) and the Advanced Television Systems Committee
(ATSC) define different SI tables and place them on different PIDs. {{DVBSI}}
specifies the DVB tables and {{ATSCPSIP}} specifies the ATSC Program and
System Information Protocol. A publisher SHOULD retain the tables that the
target standard requires.

SI tables that describe individual services carry entries for every program in
a multiplex, so a publisher deriving a per-program track SHOULD rewrite them
to leave only the entries for the carried program.

#### ES-Level Carriage {#es-level-carriage}

When `mpeg2tsEsPid` ({{mpeg2ts-es-pid}}) is present, the track carries a
single elementary stream or signaling table. The track payload contains only
TS packets for the PID identified by `mpeg2tsEsPid`; PAT, PMT, and null
packets are not included. Publishers SHOULD use the MSF `initDataList` field
to carry the PAT and PMT of the originating program so that subscribers can
identify the program structure before processing elementary-stream packets.

When `mpeg2tsPcrPid` equals `mpeg2tsEsPid`, the track carries the PCR and
provides the timing reference for the program. When `mpeg2tsPcrPid` identifies
a different PID, another track carries the PCR, and a subscriber that needs
PCR timing MUST subscribe to that track. {{pcr-timing}} applies to the track
that carries the PCR.

A publisher producing multiple ES-level tracks for the same program SHOULD
align Group boundaries across those tracks so that matching Group numbers
correspond to the same presentation position. Elementary streams have
different frame durations, so exact alignment is not always possible.

A subscriber that combines multiple ES-level tracks and wants to output a
valid MPEG-2 Transport Stream SHOULD build a PAT listing the carried program
and a PMT listing the PIDs of the subscribed tracks. It SHOULD then take PCR
from the track whose `mpeg2tsEsPid` equals the `mpeg2tsPcrPid` that those
tracks declare.

A constructed PAT and PMT reach a receiver only if the subscriber repeats
them. The subscriber SHOULD repeat them at the interval that the standard
governing the output requires, for example {{TR101290}} for a DVB deployment.

An ES-level track carries no PAT, PMT, or CAT, so it carries no conditional
access association between a scrambled elementary stream and the streams that
key it. A publisher also cannot identify random access points in a payload it
cannot decrypt, so it cannot set `mpeg2tsRandomAccess` to true. A publisher
carrying a scrambled source SHOULD use unmodified or per-program carriage.

## PCR and Timing {#pcr-timing}

The PCR is carried inside adaptation fields of transport-stream packets as
defined by {{ISO138181}}. MOQT Object and Group boundaries are packaging
boundaries and do not alter PCR continuity within a track.

A publisher MUST NOT introduce a PCR discontinuity within a single MOQT Group.
A publisher that introduces a PCR discontinuity between consecutive MOQT
Groups MUST signal it by setting the discontinuity_indicator bit
({{ISO138181}}, Section 2.4.3.5) in the adaptation field of the first TS
packet carrying PCR in the new Group. The PCR base field wraps around during
long-running streams, and a wrap is not a discontinuity: a publisher MUST NOT
signal one when the PCR base wraps.

A subscriber cannot recover the source mux clock from the rate at which
packets arrive. MOQT delivers whole Objects, and a relay can serve them from
its cache as fast as the link allows, so arrival timing carries no information
about the source. A deployment that feeds equipment relying on arrival rate
needs a gateway that paces the reconstructed packet stream. `mpeg2tsMuxRate`
({{mpeg2ts-mux-rate}}) gives that gateway a target rate. A target rate does
not reproduce the source schedule, because the source byte clock runs
independently of the gateway clock. Reproducing the schedule requires
per-packet timing that this document does not define. Conformance to the
delivery schedule is therefore a property of the point where a
transport-stream output is produced, and not of the carriage between publisher
and subscriber.

## Splice Signaling {#splice-signaling}

An mpeg2ts track carries SCTE-35 {{SCTE35}} splice information in band, as
splice_info_section() messages on the PID that `mpeg2tsScte35Pid`
({{mpeg2ts-scte35-pid}}) declares. This document does not specify SCTE-35
processing.

A publisher MAY also publish the same splice events out of band, on an MSF
Event Timeline track. {{SCTE35Timeline}} defines the event type identifiers
and the payload format for that track. A subscriber can then read splice
events without parsing the packet stream.

# Catalog {#catalog}

The MSF catalog {{MSF}} describes an mpeg2ts track. This document extends that
catalog by defining the `mpeg2ts` value for the inherited `packaging` field
and additional fields for track objects that use that value. The catalog track
name, root catalog fields, common track fields, delta update rules, variable
substitution rules, and authorization signaling are inherited unchanged from
MSF unless this document explicitly states otherwise. A parser MUST ignore
fields it does not understand.

## Track Object Fields {#track-fields}

{{track-fields-table}} lists the mpeg2ts-specific fields defined within a
track object.

| Field                         | Name                    | Definition |
|:==============================|:========================|:===========|
| Packet size                   | mpeg2tsPacketSize         | {{mpeg2ts-packet-size}} |
| Stream modified               | mpeg2tsModified           | {{mpeg2ts-modified}} |
| ES PID                        | mpeg2tsEsPid              | {{mpeg2ts-es-pid}} |
| MPTS                          | mpeg2tsMpts               | {{mpeg2ts-mpts}} |
| Program number                | mpeg2tsProgramNumber      | {{mpeg2ts-program-number}} |
| PCR PID                       | mpeg2tsPcrPid             | {{mpeg2ts-pcr-pid}} |
| Mux rate                      | mpeg2tsMuxRate            | {{mpeg2ts-mux-rate}} |
| SI PIDs                       | mpeg2tsSiPids             | {{mpeg2ts-si-pids}} |
| Random access                 | mpeg2tsRandomAccess       | {{mpeg2ts-random-access}} |
| Timestamp mode                | mpeg2tsTimestampMode      | {{mpeg2ts-timestamp-mode}} |
| SCTE-35 PID                   | mpeg2tsScte35Pid          | {{mpeg2ts-scte35-pid}} |
{: #track-fields-table title="Track object fields defined by this document"}

Use of the MSF `initRef` and `initDataList` fields by mpeg2ts tracks is
described in {{init-data}}.

## Packet Size {#mpeg2ts-packet-size}

Required: Yes JSON Type: Number Location: Track Object

The source-packet size in octets. The value MUST be either 188 or 192. A value
of 188 identifies ordinary MPEG-2 TS packets. A value of 192 identifies M2TS
source packets with a four-octet timestamp prefix followed by a 188-octet TS
packet.

## Stream Modified {#mpeg2ts-modified}

Required: Yes JSON Type: Boolean Location: Track Object

Whether the publisher has changed the source packet stream. When false, the
published stream is a byte-for-byte copy of the source. {{carriage-modes}}
defines what a publisher may change when this field is true.

## ES PID {#mpeg2ts-es-pid}

Required: Optional JSON Type: Number Location: Track Object

The PID of the single elementary stream or signaling table carried by this
track. When present, the track carries only TS packets for that PID, and
carries neither PAT, PMT, nor null packets. This field selects ES-level
carriage, which {{es-level-carriage}} defines.

This field MUST be absent when `mpeg2tsMpts` is true. When it is present,
`mpeg2tsModified` MUST be true, and `mpeg2tsSiPids` and `mpeg2tsScte35Pid`
MUST be absent.

The MSF `role` field is a useful companion to `mpeg2tsEsPid`, because a PID
alone does not say what the track carries. For tracks carrying DVB or ATSC SI
tables, publishers SHOULD set `role` to one of the following values: `"nit"`
for the Network Information Table (PID 0x0010), `"sdt"` for the Service
Description Table and Bouquet Association Table (PID 0x0011), `"eit"` for the
Event Information Table (PID 0x0012), and `"tdt"` for the Time and Date Table
and Time Offset Table (PID 0x0014). For tracks carrying SCTE-35 splice
information, publishers SHOULD set `role` to `"scte35"`. For media elementary
streams, publishers SHOULD set `role` to the MSF-defined value for the stream
type, for example `"video"` or `"audio"`.

## MPTS {#mpeg2ts-mpts}

Required: Optional JSON Type: Boolean Location: Track Object

When true, this track carries a whole multi-program transport stream, with no
program selected and no PID filtered. `mpeg2tsModified` MUST be false when
this field is true, because a whole multiplex reaches the subscriber exactly
as the publisher received it. {{unmodified-carriage}} defines the carriage.

The following fields MUST be absent when this field is true:
`mpeg2tsProgramNumber`, `mpeg2tsPcrPid`, `mpeg2tsEsPid`, `mpeg2tsSiPids`,
`mpeg2tsScte35Pid`, and `mpeg2tsMuxRate`.

The catalog does not describe the program structure of an MPTS track. A
subscriber reads it from the PAT and the PMTs in the packet stream, which the
publisher forwards untouched.

## Program Number {#mpeg2ts-program-number}

Required: Optional JSON Type: Number Location: Track Object

The MPEG-2 Transport Stream program number carried by this track. When
present, the track SHOULD carry packets from only that program. This field
identifies the selected program in per-program carriage
({{per-program-carriage}}) and the originating program in ES-level carriage
({{es-level-carriage}}). It MUST be absent when `mpeg2tsMpts` is true.

## PCR PID {#mpeg2ts-pcr-pid}

Required: Optional JSON Type: Number Location: Track Object

The PID carrying the PCR of the program that this track carries. This field is
advisory and does not replace the PCR signaling in the transport stream. It
MUST be absent when `mpeg2tsMpts` is true.

In ES-level carriage, the PCR may travel on a different track. A subscriber
that needs PCR timing for an ES-level track MUST also subscribe to the track
whose `mpeg2tsEsPid` equals the `mpeg2tsPcrPid` declared by that ES-level
track.

## Mux Rate {#mpeg2ts-mux-rate}

Required: Optional JSON Type: Number Location: Track Object

The nominal mux rate of the source transport stream in bits per second,
counted over 188-octet TS packets. The count excludes the four-octet timestamp
prefix of an M2TS source packet.

A publisher SHOULD declare this field when it removes null packets, and on
ES-level tracks, which carry no null packets at all. Where a program is
published as several ES-level tracks, every track of that program SHOULD
declare the same value, which describes the reconstructed program and not any
single track.

The declared rate is a stuffing target rather than a timing source. An
implementation producing a transport-stream output recovers its clock from the
PCR values in the stream, and uses this rate to decide how much null stuffing
to insert.

This field MUST be absent when `mpeg2tsMpts` is true.

## SI PIDs {#mpeg2ts-si-pids}

Required: Optional JSON Type: Array Location: Track Object

An array of the PIDs carrying the SI tables that a per-program track retains.
DVB and ATSC place each table on its own PID, so a publisher retaining more
than one table lists one PID per table. The array does not repeat the PIDs
that the PMT lists.

A publisher SHOULD include this field when it retains SI tables
({{per-program-carriage}}). The field is advisory: a subscriber MAY use it to
learn which tables are present without parsing the packet stream.

This field MUST be absent when `mpeg2tsEsPid` is present or `mpeg2tsMpts` is
true. An ES-level track carries one table, which its `mpeg2tsEsPid` and MSF
`role` field identify ({{mpeg2ts-es-pid}}).

## Random Access {#mpeg2ts-random-access}

Required: Optional JSON Type: Boolean Location: Track Object

When true, every MOQT Group starts with a random access point, as defined in
{{group-boundaries}}. When absent or false, this document makes no guarantee
about where in a Group decoding can begin.

## Timestamp Mode {#mpeg2ts-timestamp-mode}

Required: Optional JSON Type: String Location: Track Object

For 192-octet source packets, this field identifies the interpretation of the
four-octet source-packet timestamp. The value "arrival-time" indicates an
arrival-time or emission-time stamp associated with the following TS packet.
The value "opaque" indicates that the timestamp prefix is carried without
specified semantics. This field MUST NOT be present when `mpeg2tsPacketSize`
is 188.

## SCTE-35 PID {#mpeg2ts-scte35-pid}

Required: Optional JSON Type: Number Location: Track Object

The PID carrying SCTE-35 splice_info_section() messages for this track. This
field is advisory; SCTE-35 messages are also discoverable via the PMT
conditional access or registration descriptor. When present, a subscriber MAY
use this value to locate splice events without parsing the PMT. Publishers
SHOULD include this field when the track carries SCTE-35 splice signaling.
This field MUST be absent when `mpeg2tsEsPid` is present or `mpeg2tsMpts` is
true; when SCTE-35 is published as an ES-level track, the track's
`mpeg2tsEsPid` and `role` fields identify it.

## Use of MSF Initialization Data {#init-data}

A subscriber obtains the PAT and the PMT in one of three ways: it reads them
from `initDataList` when the track declares `initRef`, it accumulates packets
from the joining point until the publisher repeats the PSI, or it fetches a
past Object that carries them.

MSF defines the `initRef` track field and the root `initDataList` field. An
mpeg2ts track MAY use those fields to carry initialization data. The track
sets `initRef` to the `id` of an `initDataList` entry whose `type` MUST be
"inline". The Base64
{{BASE64}} decoded value of the entry's `data` field MUST be a sequence of
whole source packets using the packet size declared by `mpeg2tsPacketSize`.

Publishers SHOULD include current PAT and PMT packets in the referenced
initialization data when those tables are not guaranteed to be available at
the first Object of each Group. When PSI changes within a live track, the
publisher SHOULD publish an updated initialization data entry in a new
independent catalog before publishing Objects that rely on the changed PSI.
Subscribers MUST NOT assume that referenced initialization data remains valid
after the MPEG-2 PSI `version_number` changes; updated PSI in media Objects
takes precedence.

A publisher using unmodified carriage ({{unmodified-carriage}}) typically
omits `initRef`, because it does not inspect the source stream and the PSI
reaches the subscriber unchanged.

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
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPcrPid": 257,
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
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPcrPid": 257,
      "mpeg2tsMuxRate": 6500000,
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
      "mpeg2tsProgramNumber": 2,
      "mpeg2tsPcrPid": 513,
      "mpeg2tsMuxRate": 4500000,
      "mpeg2tsRandomAccess": true
    }
  ]
}
~~~

## Transparent MPTS Carriage {#example-mpts-transparent}

This example shows a catalog for a publisher that carries a complete
multi-program transport stream without program selection, so no per-program
catalog fields are present.

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
      "mpeg2tsMpts": true
    }
  ]
}
~~~

## Alternate Renditions - Two Bitrate Tracks {#example-abr}

This example shows a catalog for a live channel published at two bitrates as
alternate renditions. Both tracks are in the same `altGroup`; video tracks
MUST align Group boundaries at identical presentation positions. The tracks
use different PID assignments: a subscriber switching between them MUST
re-parse PAT and PMT on the new track before routing packets to a decoder.

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
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPcrPid": 257,
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
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPcrPid": 513,
      "mpeg2tsRandomAccess": true
    }
  ]
}
~~~

## ES-Level Tracks - Per-Elementary-Stream Publishing {#example-es-level}

This example shows a live program published as separate ES-level tracks: one
video track carrying the PCR, two audio tracks for different languages
(English and Spanish), and one Event Information Table track. The video and
audio tracks MUST have synchronized Group boundaries. A subscriber combines
the video track and the audio track of its choice by constructing a PAT and
PMT listing the subscribed PIDs and sourcing PCR from the video track (PID
257).

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
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPcrPid": 257,
      "mpeg2tsMuxRate": 6000000,
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
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPcrPid": 257,
      "mpeg2tsMuxRate": 6000000,
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
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsPcrPid": 257,
      "mpeg2tsMuxRate": 6000000,
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
      "mpeg2tsProgramNumber": 1,
      "mpeg2tsMuxRate": 6000000,
      "mpeg2tsEsPid": 18
    }
  ]
}
~~~

# Switching and Alternate Renditions {#switching}

A publisher advertises multiple mpeg2ts tracks as alternatives using the MSF
`altGroup` field. Video tracks in the same alternate group MUST place Group
boundaries at identical presentation positions, and other tracks SHOULD align
their Group boundaries to the same positions where possible. A track with
`mpeg2tsMpts` set to true MUST NOT appear in an `altGroup`.

A subscriber switches between alternate mpeg2ts tracks either at a Group
boundary or at a transport-stream random access point that it can
independently decode. This document does not require continuity counter values
or PID assignments to match across alternate tracks, so a subscriber MUST
treat a switch as a packet-stream discontinuity.

After a switch, a receiver MUST re-initialize its system time clock (STC)
recovery from the first PCR of the new track. It MUST also re-parse the PAT
and PMT of the new track before routing elementary-stream packets to a
decoder.

# Content Protection {#content-protection}

Unmodified carriage preserves any scrambling and conditional access
information present in the MPEG-2 Transport Stream. Per-program carriage
preserves it when the publisher retains the conditional access packets
({{per-program-carriage}}). ES-level carriage does not preserve it.
Transport-stream scrambling is opaque to MOQT relays and to this
specification.

A publisher MAY apply object-level encryption using a mechanism such as Secure
Objects {{SecureObjects}}, when the catalog signals it. A subscriber then
validates source packets after decrypting the Object payload.

# Security Considerations {#security-considerations}

The security considerations of MOQT {{MOQTransport}}, MSF {{MSF}}, MPEG-2
Transport Stream {{ISO138181}}, and any object encryption scheme apply.

Receivers need to treat transport-stream syntax as untrusted input. Invalid
packet sizes, invalid sync bytes, malformed PSI, inconsistent continuity
counters, excessive table repetition, and timestamp discontinuities can cause
decoder failures or resource exhaustion if not bounded by implementation
policy.

Catalog metadata is also untrusted input. Subscribers MUST validate packet
sizes, payload lengths, Base64 values, PIDs, program numbers, and object
ordering before using the values to allocate memory or configure decoders.

Object-level encryption protects MOQT Object payloads but does not hide MOQT
namespace, track name, Group ID, Object ID, object size, or delivery timing
from authorized relays. Applications that require confidentiality for media
payloads SHOULD use an object encryption scheme in addition to transport
security.

# IANA Considerations {#iana-considerations}

This document requests that, once MSF establishes an IANA registry for
packaging values, IANA register the value "mpeg2ts" with this document as the
reference.

--- back

# Acknowledgments

This document follows the repository and draft structure used by the MOQT
Streaming Format work.
