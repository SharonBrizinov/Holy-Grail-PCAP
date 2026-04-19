# The Holy Grail PCAP by Sharon Brizinov

<p align="center">
  <img src="docs/logo.png" alt="The Holy Grail PCAP by Sharon Brizinov" width="300">
</p>

Some people collect Pokemon cards. I collect packets.

This project is the result of months of work to create the **"Holy Grail" PCAP** - a massive pcap file containing packets that trigger the code-flows of virtually all **1,600+ Wireshark dissectors** across all encapsulation types. When parsed by Wireshark tshark, this pcap exercises the code of almost every protocol dissector in the project.

**This is the most minimal set of packets that generates the greatest possible code coverage across all Wireshark dissectors.** Every packet earns its place - throughout the project I continuously measured dissector code coverage using [wirecov](https://github.com/SharonBrizinov/wirecov), removing redundant packets and keeping only those that contribute unique code paths.

This pcap goes far beyond just Ethernet traffic. It covers **186 different link-layer types** including Bluetooth, USB, Wi-Fi (802.11), IEEE 802.15.4, CAN bus, DOCSIS, Frame Relay, ATM, PPP, FDDI, Token Ring, Linux Cooked Capture, and many more. Each link-layer type (also known as an encapsulation type) tells Wireshark how to interpret the raw bytes at the lowest level of a packet - before any IP, TCP, or application-layer parsing begins. Wireshark identifies these using internal WTAP numbers, which map to DLT (Data Link Type) values from libpcap. See [Understanding WTAP, ENCAP, and DLT](#understanding-wtap-encap-and-dlt) for more details.

## Stats

| | |
|---|---|
| **Mega Merge Size** | 824 MB |
| **Unique Protocol Stacks** | ~98,000 |
| **Total Packets** | 3,163,318 |
| **Encap Types** | 186 |

## Download

| File | Description |
|---|---|
| [`ALL_ENCAPS_HOLY_GRAIL_PACP.pcapng`](pcaps/all_merged/ALL_ENCAPS_HOLY_GRAIL_PACP.pcapng) | The mega merge - 2,773,127 packets across 115 known encap types in a single pcapng file |
| [`pcaps/encap_types/`](pcaps/encap_types/) | Individual per-encap pcapng files for all 186 encap types (including 67 unknown DLTs and edge cases not in the mega merge) |

## How to Clone

You can download pcaps manually, for example - [`ALL_ENCAPS_HOLY_GRAIL_PACP.pcapng`](pcaps/all_merged/ALL_ENCAPS_HOLY_GRAIL_PACP.pcapng). However, if you want to clone this project and download all pcaps, you'll need to use `git-lfs`. The pcap files in this repository are stored using **Git LFS (Large File Storage)** because they are too large for regular Git. Git LFS is a Git extension that replaces large files with lightweight pointers in the repository while storing the actual file contents on a separate server. Without it, cloning would only give you small pointer files instead of the real pcapngs.

```bash
# 1. Install Git LFS (one-time setup)
# macOS - brew install git-lfs
# Ubuntu Debian - sudo apt install git-lfs
# Windows - Download from https://git-lfs.com

# 2. Initialize Git LFS (one-time setup)
git lfs install

# 3. Clone the repository (LFS files are downloaded automatically)
git clone https://github.com/SharonBrizinov/Holy-Grail-PCAP.git
cd Holy-Grail-PCAP

# 4. If you already cloned without LFS, pull the actual files
git lfs pull
```

To verify the download worked, check that the mega merge file is ~824 MB and not a tiny pointer file:

```bash
ls -lh pcaps/all_merged/ALL_ENCAPS_HOLY_GRAIL_PACP.pcapng
# Should show ~824M, not 130 bytes
```

## Benchmark

I ran this pcap through `tshark` (~v4.7.0, compiled with ASAN) using various flag combinations on an Apple M1 Max (32 GB RAM, 10 CPU cores). The results below make it clear that `-2` (two-pass analysis) is the dominant cost driver.

| id | time (hours, minutes) | flags used |
|----|----------------------|------------|
| 1 | 0h 1m | `-n` |
| 2 | 0h 1m | `-n -o ip.defragment:FALSE` |
| 3 | 0h 14m | `-n -V` |
| 4 | 0h 16m | `-n -o tcp.desegment_tcp_streams:FALSE -o ip.defragment:FALSE` |
| 5 | 5h 6m | `-n -2 -z expert` |
| 6 | 7h 15m | `-n -2 -o ip.defragment:FALSE -V -z expert` |
| 7 | 11h 58m | `-n -2 -o tcp.desegment_tcp_streams:FALSE -o ip.defragment:FALSE -V -z expert` |


## How to Run

Use `tshark` (Wireshark's command-line tool) to process the Holy Grail PCAP. Here are the most useful flags:

| Flag | Description | When to Use |
|------|-------------|-------------|
| `-r <file>` | Read packets from a pcapng file | Always - this is how you specify the input file |
| `-n` | Disable network name resolution (no DNS lookups) | Always recommended - avoids slow DNS queries and hangs on large pcaps |
| `-2` | Perform two-pass analysis | When you need accurate reassembly, conversation tracking, or response-time statistics. First pass builds state, second pass uses it |
| `-V` | Show full packet dissection tree (verbose output) | When you want to see every protocol field decoded in detail. Warning: produces massive output on large pcaps |
| `-z expert` | Show expert info summary (errors, warnings, notes) | Great for quickly finding dissector problems - shows malformed packets, protocol violations, and anomalies |
| `-o tcp.desegment_tcp_streams:FALSE` | Disable TCP reassembly | When stress-testing dissectors on individual packets without waiting for full TCP streams. Also useful when you care about per-packet parsing rather than stream-level analysis |
| `-o ip.defragment:FALSE` | Disable IP fragment reassembly | When testing dissector handling of individual IP fragments rather than reassembled datagrams |

### Example Commands

```bash
# Stress-test dissectors without reassembly (maximizes per-packet code coverage)
tshark -n -r pcaps/all_merged/ALL_ENCAPS_HOLY_GRAIL_PACP.pcapng \
  -o tcp.desegment_tcp_streams:FALSE \
  -o ip.defragment:FALSE

# Stress-test dissectors without defragment, verbose mode
tshark -n -V -r pcaps/all_merged/ALL_ENCAPS_HOLY_GRAIL_PACP.pcapng \
  -o ip.defragment:FALSE

# Full two-pass analysis with expert info (slower but more accurate)
tshark -n -2 -r pcaps/all_merged/ALL_ENCAPS_HOLY_GRAIL_PACP.pcapng -z expert

# Full verbose dissection of a specific encap type (e.g., Bluetooth)
tshark -n -V -r pcaps/encap_types/WTAP-041_DLT-187_BLUETOOTH_H4/*.pcapng

# Parse first 1000 packets with full detail to preview
tshark -n -V -c 1000 -r pcaps/all_merged/ALL_ENCAPS_HOLY_GRAIL_PACP.pcapng

# Crash-test a dev build of tshark - parse everything with all features enabled
tshark -n -2 -r pcaps/all_merged/ALL_ENCAPS_HOLY_GRAIL_PACP.pcapng \
  -o tcp.desegment_tcp_streams:FALSE \
  -o ip.defragment:FALSE \
  -V -z expert > /dev/null
```

## How It Was Made

Building this pcap was a multi-stage process:

1. **Collection** - Gathered hundreds of pcap pcapng files from every source I could find, including the [Wireshark Sample Captures](https://wiki.wireshark.org/SampleCaptures) wiki, the [Wireshark Automated Captures](https://www.wireshark.org/download/automated/captures/) repository, and many other sources across the internet.

2. **Minimization & Coverage Testing** - Minimized the collected captures and measured code coverage against Wireshark's dissector codebase using [wirecov](https://github.com/SharonBrizinov/wirecov) to identify gaps.

3. **Fuzzing** - Fuzzed Wireshark tshark extensively using [wirefuzz](https://github.com/SharonBrizinov/wirefuzz) to generate packets that hit uncovered code paths, then measured coverage again.

4. **Manual Packet Generation** - Manually crafted packets for specific dissectors and code paths that fuzzing couldn't reach, to push coverage even higher.

5. **Iterate** - Repeated the minimization, coverage testing, and fuzzing cycle multiple times until coverage plateaued.

## Custom Tools

I developed two open-source tools specifically to support this project:

### [wirecov](https://github.com/SharonBrizinov/wirecov)

A code coverage tool for Wireshark with **deep focus on dissector code**. It compiles Wireshark with coverage instrumentation, runs pcap files through tshark, and generates detailed coverage reports showing exactly which dissector code paths were hit and which were missed. It also supports testing a **matrix of Wireshark versions** (master, v4.4, v4.6, etc.) so you can compare coverage across releases. This was essential for identifying gaps and measuring progress throughout the project.

### [wirefuzz](https://github.com/SharonBrizinov/wirefuzz)

A fuzzing framework tailored for Wireshark. It uses coverage-guided fuzzing to generate mutated packets that explore new code paths in Wireshark's dissectors. Unlike generic fuzzers, wirefuzz understands pcap file structure and can target specific encapsulation types and protocol layers, making it far more effective at reaching deep dissector code.

## Use Cases

- **Bug Hunting and Vulnerability Research** - Because this pcap exercises nearly every dissector code path, it is extremely effective at surfacing bugs. I've already discovered **dozens of zero-day vulnerabilities** across multiple Wireshark versions simply by running this pcap through different builds and observing crashes, assertion failures, and memory errors. If a dissector has a latent bug, chances are this pcap will hit it. Take it further with [wirefuzz](https://github.com/SharonBrizinov/wirefuzz) - unlike generic fuzzers, wirefuzz can target **any encapsulation type** and accepts pcap files as base input for mutations. This means you can feed it the per-encap pcapngs from this project and fuzz specific dissectors with high-quality seed inputs that already reach deep code paths, rather than starting from scratch.

- **Code Coverage Analysis** - Pair this pcap with [wirecov](https://github.com/SharonBrizinov/wirecov) to get a detailed picture of exactly which dissector code paths are being exercised and which are not. This is invaluable for Wireshark developers who want to understand how well their test suite covers the codebase, or for security researchers looking for untested (and therefore potentially vulnerable) code.

- **CI CD Regression Testing** - Integrate this pcap into your CI CD pipeline as a regression test. Run `tshark -r` against the Holy Grail PCAP on every commit or nightly build - if a code change introduces a crash, memory leak, or parsing regression in any of the 1,600+ dissectors, this pcap is likely to catch it. This is significantly more thorough than testing with a handful of protocol-specific captures.

- **Stress Testing and Benchmarking** - With over 3 million packets spanning 186 encapsulation types and ~98,000 unique protocol stacks, this pcap is an excellent stress test for Wireshark, tshark, or any tool that processes pcapng files. Use it to benchmark parsing performance, identify memory bottlenecks, or validate that your tool handles exotic and deeply nested protocol combinations gracefully.

- **Learning and Research** - The per-encap pcapng files in [`pcaps/encap_types/`](pcaps/encap_types/) are a curated library of packet captures organized by link-layer type. If you need sample traffic for a specific protocol - whether it's Juniper PPP, BACnet MS TP, GPRS LLC, or Z-Wave - you can find it here.

## pcap vs pcapng

There are two major capture file formats in the packet capture world:

- **pcap** - The classic format introduced by libpcap. It uses a global header that defines a single link-layer type for the entire file, meaning every packet in a pcap file must share the same encapsulation. It has no support for metadata beyond timestamps and packet lengths, and no way to store multiple interfaces, comments, name resolution records, or other auxiliary data.

- **pcapng (pcap Next Generation)** - The modern replacement for pcap. It supports multiple interfaces with different link-layer types in a single file through Interface Description Blocks (IDBs), per-packet encapsulation, file and packet comments, name resolution blocks, custom metadata, and a flexible block-based structure that can be extended without breaking compatibility.

**All capture files in this project use the pcapng format.** This is a deliberate choice - the Holy Grail PCAP spans 186 different encapsulation types, and the mega merge file alone contains 115 encap types across 290 interfaces. This is fundamentally impossible with the classic pcap format, which is limited to a single link-layer type per file. pcapng's support for multiple IDBs with per-packet encapsulation is what makes it possible to merge captures from dozens of different link-layer technologies (Ethernet, Wi-Fi, USB, Bluetooth, CAN bus, and many more) into a single unified file.

## Understanding WTAP, ENCAP, and DLT

When working with pcap files and Wireshark, you'll encounter three related concepts for identifying the link-layer type of captured packets:

- **WTAP (Wiretap)** - Wireshark's internal encapsulation type identifier. This is the ID Wireshark uses internally to determine which dissector should handle the raw bytes of each packet. Each WTAP number maps to a specific link-layer protocol handler within Wireshark (e.g., WTAP-001 = Ethernet, WTAP-025 = Linux Cooked Capture).

- **ENCAP (Encapsulation)** - A general term for the link-layer encapsulation type of a packet. In Wireshark's context, ENCAP and WTAP are often used interchangeably - they refer to the same internal mapping. The `encapsulation type` field in pcapng Interface Description Blocks specifies the WTAP ENCAP type.

- **DLT (Data Link Type)** - The link-layer header type as defined by libpcap tcpdump. DLT values are used in the classic pcap file format header and in pcapng files to indicate how to interpret the raw packet data. Wireshark maps each DLT value to its corresponding WTAP type. DLT values are standardized by [tcpdump.org](https://www.tcpdump.org/linktypes.html). Note that the same WTAP type can map to multiple DLT values (e.g., WTAP-007 RAW_IP maps to DLT-012, DLT-014, and DLT-101).

## Pcap Table

The Holy Grail PCAP contains **186 encapsulation types** organized into individual per-encap pcapng files. The mega merge file combines 115 known encap types into a single pcapng. The remaining files (67 unknown DLTs + 4 problem files) are available as individual per-encap pcapngs.

### Known Encap Types

| WTAP | DLT | Name | Packets | Pcap |
|------|-----|------|--------:|------|
| WTAP-001 | DLT-001 | Ethernet (part 1) | 220,243 | [pcapng](pcaps/encap_types/WTAP-001_DLT-001_ETHERNET/WTAP-001_DLT-001_ETHERNET.part-1.merged.pcapng) |
| WTAP-001 | DLT-001 | Ethernet (part 2) | 129,329 | [pcapng](pcaps/encap_types/WTAP-001_DLT-001_ETHERNET/WTAP-001_DLT-001_ETHERNET.part-2.merged.pcapng) |
| WTAP-002 | DLT-006 | Token Ring | 33,332 | [pcapng](pcaps/encap_types/WTAP-002_DLT-006_TOKEN_RING/) |
| WTAP-003 | DLT-008 | SLIP | 40,430 | [pcapng](pcaps/encap_types/WTAP-003_DLT-008_SLIP/) |
| WTAP-004 | DLT-050 | PPP | 43,936 | [pcapng](pcaps/encap_types/WTAP-004_DLT-050_PPP/) |
| WTAP-006 | DLT-010 | FDDI | 38,835 | [pcapng](pcaps/encap_types/WTAP-006_DLT-010_FDDI_WITH_BIT-SWAPPED_MAC_ADDRESSES/) |
| WTAP-007 | DLT-012 | Raw IP | 1,266 | [pcapng](pcaps/encap_types/WTAP-007_DLT-012_RAW_IP/) |
| WTAP-007 | DLT-014 | Raw IP | 6 | [pcapng](pcaps/encap_types/WTAP-007_DLT-014_RAW_IP/) |
| WTAP-007 | DLT-101 | Raw IP | 61,858 | [pcapng](pcaps/encap_types/WTAP-007_DLT-101_RAW_IP/) |
| WTAP-009 | DLT-129 | Linux ARCNET | 38,874 | [pcapng](pcaps/encap_types/WTAP-009_DLT-129_LINUX_ARCNET/) |
| WTAP-010 | DLT-100 | RFC 1483 ATM | 39,642 | [pcapng](pcaps/encap_types/WTAP-010_DLT-100_RFC_1483_ATM/) |
| WTAP-011 | DLT-016 | Linux ATM CLIP | 4 | [pcapng](pcaps/encap_types/WTAP-011_DLT-016_LINUX_ATM_CLIP/) |
| WTAP-011 | DLT-106 | Linux ATM CLIP | 41,450 | [pcapng](pcaps/encap_types/WTAP-011_DLT-106_LINUX_ATM_CLIP/) |
| WTAP-013 | DLT-123 | ATM PDUs | 1,066 | [pcapng](pcaps/encap_types/WTAP-013_DLT-123_ATM_PDUS/) |
| WTAP-015 | DLT-000 | Null Loopback | 40,116 | [pcapng](pcaps/encap_types/WTAP-015_DLT-000_NULLLOOPBACK/) |
| WTAP-018 | DLT-122 | RFC 2625 IP-over-Fibre Channel | 63,162 | [pcapng](pcaps/encap_types/WTAP-018_DLT-122_RFC_2625_IP-OVER-FIBRE_CHANNEL/) |
| WTAP-019 | DLT-204 | PPP with Directional Info | 74,919 | [pcapng](pcaps/encap_types/WTAP-019_DLT-204_PPP_WITH_DIRECTIONAL_INFO/) |
| WTAP-020 | DLT-105 | IEEE 802.11 Wireless LAN | 32,868 | [pcapng](pcaps/encap_types/WTAP-020_DLT-105_IEEE_80211_WIRELESS_LAN/) |
| WTAP-021 | DLT-119 | IEEE 802.11 + Prism II Header | 64,810 | [pcapng](pcaps/encap_types/WTAP-021_DLT-119_IEEE_80211_PLUS_PRISM_II_MONITOR_MODE_RADIO_HEADER/) |
| WTAP-023 | DLT-127 | IEEE 802.11 + Radiotap Header | 31,878 | [pcapng](pcaps/encap_types/WTAP-023_DLT-127_IEEE_80211_PLUS_RADIOTAP_RADIO_HEADER/) |
| WTAP-024 | DLT-163 | IEEE 802.11 + AVS Header | 23,686 | [pcapng](pcaps/encap_types/WTAP-024_DLT-163_IEEE_80211_PLUS_AVS_RADIO_HEADER/) |
| WTAP-025 | DLT-113 | Linux Cooked Capture v1 | 87,920 | [pcapng](pcaps/encap_types/WTAP-025_DLT-113_LINUX_COOKED-MODE_CAPTURE_V1/) |
| WTAP-026 | DLT-107 | Frame Relay | 44,972 | [pcapng](pcaps/encap_types/WTAP-026_DLT-107_FRAME_RELAY/) |
| WTAP-028 | DLT-104 | Cisco HDLC | 37,460 | [pcapng](pcaps/encap_types/WTAP-028_DLT-104_CISCO_HDLC/) |
| WTAP-028 | DLT-112 | Cisco HDLC | 4 | [pcapng](pcaps/encap_types/WTAP-028_DLT-112_CISCO_HDLC/) |
| WTAP-029 | DLT-118 | Cisco IOS Internal | 19,286 | [pcapng](pcaps/encap_types/WTAP-029_DLT-118_CISCO_IOS_INTERNAL/) |
| WTAP-030 | DLT-114 | LocalTalk | 3,310 | [pcapng](pcaps/encap_types/WTAP-030_DLT-114_LOCALTALK/) |
| WTAP-032 | DLT-121 | HiPath HDLC | 28,783 | [pcapng](pcaps/encap_types/WTAP-032_DLT-121_HIPATH_HDLC/) |
| WTAP-033 | DLT-143 | DOCSIS | 56,709 | [pcapng](pcaps/encap_types/WTAP-033_DLT-143_DATA_OVER_CABLE_SERVICE_INTERFACE_SPECIFICATION/) |
| WTAP-037 | DLT-128 | Tazmen Sniffer Protocol | 39,668 | [pcapng](pcaps/encap_types/WTAP-037_DLT-128_TAZMEN_SNIFFER_PROTOCOL/) |
| WTAP-038 | DLT-109 | OpenBSD enc4 | 38,672 | [pcapng](pcaps/encap_types/WTAP-038_DLT-109_OPENBSD_ENC4_ENCAPSULATING_INTERFACE/) |
| WTAP-039 | DLT-117 | OpenBSD pflog | 39,856 | [pcapng](pcaps/encap_types/WTAP-039_DLT-117_OPENBSD_PF_FIREWALL_LOGS/) |
| WTAP-041 | DLT-187 | Bluetooth H4 | 11,984 | [pcapng](pcaps/encap_types/WTAP-041_DLT-187_BLUETOOTH_H4/) |
| WTAP-042 | DLT-140 | SS7 MTP2 | 18,977 | [pcapng](pcaps/encap_types/WTAP-042_DLT-140_SS7_MTP2/) |
| WTAP-043 | DLT-141 | SS7 MTP3 | 18,877 | [pcapng](pcaps/encap_types/WTAP-043_DLT-141_SS7_MTP3/) |
| WTAP-044 | DLT-144 | IrDA | 4 | [pcapng](pcaps/encap_types/WTAP-044_DLT-144_IRDA/) |
| WTAP-045 | DLT-147 | USER 0 | 33,784 | [pcapng](pcaps/encap_types/WTAP-045_DLT-147_USER_0/) |
| WTAP-046 | DLT-148 | USER 1 | 35,904 | [pcapng](pcaps/encap_types/WTAP-046_DLT-148_USER_1/) |
| WTAP-047 | DLT-149 | USER 2 | 41,113 | [pcapng](pcaps/encap_types/WTAP-047_DLT-149_USER_2/) |
| WTAP-048 | DLT-150 | USER 3 | 41,706 | [pcapng](pcaps/encap_types/WTAP-048_DLT-150_USER_3/) |
| WTAP-049 | DLT-151 | USER 4 | 38,764 | [pcapng](pcaps/encap_types/WTAP-049_DLT-151_USER_4/) |
| WTAP-050 | DLT-152 | USER 5 | 5 | [pcapng](pcaps/encap_types/WTAP-050_DLT-152_USER_5/) |
| WTAP-051 | DLT-153 | USER 6 | 18 | [pcapng](pcaps/encap_types/WTAP-051_DLT-153_USER_6/) |
| WTAP-052 | DLT-154 | USER 7 | 5 | [pcapng](pcaps/encap_types/WTAP-052_DLT-154_USER_7/) |
| WTAP-053 | DLT-155 | USER 8 | 4 | [pcapng](pcaps/encap_types/WTAP-053_DLT-155_USER_8/) |
| WTAP-054 | DLT-156 | USER 9 | 945 | [pcapng](pcaps/encap_types/WTAP-054_DLT-156_USER_9/) |
| WTAP-055 | DLT-157 | USER 10 | 115 | [pcapng](pcaps/encap_types/WTAP-055_DLT-157_USER_10/) |
| WTAP-056 | DLT-158 | USER 11 | 5 | [pcapng](pcaps/encap_types/WTAP-056_DLT-158_USER_11/) |
| WTAP-057 | DLT-159 | USER 12 | 4 | [pcapng](pcaps/encap_types/WTAP-057_DLT-159_USER_12/) |
| WTAP-058 | DLT-160 | USER 13 | 6 | [pcapng](pcaps/encap_types/WTAP-058_DLT-160_USER_13/) |
| WTAP-059 | DLT-161 | USER 14 | 10,611 | [pcapng](pcaps/encap_types/WTAP-059_DLT-161_USER_14/) |
| WTAP-060 | DLT-162 | USER 15 | 39,516 | [pcapng](pcaps/encap_types/WTAP-060_DLT-162_USER_15/) |
| WTAP-061 | DLT-099 | Symantec Enterprise Firewall | 40,678 | [pcapng](pcaps/encap_types/WTAP-061_DLT-099_SYMANTEC_ENTERPRISE_FIREWALL/) |
| WTAP-063 | DLT-165 | BACnet MS TP | 11,921 | [pcapng](pcaps/encap_types/WTAP-063_DLT-165_BACNET_MSTP/) |
| WTAP-066 | DLT-169 | GPRS LLC | 26,450 | [pcapng](pcaps/encap_types/WTAP-066_DLT-169_GPRS_LLC/) |
| WTAP-067 | DLT-137 | Juniper ATM1 | 37,558 | [pcapng](pcaps/encap_types/WTAP-067_DLT-137_JUNIPER_ATM1/) |
| WTAP-068 | DLT-135 | Juniper ATM2 | 37,751 | [pcapng](pcaps/encap_types/WTAP-068_DLT-135_JUNIPER_ATM2/) |
| WTAP-069 | DLT-032 | Redback SmartEdge | 41,191 | [pcapng](pcaps/encap_types/WTAP-069_DLT-032_REDBACK_SMARTEDGE/) |
| WTAP-075 | DLT-139 | Encap 75 | 873 | [pcapng](pcaps/encap_types/WTAP-075_DLT-139_ENCAP_75/) |
| WTAP-076 | DLT-167 | Juniper PPPoE | 490 | [pcapng](pcaps/encap_types/WTAP-076_DLT-167_JUNIPER_PPPOE/) |
| WTAP-077 | DLT-172 | GCOM TIE1 | 34,990 | [pcapng](pcaps/encap_types/WTAP-077_DLT-172_GCOM_TIE1/) |
| WTAP-078 | DLT-173 | GCOM Serial | 144 | [pcapng](pcaps/encap_types/WTAP-078_DLT-173_GCOM_SERIAL/) |
| WTAP-083 | DLT-178 | Juniper Ethernet | 45,920 | [pcapng](pcaps/encap_types/WTAP-083_DLT-178_JUNIPER_ETHERNET/) |
| WTAP-084 | DLT-179 | Juniper PPP | 40,391 | [pcapng](pcaps/encap_types/WTAP-084_DLT-179_JUNIPER_PPP/) |
| WTAP-085 | DLT-180 | Juniper Frame Relay | 31,134 | [pcapng](pcaps/encap_types/WTAP-085_DLT-180_JUNIPER_FRAME-RELAY/) |
| WTAP-086 | DLT-181 | Juniper C-HDLC | 39,485 | [pcapng](pcaps/encap_types/WTAP-086_DLT-181_JUNIPER_C-HDLC/) |
| WTAP-087 | DLT-133 | Juniper GGSN | 9,081 | [pcapng](pcaps/encap_types/WTAP-087_DLT-133_JUNIPER_GGSN/) |
| WTAP-091 | DLT-183 | Juniper Voice PIC | 488 | [pcapng](pcaps/encap_types/WTAP-091_DLT-183_JUNIPER_VOICE_PIC/) |
| WTAP-092 | DLT-186 | USB (FreeBSD Header) | 671 | [pcapng](pcaps/encap_types/WTAP-092_DLT-186_USB_PACKETS_WITH_FREEBSD_HEADER/) |
| WTAP-093 | DLT-188 | IEEE 802.16 MAC CPS | 4 | [pcapng](pcaps/encap_types/WTAP-093_DLT-188_IEEE_80216_MAC_COMMON_PART_SUBLAYER/) |
| WTAP-095 | DLT-189 | USB (Linux Header) | 2,606 | [pcapng](pcaps/encap_types/WTAP-095_DLT-189_USB_PACKETS_WITH_LINUX_HEADER/) |
| WTAP-097 | DLT-192 | PPI (Per-Packet Info) | 78,078 | [pcapng](pcaps/encap_types/WTAP-097_DLT-192_PER-PACKET_INFORMATION_HEADER/) |
| WTAP-098 | DLT-197 | Encap 98 | 1 | [pcapng](pcaps/encap_types/WTAP-098_DLT-197_ENCAP_98/) |
| WTAP-099 | DLT-201 | Encap 99 | 90 | [pcapng](pcaps/encap_types/WTAP-099_DLT-201_ENCAP_99/) |
| WTAP-100 | DLT-196 | Encap 100 | 12 | [pcapng](pcaps/encap_types/WTAP-100_DLT-196_ENCAP_100/) |
| WTAP-101 | DLT-142 | SS7 SCCP | 3,066 | [pcapng](pcaps/encap_types/WTAP-101_DLT-142_SS7_SCCP/) |
| WTAP-103 | DLT-199 | IPMB (Kontron Header) | 37,379 | [pcapng](pcaps/encap_types/WTAP-103_DLT-199_INTELLIGENT_PLATFORM_MANAGEMENT_BUS_WITH_KONTRON_PSEUDO-HEADER/) |
| WTAP-104 | DLT-195 | IEEE 802.15.4 | 38,812 | [pcapng](pcaps/encap_types/WTAP-104_DLT-195_IEEE_802154_WIRELESS_PAN/) |
| WTAP-109 | DLT-190 | CAN 2.0B | 433 | [pcapng](pcaps/encap_types/WTAP-109_DLT-190_CONTROLLER_AREA_NETWORK_20B/) |
| WTAP-111 | DLT-213 | X2E Serial | 4 | [pcapng](pcaps/encap_types/WTAP-111_DLT-213_X2E_SERIAL_LINE_CAPTURE/) |
| WTAP-115 | DLT-220 | USB (Linux Header + Padding) | 3,399 | [pcapng](pcaps/encap_types/WTAP-115_DLT-220_USB_PACKETS_WITH_LINUX_HEADER_AND_PADDING/) |
| WTAP-121 | DLT-224 | Fibre Channel FC-2 | 254 | [pcapng](pcaps/encap_types/WTAP-121_DLT-224_FIBRE_CHANNEL_FC-2/) |
| WTAP-124 | DLT-226 | Solaris IPNET | 282 | [pcapng](pcaps/encap_types/WTAP-124_DLT-226_SOLARIS_IPNET/) |
| WTAP-125 | DLT-227 | SocketCAN | 82 | [pcapng](pcaps/encap_types/WTAP-125_DLT-227_SOCKETCAN/) |
| WTAP-127 | DLT-230 | IEEE 802.15.4 (no FCS) | 41,895 | [pcapng](pcaps/encap_types/WTAP-127_DLT-230_IEEE_802154_WIRELESS_PAN_WITH_FCS_NOT_PRESENT/) |
| WTAP-129 | DLT-228 | Raw IPv4 | 39,483 | [pcapng](pcaps/encap_types/WTAP-129_DLT-228_RAW_IPV4/) |
| WTAP-130 | DLT-229 | Raw IPv6 | 36,639 | [pcapng](pcaps/encap_types/WTAP-130_DLT-229_RAW_IPV6/) |
| WTAP-131 | DLT-203 | LAPD | 2,042 | [pcapng](pcaps/encap_types/WTAP-131_DLT-203_LAPD/) |
| WTAP-138 | DLT-243 | MPEG2-TS | 8,073 | [pcapng](pcaps/encap_types/WTAP-138_DLT-243_ISOIEC_13818-1_MPEG2-TS/) |
| WTAP-141 | DLT-239 | NFLOG | 20,040 | [pcapng](pcaps/encap_types/WTAP-141_DLT-239_NFLOG/) |
| WTAP-146 | DLT-231 | D-Bus | 1,742 | [pcapng](pcaps/encap_types/WTAP-146_DLT-231_D-BUS/) |
| WTAP-147 | DLT-202 | AX.25 + KISS Header | 1,442 | [pcapng](pcaps/encap_types/WTAP-147_DLT-202_AX25_WITH_KISS_HEADER/) |
| WTAP-149 | DLT-248 | SCTP | 32,576 | [pcapng](pcaps/encap_types/WTAP-149_DLT-248_SCTP/) |
| WTAP-151 | DLT-136 | Juniper Services | 76 | [pcapng](pcaps/encap_types/WTAP-151_DLT-136_JUNIPER_SERVICES/) |
| WTAP-152 | DLT-249 | USB (USBPcap Header) | 3,244 | [pcapng](pcaps/encap_types/WTAP-152_DLT-249_USB_PACKETS_WITH_USBPCAP_HEADER/) |
| WTAP-154 | DLT-251 | Bluetooth LE Link Layer | 14,025 | [pcapng](pcaps/encap_types/WTAP-154_DLT-251_BLUETOOTH_LOW_ENERGY_LINK_LAYER/) |
| WTAP-155 | DLT-252 | Wireshark Upper PDU Export | 32,390 | [pcapng](pcaps/encap_types/WTAP-155_DLT-252_WIRESHARK_UPPER_PDU_EXPORT/) |
| WTAP-158 | DLT-253 | Linux Netlink | 27,632 | [pcapng](pcaps/encap_types/WTAP-158_DLT-253_LINUX_NETLINK/) |
| WTAP-174 | DLT-108 | OpenBSD Loopback | 41,093 | [pcapng](pcaps/encap_types/WTAP-174_DLT-108_OPENBSD_LOOPBACK/) |
| WTAP-175 | DLT-000 | JSON | 1 | [pcapng](pcaps/encap_types/WTAP-175_DLT-000_JAVASCRIPT_OBJECT_NOTATION/) |
| WTAP-178 | DLT-170 | GFP Transparent Mode | 478 | [pcapng](pcaps/encap_types/WTAP-178_DLT-170_ITU-T_G7041Y1303_GENERIC_FRAMING_PROCEDURE_TRANSPARENT_MODE/) |
| WTAP-179 | DLT-171 | GFP Frame-Mapped Mode | 33,564 | [pcapng](pcaps/encap_types/WTAP-179_DLT-171_ITU-T_G7041Y1303_GENERIC_FRAMING_PROCEDURE_FRAME-MAPPED_MODE/) |
| WTAP-181 | DLT-184 | Juniper VN | 4 | [pcapng](pcaps/encap_types/WTAP-181_DLT-184_JUNIPER_VN/) |
| WTAP-186 | DLT-272 | nRF Sniffer for BLE | 653 | [pcapng](pcaps/encap_types/WTAP-186_DLT-272_NRF_SNIFFER_FOR_BLUETOOTH_LE/) |
| WTAP-197 | DLT-200 | Juniper Secure Tunnel | 120 | [pcapng](pcaps/encap_types/WTAP-197_DLT-200_JUNIPER_SECURE_TUNNEL_INFORMATION/) |
| WTAP-206 | DLT-283 | IEEE 802.15.4 + TAP Header | 36,049 | [pcapng](pcaps/encap_types/WTAP-206_DLT-283_IEEE_802154_WIRELESS_WITH_TAP_PSEUDO-HEADER/) |
| WTAP-208 | DLT-288 | USB 2.0 (1.1.0) | 216 | [pcapng](pcaps/encap_types/WTAP-208_DLT-288_USB_201110_PACKETS/) |
| WTAP-210 | DLT-276 | Linux Cooked Capture v2 | 40,431 | [pcapng](pcaps/encap_types/WTAP-210_DLT-276_LINUX_COOKED-MODE_CAPTURE_V2/) |
| WTAP-211 | DLT-287 | Z-Wave Serial API | 4 | [pcapng](pcaps/encap_types/WTAP-211_DLT-287_Z-WAVE_SERIAL_API_PACKETS/) |
| WTAP-212 | DLT-290 | ETW (Event Tracing for Windows) | 4,755 | [pcapng](pcaps/encap_types/WTAP-212_DLT-290_EVENT_TRACING_FOR_WINDOWS_MESSAGES/) |
| WTAP-214 | DLT-292 | ZBOSS NCP | 1,521 | [pcapng](pcaps/encap_types/WTAP-214_DLT-292_ZBOSS_NCP/) |
| WTAP-215 | DLT-293 | Low-Speed USB 2.0 (1.1.0) | 229 | [pcapng](pcaps/encap_types/WTAP-215_DLT-293_LOW-SPEED_USB_201110_PACKETS/) |
| WTAP-216 | DLT-294 | Full-Speed USB 2.0 (1.1.0) | 213 | [pcapng](pcaps/encap_types/WTAP-216_DLT-294_FULL-SPEED_USB_201110_PACKETS/) |
| WTAP-217 | DLT-295 | High-Speed USB 2.0 | 253 | [pcapng](pcaps/encap_types/WTAP-217_DLT-295_HIGH-SPEED_USB_20_PACKETS/) |
| WTAP-219 | DLT-296 | Auerswald Log | 4 | [pcapng](pcaps/encap_types/WTAP-219_DLT-296_AUERSWALD_LOG/) |
| WTAP-220 | DLT-289 | ATSC ALP (A/330) | 39,154 | [pcapng](pcaps/encap_types/WTAP-220_DLT-289_ATSC_LINK-LAYER_PROTOCOL_A330_PACKETS/) |
| WTAP-221 | DLT-299 | FiRa UCI | 1,560 | [pcapng](pcaps/encap_types/WTAP-221_DLT-299_FIRA_UWB_CONTROLLER_INTERFACE_UCI_PROTOCOL/) |
| WTAP-222 | DLT-298 | Silicon Labs Debug Channel | 1,299 | [pcapng](pcaps/encap_types/WTAP-222_DLT-298_SILABS_DEBUG_CHANNEL/) |
| WTAP-223 | DLT-300 | MDB (Multi-Drop Bus) | 36,788 | [pcapng](pcaps/encap_types/WTAP-223_DLT-300_MDB_MULTI-DROP_BUS/) |

### Unknown Encap Types (WTAP-000)

<details>
<summary>67 unknown DLT types with 390,191 total packets (click to expand)</summary>

| DLT | Name | Packets | Pcap |
|-----|------|--------:|------|
| DLT-005 | DLT_CHAOS | 10,701 | [pcapng](pcaps/encap_types/WTAP-000_DLT-005_UNKNOWN/) |
| DLT-017 | Unassigned | 2,970 | [pcapng](pcaps/encap_types/WTAP-000_DLT-017_UNKNOWN/) |
| DLT-022 | Unassigned | 6,274 | [pcapng](pcaps/encap_types/WTAP-000_DLT-022_UNKNOWN/) |
| DLT-027 | Unassigned | 10,433 | [pcapng](pcaps/encap_types/WTAP-000_DLT-027_UNKNOWN/) |
| DLT-031 | Unassigned | 10,375 | [pcapng](pcaps/encap_types/WTAP-000_DLT-031_UNKNOWN/) |
| DLT-034 | Unassigned | 2 | [pcapng](pcaps/encap_types/WTAP-000_DLT-034_UNKNOWN/) |
| DLT-035 | Unassigned | 12,642 | [pcapng](pcaps/encap_types/WTAP-000_DLT-035_UNKNOWN/) |
| DLT-036 | Unassigned | 440 | [pcapng](pcaps/encap_types/WTAP-000_DLT-036_UNKNOWN/) |
| DLT-040 | Unassigned | 7,239 | [pcapng](pcaps/encap_types/WTAP-000_DLT-040_UNKNOWN/) |
| DLT-062 | Unassigned | 13,938 | [pcapng](pcaps/encap_types/WTAP-000_DLT-062_UNKNOWN/) |
| DLT-064 | Unassigned | 9,737 | [pcapng](pcaps/encap_types/WTAP-000_DLT-064_UNKNOWN/) |
| DLT-065 | Unassigned | 7,472 | [pcapng](pcaps/encap_types/WTAP-000_DLT-065_UNKNOWN/) |
| DLT-067 | Unassigned | 11,049 | [pcapng](pcaps/encap_types/WTAP-000_DLT-067_UNKNOWN/) |
| DLT-068 | Unassigned | 12,939 | [pcapng](pcaps/encap_types/WTAP-000_DLT-068_UNKNOWN/) |
| DLT-070 | Unassigned | 2 | [pcapng](pcaps/encap_types/WTAP-000_DLT-070_UNKNOWN/) |
| DLT-071 | Unassigned | 13,067 | [pcapng](pcaps/encap_types/WTAP-000_DLT-071_UNKNOWN/) |
| DLT-072 | Unassigned | 8,868 | [pcapng](pcaps/encap_types/WTAP-000_DLT-072_UNKNOWN/) |
| DLT-073 | Unassigned | 9,499 | [pcapng](pcaps/encap_types/WTAP-000_DLT-073_UNKNOWN/) |
| DLT-074 | Unassigned | 2 | [pcapng](pcaps/encap_types/WTAP-000_DLT-074_UNKNOWN/) |
| DLT-076 | Unassigned | 13,076 | [pcapng](pcaps/encap_types/WTAP-000_DLT-076_UNKNOWN/) |
| DLT-077 | Unassigned | 2 | [pcapng](pcaps/encap_types/WTAP-000_DLT-077_UNKNOWN/) |
| DLT-078 | Unassigned | 2 | [pcapng](pcaps/encap_types/WTAP-000_DLT-078_UNKNOWN/) |
| DLT-079 | Unassigned | 7,326 | [pcapng](pcaps/encap_types/WTAP-000_DLT-079_UNKNOWN/) |
| DLT-080 | Unassigned | 2 | [pcapng](pcaps/encap_types/WTAP-000_DLT-080_UNKNOWN/) |
| DLT-081 | Unassigned | 11,144 | [pcapng](pcaps/encap_types/WTAP-000_DLT-081_UNKNOWN/) |
| DLT-082 | Unassigned | 10,050 | [pcapng](pcaps/encap_types/WTAP-000_DLT-082_UNKNOWN/) |
| DLT-083 | Unassigned | 12,857 | [pcapng](pcaps/encap_types/WTAP-000_DLT-083_UNKNOWN/) |
| DLT-084 | Unassigned | 9,769 | [pcapng](pcaps/encap_types/WTAP-000_DLT-084_UNKNOWN/) |
| DLT-085 | Unassigned | 9,767 | [pcapng](pcaps/encap_types/WTAP-000_DLT-085_UNKNOWN/) |
| DLT-086 | Unassigned | 8,044 | [pcapng](pcaps/encap_types/WTAP-000_DLT-086_UNKNOWN/) |
| DLT-087 | Unassigned | 10,525 | [pcapng](pcaps/encap_types/WTAP-000_DLT-087_UNKNOWN/) |
| DLT-088 | Unassigned | 4,936 | [pcapng](pcaps/encap_types/WTAP-000_DLT-088_UNKNOWN/) |
| DLT-089 | Unassigned | 12,044 | [pcapng](pcaps/encap_types/WTAP-000_DLT-089_UNKNOWN/) |
| DLT-091 | Unassigned | 9,480 | [pcapng](pcaps/encap_types/WTAP-000_DLT-091_UNKNOWN/) |
| DLT-092 | Unassigned | 129 | [pcapng](pcaps/encap_types/WTAP-000_DLT-092_UNKNOWN/) |
| DLT-093 | Unassigned | 2 | [pcapng](pcaps/encap_types/WTAP-000_DLT-093_UNKNOWN/) |
| DLT-094 | Unassigned | 2,220 | [pcapng](pcaps/encap_types/WTAP-000_DLT-094_UNKNOWN/) |
| DLT-096 | Unassigned | 327 | [pcapng](pcaps/encap_types/WTAP-000_DLT-096_UNKNOWN/) |
| DLT-102 | DLT_SLIP_BSDOS | 3 | [pcapng](pcaps/encap_types/WTAP-000_DLT-102_UNKNOWN/) |
| DLT-103 | DLT_PPP_BSDOS | 24 | [pcapng](pcaps/encap_types/WTAP-000_DLT-103_UNKNOWN/) |
| DLT-110 | LINKTYPE_LANE8023 | 329 | [pcapng](pcaps/encap_types/WTAP-000_DLT-110_UNKNOWN/) |
| DLT-111 | DLT_HIPPI | 2 | [pcapng](pcaps/encap_types/WTAP-000_DLT-111_UNKNOWN/) |
| DLT-116 | DLT_IPFILTER | 2 | [pcapng](pcaps/encap_types/WTAP-000_DLT-116_UNKNOWN/) |
| DLT-120 | DLT_AIRONET_HEADER | 12,354 | [pcapng](pcaps/encap_types/WTAP-000_DLT-120_UNKNOWN/) |
| DLT-124 | DLT_RIO | 4,803 | [pcapng](pcaps/encap_types/WTAP-000_DLT-124_UNKNOWN/) |
| DLT-126 | DLT_AURORA | 6,726 | [pcapng](pcaps/encap_types/WTAP-000_DLT-126_UNKNOWN/) |
| DLT-132 | DLT_JUNIPER_ES | 267 | [pcapng](pcaps/encap_types/WTAP-000_DLT-132_UNKNOWN/) |
| DLT-134 | DLT_JUNIPER_MFR | 1,470 | [pcapng](pcaps/encap_types/WTAP-000_DLT-134_UNKNOWN/) |
| DLT-145 | DLT_IBM_SP | 9 | [pcapng](pcaps/encap_types/WTAP-000_DLT-145_UNKNOWN/) |
| DLT-164 | DLT_JUNIPER_MONITOR | 245 | [pcapng](pcaps/encap_types/WTAP-000_DLT-164_UNKNOWN/) |
| DLT-166 | DLT_PPP_PPPD | 238 | [pcapng](pcaps/encap_types/WTAP-000_DLT-166_UNKNOWN/) |
| DLT-168 | DLT_JUNIPER_PPPOE_ATM | 233 | [pcapng](pcaps/encap_types/WTAP-000_DLT-168_UNKNOWN/) |
| DLT-175 | DLT_ERF_ETH | 554 | [pcapng](pcaps/encap_types/WTAP-000_DLT-175_UNKNOWN/) |
| DLT-176 | DLT_ERF_POS | 12,412 | [pcapng](pcaps/encap_types/WTAP-000_DLT-176_UNKNOWN/) |
| DLT-182 | DLT_MFR | 2,577 | [pcapng](pcaps/encap_types/WTAP-000_DLT-182_UNKNOWN/) |
| DLT-185 | DLT_A653_ICM | 14 | [pcapng](pcaps/encap_types/WTAP-000_DLT-185_UNKNOWN/) |
| DLT-191 | DLT_IEEE802_15_4_LINUX | 13,357 | [pcapng](pcaps/encap_types/WTAP-000_DLT-191_UNKNOWN/) |
| DLT-193 | DLT_IEEE802_16_MAC_CPS_RADIO | 12,476 | [pcapng](pcaps/encap_types/WTAP-000_DLT-193_UNKNOWN/) |
| DLT-194 | DLT_JUNIPER_ISM | 11,096 | [pcapng](pcaps/encap_types/WTAP-000_DLT-194_UNKNOWN/) |
| DLT-198 | DLT_RAIF1 | 13,067 | [pcapng](pcaps/encap_types/WTAP-000_DLT-198_UNKNOWN/) |
| DLT-205 | DLT_C_HDLC_WITH_DIR | 12,330 | [pcapng](pcaps/encap_types/WTAP-000_DLT-205_UNKNOWN/) |
| DLT-207 | DLT_LAPB_WITH_DIR | 13,245 | [pcapng](pcaps/encap_types/WTAP-000_DLT-207_UNKNOWN/) |
| DLT-223 | DLT_WIHART | 47 | [pcapng](pcaps/encap_types/WTAP-000_DLT-223_UNKNOWN/) |
| DLT-244 | DLT_NG40 | 2 | [pcapng](pcaps/encap_types/WTAP-000_DLT-244_UNKNOWN/) |
| DLT-262 | DLT_ZWAVE_R3 | 253 | [pcapng](pcaps/encap_types/WTAP-000_DLT-262_UNKNOWN/) |
| DLT-302 | DLT_EDK2_MM | 1,424 | [pcapng](pcaps/encap_types/WTAP-000_DLT-302_UNKNOWN/) |
| DLT-303 | DLT_DEBUG_ONLY | 305 | [pcapng](pcaps/encap_types/WTAP-000_DLT-303_UNKNOWN/) |

</details>

## License

MIT
