+++
title = 'SFP, QSFP, Fiber Types'
date = 2026-07-19T18:00:00+08:00
draft = false
categories = ['Network']
tags = ['SFP', 'QSFP', '400G', '800G', 'Fiber Optics', 'Data Center', 'Ethernet', 'Network']
+++

Choosing an optical transceiver is harder than matching the number printed on the switch port. A working link has to align the **form factor, host lane rate, Ethernet standard, optical organization, fiber type, connector, wavelength, FEC mode, loss budget, and platform support** at both ends.

The first clarification is terminology: **SFP** is a specific one-lane form factor, not the general name for every removable optic. Engineers often say “SFP” informally, but 400G and 800G modules are normally **QSFP112, QSFP-DD, QSFP-DD800, or OSFP**.

This article builds the subject from ordinary SFP modules and OM/OS fiber through modern 400G and 800G optics, including representative models, photographs, lane arithmetic, breakout behavior, and a practical selection workflow.

{{< toc >}}

## 1. Read every transceiver name in layers

A label such as `QDD-400G-DR4-S` describes several independent properties:

| Layer | Example | What it tells you |
| --- | --- | --- |
| Form factor | SFP+, SFP28, QSFP28, QSFP-DD, OSFP | The physical cage, electrical connector, lane count, power envelope, and cooling model |
| Aggregate Ethernet rate | 10G, 100G, 400G, 800G | The nominal MAC-layer capacity |
| Optical class | SR, DR, FR, LR, ER, ZR | Approximate reach and optical technology |
| Optical lane count | 4, 8 | Parallel fibers or WDM wavelengths, depending on the standard |
| Breakout organization | 2X400G, 4X100G, 8X100G | Independent subports exposed by the module |
| Vendor suffix | `-S`, `-I`, `P` | Vendor-specific temperature, connector, feature, or product-class information |

Do not infer the speed from appearance. A 1G SFP, 10G SFP+, and 25G SFP28 can use nearly identical metal housings and duplex-LC fronts.

## 2. What the hardware looks like

### 2.1 Standard optical SFP

![Optical SFP module with a duplex optical receptacle](/posts/sfp-qsfp-fiber-400g-800g-optics/optical-sfp.jpg)

The classic optical SFP-family module is narrow and usually has two small LC openings: one transmit and one receive. SX/SR multimode and LX/LR single-mode models may look the same externally. Read the label or the module EEPROM rather than relying on pull-tab color.

### 2.2 Copper RJ45 SFP

![Copper SFP module with an RJ45 socket](/posts/sfp-qsfp-fiber-400g-800g-optics/rj45-sfp.png)

An RJ45 SFP replaces the optical receptacle with a shielded twisted-pair Ethernet socket. A 1G 1000BASE-T SFP commonly supports 100 m, while a 10GBASE-T SFP+ may be limited to about 30 m and consume substantially more power. Cisco's current [1G SFP documentation](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/gigabit-ethernet-gbic-sfp-modules/datasheet-c78-366584.html) and [10G SFP+ documentation](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/transceiver-modules/data_sheet_c78-455693.html) are useful examples of the power and reach differences.

### 2.3 Compact and bidirectional SFPs

![Compact dual-channel bidirectional SFP](/posts/sfp-qsfp-fiber-400g-800g-optics/compact-bidi-sfp.png)

The pictured Compact SFP contains two single-fiber BiDi channels. A more common BX/BiDi module presents one simplex LC port and sends the two directions on different wavelengths over one strand. BiDi modules must be installed as complementary pairs: for example, an upstream unit transmitting at one wavelength connects to a downstream unit transmitting at the other.

### 2.4 DAC and AOC assemblies

![SFP+ direct-attach copper cable](/posts/sfp-qsfp-fiber-400g-800g-optics/sfp-dac.png)

A **direct-attach cable (DAC)** is twinax copper with the transceiver ends permanently attached. Passive DAC is normally the lowest-power, lowest-cost choice for a short in-rack link.

![SFP+ active optical cable](/posts/sfp-qsfp-fiber-400g-800g-optics/sfp-aoc.jpg)

An **active optical cable (AOC)** also has permanently attached ends, but the cable between them is fiber. It is thinner and easier to route than twinax and can reach farther, but a damaged connector normally requires replacing the complete assembly.

### 2.5 QSFP is visibly wider

![QSFP100 optical modules with duplex-LC and MPO fronts](/posts/sfp-qsfp-fiber-400g-800g-optics/qsfp100-optical-lineup.png)

QSFP-family modules are wider because they expose four host electrical lanes. The photograph shows that aggregate rate still does not determine the front connector: LR4 and CWDM4 use duplex LC, while SR4 and PSM4 use a rectangular MPO connector.

![QSFP28-to-SFP28 adapter with an SFP module inserted](/posts/sfp-qsfp-fiber-400g-800g-optics/qsfp-to-sfp-adapter.jpg)

A QSA adapter places a one-lane SFP/SFP+/SFP28 module in a four-lane QSFP cage. It only works when the switch supports the lower rate and the required lane mapping on that port.

## 3. Form factors from 1G through 800G

![Relative form factors and electrical lane counts](/posts/sfp-qsfp-fiber-400g-800g-optics/form-factor-comparison.svg)

| Form factor | Common Ethernet rates | Host lanes | Typical role |
| --- | --- | ---: | --- |
| SFP | 100M/1G | 1 | Access and legacy links |
| SFP+ | 10G | 1 | Server, storage, aggregation, DAC and AOC |
| SFP28 | 25G | 1 | Server access and leaf connectivity |
| SFP56 | 50G | 1 × 50G PAM4 | Newer server and switch links |
| QSFP+ | 40G | 4 × 10G | Four-lane aggregation and breakout |
| QSFP28 | 100G | 4 × 25G | Data-center uplinks and 4×25G breakout |
| QSFP56 | 200G | 4 × 50G PAM4 | Modern spine/leaf fabrics |
| QSFP112 | 400G | 4 × approximately 100G PAM4 | NICs and modern AI/HPC switches |
| QSFP-DD | 400G | Commonly 8 × approximately 50G PAM4 | Dense 400G switch and router ports |
| QSFP-DD800 | 800G | 8 × approximately 100G PAM4 | Dense 800G client and coherent optics |
| OSFP | 400G/800G | 8 lanes | Larger module with greater thermal headroom |

The [QSFP-DD MSA](https://www.qsfp-dd.com/specification/) defines eight-lane QSFP-DD, QSFP-DD800, and newer generations. A QSFP-DD cage is mechanically designed to accept earlier four-lane QSFP modules, but the host still decides which electrical rates, protocols, FEC modes, and module power classes are supported. OSFP is a different physical format and does not fit a QSFP-DD cage.

## 4. Fiber types: multimode and single-mode

![Yellow single-mode and orange multimode LC patch cords](/posts/sfp-qsfp-fiber-400g-800g-optics/singlemode-multimode-patch-cords.jpg)

Jacket color is a useful clue, not proof. Read the printed cable type because installations are often inconsistent and regional conventions vary.

![Comparison of multimode and single-mode fiber core sizes](/posts/sfp-qsfp-fiber-400g-800g-optics/fiber-core-comparison.svg)

### 4.1 Multimode fiber

Multimode fiber has a larger core. Multiple optical paths propagate through it, producing modal dispersion that limits reach as data rate increases.

| Grade | Core/cladding | Common color clue | Practical position |
| --- | --- | --- | --- |
| OM1 | 62.5/125 µm | Orange | Legacy; unsuitable for most new high-speed designs |
| OM2 | 50/125 µm | Orange | Improved legacy multimode |
| OM3 | 50/125 µm | Aqua | Laser-optimized baseline for many 10G–100G links |
| OM4 | 50/125 µm | Aqua or violet | Higher bandwidth and longer SR/VR reach |
| OM5 | 50/125 µm | Lime green | Wideband MMF for shortwave wavelength multiplexing |

OM5 does not automatically extend every ordinary 850 nm SR link. The optic must use the wideband capability.

### 4.2 Single-mode fiber

Single-mode fiber has an effective core near 9 µm and supports one spatial mode. It is used with approximately 1310 nm client optics and 1550 nm/DWDM coherent systems.

| Grade | Common color clue | Typical use |
| --- | --- | --- |
| OS1 | Yellow | Indoor, premises, often tight-buffered construction |
| OS2 | Yellow | Low-loss campus, outside plant, data-center interconnect and long haul |

The Fiber Optic Association maintains a useful [comparison of OM and OS fiber types](https://www.thefoa.org/tech/ref/OSP/fiber.html), including core sizes, attenuation, bandwidth, and color-code references.

## 5. Connector types matter as much as the fiber

### 5.1 First separate the module body from the cable connector

QSFP and OSFP describe the **pluggable module body** that enters the switch cage. LC and MPO describe the **fiber connector** that enters the front of that module. Therefore, “an OSFP cable” or “a QSFP connector” is incomplete wording: an OSFP or QSFP-family optic may present duplex LC, one MPO, dual MPO, or another optical interface.

| QSFP-family module | OSFP module |
| --- | --- |
| ![QSFP112 400G module with a single MPO receptacle](/posts/sfp-qsfp-fiber-400g-800g-optics/qsfp112-400g-dr4.png) | ![OSFP 800G module with integrated heat sink and an MPO-16 receptacle](/posts/sfp-qsfp-fiber-400g-800g-optics/osfp-800g-dr8p.jpg) |
| QSFP is the narrower four-lane family; QSFP-DD adds a second row of host contacts for eight lanes. The front shown here is MPO, but other QSFP optics use duplex LC. | OSFP is wider and commonly has an integrated heat sink for higher-power 400G/800G optics. It uses a different cage and may expose MPO-16, dual MPO-12, or LC interfaces. |

| Duplex LC cable connectors | MPO multi-fiber connector |
| --- | --- |
| ![Single-mode and multimode duplex LC patch cords](/posts/sfp-qsfp-fiber-400g-800g-optics/singlemode-multimode-patch-cords.jpg) | ![Close-up of a pinned male MPO connector end face](/posts/sfp-qsfp-fiber-400g-800g-optics/mpo-connector-closeup.jpg) |
| LC carries one fiber per ferrule. A duplex clip normally keeps the Tx and Rx fibers together. LC is common on wavelength-multiplexed FR4/LR4/ZR optics because several wavelengths share one fiber pair. | MPO places a row of fibers between two alignment-pin positions. This photograph shows a **pinned/male** plug; it must mate with an unpinned/female interface. MPO is common on parallel SR4, DR4, VR8, and DR8 optics. |

### 5.2 MPO-8 and MPO-12 can look identical from the outside

“MPO-8” usually means a Base-8 assembly with eight active fibers in the outer positions of a 12-position MPO footprint. A fully populated MPO-12 uses all twelve positions. The outer latch, boot, key, and housing can be the same, so do not identify the fiber count from jacket or connector color alone—check the end face, label, and manufacturer part number.

![MPO-8, MPO-12, and MPO-16 fiber-position comparison](/posts/sfp-qsfp-fiber-400g-800g-optics/mpo-fiber-position-comparison.svg)

For a typical four-lane parallel optic, positions 1–4 carry one direction and positions 9–12 carry the other; the middle four positions are unused. MPO-16 instead has sixteen positions on a tighter pitch and an offset key, so it does not mate with MPO-12. Some 800G modules avoid MPO-16 by using **dual MPO-12**, with each connector carrying one eight-fiber group. SENKO's connector documentation illustrates the [center-key MPO-12 versus offset-key MPO-16 distinction](https://www.senko.com/product/mpo-plus-standard-connector/).

### 5.3 Connector selection table

| Connector | Where it appears | Main caution |
| --- | --- | --- |
| Duplex LC | SFP SR/LR, 400G FR4/LR4, coherent ZR | Confirm UPC versus APC and Tx/Rx polarity |
| Simplex LC | BX/BiDi | Both endpoints must use complementary wavelength pairs |
| MPO-12 footprint, eight active fibers (MPO-8/Base-8) | SR4, DR4, PSM4, four-lane parallel breakouts | Confirm which eight positions are populated, plus polarity, key orientation, pins, and APC/UPC |
| MPO-12, twelve active fibers (Base-12) | Trunks, cassettes, and some multiport breakouts | Do not assume a twelve-fiber trunk matches an eight-active-fiber transceiver map |
| MPO-16 | 800G DR8/VR8 | Sixteen active fibers; not interchangeable with MPO-12 |
| Dual MPO-12 | Some 800G DR8/VR8 modules | Two independent eight-fiber groups |
| SC/ST | Legacy panels and industrial environments | Larger connector; adapter loss must enter the link budget |

Blue commonly indicates UPC and green indicates APC, but color is only a clue. Never directly mate APC and UPC end faces. With MPO, also check whether the module is **pinned/male** and therefore requires an **unpinned/female** cable connector.

## 6. Optical reach codes

The suffix is a technology class, not a complete specification. Exact distance depends on Ethernet generation, fiber grade, connector loss, FEC, and vendor implementation.

| Code | Usual medium | Representative reach | Organization |
| --- | --- | ---: | --- |
| SX | MMF, around 850 nm | Hundreds of metres at 1G | One optical lane |
| SR | MMF, around 850 nm | 10G SR: 300 m OM3 / 400 m OM4 | One or several short-reach lanes |
| VR | MMF | About 30–50 m at 400G/800G | Very-short-reach parallel PAM4 |
| LX/LR | SMF, around 1310 nm | Commonly 10 km | One lane or WDM lanes |
| DR | SMF, around 1310 nm | Commonly 500 m | Parallel single-mode lanes; breakout-friendly |
| FR | SMF | Commonly 2 km | One 100G wavelength or four WDM wavelengths |
| ER | SMF | Commonly 40 km | Higher optical budget |
| ZR | SMF/DWDM | Approximately 80–120 km | Coherent at 400G/800G; older 1G/10G usage differs |
| ZR+ | Amplified DWDM | Regional to long haul | Coherent, line-system dependent |

Older 1G names such as EX and ZX and older 10G ZR products are not equivalent to modern 400ZR/800ZR coherent specifications.

## 7. How 400G and 800G lane arithmetic works

Modern high-speed host interfaces use **PAM4**, which transmits two bits per symbol by using four amplitude levels.

![400G and 800G host-side PAM4 lane arithmetic](/posts/sfp-qsfp-fiber-400g-800g-optics/host-lane-math.svg)

A modern 100G electrical lane operates at roughly:

```text
53.125 gigabaud × 2 bits per PAM4 symbol = 106.25 Gb/s
```

Therefore:

```text
400G: 4 × 106.25 Gb/s = 425 Gb/s raw
800G: 8 × 106.25 Gb/s = 850 Gb/s raw
```

PCS framing, alignment markers, and forward error correction consume the difference between the raw line rate and the nominal 400/800 Gb/s Ethernet rate. Earlier 400G QSFP-DD implementations commonly use eight approximately 50G electrical lanes instead of four 100G lanes.

The transmit path is conceptually:

```text
Ethernet MAC
  → PCS encoding and FEC
  → electrical PAM4 SerDes lanes
  → module CDR / retimer / gearbox / DSP
  → laser drivers and modulators
  → optical fibers or wavelengths
```

The receiver reverses that process. Most high-speed PAM4 links depend on FEC; receiving light is not enough. Incorrect FEC can produce a link that remains down or shows an unusably high bit-error rate.

## 8. Three optical organizations

![Parallel, WDM and coherent optical organizations](/posts/sfp-qsfp-fiber-400g-800g-optics/optical-organization.svg)

### 8.1 Parallel optics: VR4, DR4, VR8 and DR8

Every lane has a separate transmit and receive fiber.

**400G DR4** uses four 100G transmit fibers and four receive fibers—eight active fibers in an MPO-12 connector. **800G DR8** uses eight transmit and eight receive fibers—sixteen active fibers in MPO-16 or dual MPO-12.

Parallel optics consume more fiber but map cleanly to breakout links.

### 8.2 Wavelength multiplexing: FR4 and LR4

Four differently colored optical carriers are combined onto one transmit fiber and separated from one receive fiber. A 400G FR4 link therefore needs only a duplex-LC pair even though it contains four 100G optical wavelengths.

The trade-off is breakout. A normal 400G FR4 module presents one 400G Ethernet interface; a passive cable cannot turn its four internal wavelengths into four independent 100G ports. Use an explicitly labeled `4X100G`, `8X100G`, or `2X400G` module when independent endpoints are required.

### 8.3 Coherent optics: ZR and ZR+

Coherent optics encode information in the amplitude, phase, and polarization of one tunable DWDM carrier. The receiver combines the incoming light with a local laser and uses high-speed ADCs and DSP to reconstruct the signal.

The result is much longer reach and compatibility with amplified DWDM networks, at the cost of more power, heat, DSP latency, and optical engineering.

## 9. Representative 400G modules

![Cisco QSFP112 400G DR4 module with MPO connector](/posts/sfp-qsfp-fiber-400g-800g-optics/qsfp112-400g-dr4.png)

The photographed QSFP112 module uses four approximately 100G host lanes and an MPO optical interface.

| Model class | Fiber and connector | Reach | How it works |
| --- | --- | ---: | --- |
| `QSFP-400G-VR4` | MMF, MPO-12 APC | About 50 m | Four parallel 100G PAM4 lanes for short AI/HPC links |
| `QSFP-400G-DR4` / `QDD-400G-DR4-S` | SMF, MPO-12 APC | 500 m | Four parallel 100G lanes; can break out to 4×100G DR1 or 2×200G DR2 |
| `QSFP-400G-FR4` / `QDD-400G-FR4-S` | SMF, duplex LC UPC | 2 km | Four CWDM wavelengths multiplexed onto two fibers |
| `QDD-400G-LR4-S` | SMF, duplex LC | 10 km | Four WDM wavelengths for campus/DCI |
| `QDD-400G-LR8-S` | SMF, duplex LC | 10 km | Older eight-wavelength organization |
| `QDD-4X100G-FR-S` | SMF, MPO-12 APC | 2 km | Four independent 100G FR1 interfaces |
| `QDD-4X100G-LR-S` | SMF, MPO-12 APC | 10 km | Four independent 100G LR1 interfaces |
| `QDD-400G-BD` | OM5 MMF, duplex LC | About 100 m | Bidirectional wavelength scheme over an existing duplex pair |
| `QDD-400G-ZR-S` | SMF, duplex LC, DWDM | 80–120 km class | One coherent 400G wavelength |
| `QDD-400G-ZRP-S` / ULH | Amplified DWDM | Line-system dependent | Regional to ultra-long-haul coherent operation |

Cisco's current [400G QSFP112 data sheet](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/400g-qsfp-transceiver-modules-ds.html) describes VR4, DR4, and FR4 implementations; the broader [400G QSFP-DD portfolio](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/transceiver-modules/datasheet-c78-743172.html) includes SR/VR, DR, FR, LR, BiDi, breakout, DAC, and AOC models.

For coherent transport, 400ZR targets interoperable single-span DCI, while ZR+ and newer ultra-long-haul modules support configurable modulation, channel spacing, and amplified networks. Cisco documents both its [400G ZR/ZR+ modules](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/transceiver-modules/datasheet-c78-744377.html) and a newer [400G ultra-long-haul QSFP-DD](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/transceiver-modules/400g-qsfp-dd-haul-coherent-module-ds.html).

## 10. Representative 800G modules

### 10.1 QSFP-DD800 twin-port FR4

![QSFP-DD800 module with two duplex-LC 400G FR4 ports](/posts/sfp-qsfp-fiber-400g-800g-optics/qsfpdd800-2x400g-fr4.png)

`QDD-2X400G-FR4` contains two independent 400G FR4 engines. Each engine multiplexes four 100G wavelengths onto its own duplex-LC pair, giving two 400G links and 800G aggregate electrical capacity.

### 10.2 OSFP DR8

![OSFP 800G DR8P module with integrated heat sink and MPO-16 front](/posts/sfp-qsfp-fiber-400g-800g-optics/osfp-800g-dr8p.jpg)

OSFP is physically larger and has more thermal headroom. The photographed `OSFP-800G-DR8P` uses an MPO-16 APC front for eight transmit and eight receive single-mode lanes.

| Example model | Fiber and connector | Reach | Organization and breakout |
| --- | --- | ---: | --- |
| `OSFP-800G-VR8` | MMF, dual MPO-12 APC | 30 m OM3 / 50 m OM4/5 | Eight parallel lanes; 2×400G, 4×200G, or 8×100G |
| `OSFP-800G-VR8P` | MMF, MPO-16 APC | 30/50 m | Same lane structure using one sixteen-fiber connector |
| `OSFP-800G-DR8` | SMF, dual MPO-12 APC | 500 m | Eight parallel 100G lanes |
| `OSFP-800G-DR8P` | SMF, MPO-16 APC | 500 m | Eight parallel 100G lanes in one MPO-16 |
| `OSFP-2X400G-FR4` | SMF, two duplex-LC UPC pairs | 2 km | Two independent 400G FR4 engines |
| `QDD-8X100G-FR` | SMF, dual MPO-12 APC | 2 km | Eight independent 100G FR1 lanes or two 400G groups |
| `QDD-2X400G-FR4` | SMF, two duplex-LC UPC pairs | 2 km | Two WDM 400G FR4 links |
| 800ZR | SMF, duplex LC, amplified DWDM | Up to about 120 km | One coherent 800G carrier for DCI |
| 800G ZR+ | SMF, duplex LC, amplified DWDM | Over 1,000 km in suitable designs | Coherent metro, regional, and long-haul operation |

Cisco lists typical OSFP client-optic power near 13–14 W and maximum values around 16–17 W for several 800G VR8/DR8 variants. Its QSFP-DD800 FR examples are specified at up to 17 W. These values are implementation-specific, but they explain why port power class, airflow direction, heat sinks, ambient temperature, and adjacent-port population rules must be checked. See the current [Cisco OSFP 800G data sheet](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/transceiver-modules/osfp-800g-transceiver-modules-ds.html) and [QSFP-DD800 data sheet](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/transceiver-modules/qsfp-dd800-transceiver-modules-ds.html).

OIF defines 800ZR for interoperable amplified 80–120 km DCI links. ZR+ extends the design into metro and regional/long-haul line systems. See the [OIF 800G coherent overview](https://www.oiforum.com/technical-work/hot-topics/800g-coherent/) and Cisco's [800G ZR/ZR+ QSFP-DD and OSFP documentation](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/transceiver-modules/qsfp-dd-osfp-800g-zr-zr-plus-coherent-optics.html).

## 11. Breakout is a property of the complete system

![400G DR4 and 800G DR8 breakout patterns](/posts/sfp-qsfp-fiber-400g-800g-optics/breakout-map.svg)

A breakout requires all three layers to cooperate:

1. The switch ASIC and operating system must support port partitioning.
2. The module must expose independent optical lanes or subports.
3. The cable must map the MPO fibers to the correct remote connectors.

Natural examples are:

```text
400G DR4 → 4 × 100G DR1
400G DR4 → 2 × 200G DR2

800G DR8 → 2 × 400G DR4
800G DR8 → 4 × 200G DR2
800G DR8 → 8 × 100G DR1
```

A twin-port `2X400G-FR4` module is also a breakout device, but the separation occurs inside the module and exits as two duplex-LC interfaces.

## 12. Optical budget and receiver limits

Distance is shorthand. The real engineering limit is the optical budget:

```text
available loss budget = minimum transmitter power − receiver sensitivity
```

Path loss must include:

- Fiber attenuation at the operating wavelength
- Connector and adapter loss
- Fusion or mechanical splices
- MPO fan-out and patch-panel loss
- Splitters or passive WDM devices
- Aging, repair, and engineering margin

The received signal must also remain below the receiver's maximum input power. Long-reach EX/ER/ZX/ZR optics are not automatically better on a short jumper; a high-power transmitter may overload the receiver and require an attenuator.

## 13. FEC, CMIS, and diagnostics

### 13.1 Forward error correction

PAM4 has three decision boundaries instead of the single boundary used by NRZ. Its optical and electrical eye openings are therefore smaller, making FEC a normal part of 100G-per-lane designs.

When troubleshooting, verify:

- RS-FEC or other FEC mode at both ends
- Pre-FEC BER and corrected-codeword counters
- Uncorrectable codewords
- Lane alignment and deskew
- Whether the optic performs FEC internally or relies on the host

### 13.2 CMIS management

Modern QSFP-DD and OSFP modules commonly use the Common Management Interface Specification. CMIS lets the host:

- Identify supported applications and lane mappings
- Select 800G, 2×400G, or other operating modes
- Read temperature, voltage, laser bias, and per-lane optical power
- Read thresholds, alarms, and versatile diagnostic monitoring data
- Report module state and firmware revision
- Upgrade module firmware where supported

A module can be mechanically correct and optically compatible yet remain unusable because the host does not support its CMIS application code or power class.

## 14. Practical selection table

| Requirement | Sensible first candidate |
| --- | --- |
| Same rack, minimum power and cost | Passive DAC |
| Nearby racks, easier routing | AOC |
| 30–50 m over multimode | VR4 or VR8 |
| Up to 100 m while reusing duplex MMF | Compatible BiDi model |
| Up to 500 m and breakout is important | DR4 or DR8 |
| Up to 2 km on duplex single-mode | FR4 or 2X400G-FR4 |
| Up to 10 km | LR4 |
| Four or eight independent 100G endpoints | Explicit 4X100G or 8X100G optic |
| 80–120 km DCI | ZR with an appropriate amplified design |
| Regional or long haul | ZR+ or ULH coherent optic and engineered DWDM line system |

## 15. Pre-purchase and troubleshooting checklist

Before ordering or installing a transceiver, verify:

1. **Cage:** SFP, QSFP, QSFP112, QSFP-DD, QSFP-DD800, or OSFP.
2. **Host rate:** Electrical lane count and per-lane baud rate.
3. **Application:** 400GE, 800GE, 2×400GE, or breakout mode.
4. **FEC:** Required mode on both hosts.
5. **Power and cooling:** Maximum module wattage, heat sink, airflow, and port-population restrictions.
6. **Optical standard:** DR4-to-DR4, FR4-to-FR4, or an explicitly supported asymmetric/breakout combination.
7. **Fiber:** OM3/OM4/OM5 or OS2/G.652.
8. **Connector:** LC, MPO-12, MPO-16, or dual MPO-12; APC/UPC; pins and polarity.
9. **Wavelength:** Especially for BiDi, CWDM, DWDM, ZR, and ZR+.
10. **Loss budget:** Include every connector, splice, panel, and margin.
11. **Platform qualification:** Switch model, line card, software version, and vendor coding policy.
12. **Diagnostics:** Tx/Rx power, temperature, voltage, FEC counters, alarms, and lane state.

## 16. Takeaways

1. **400G and 800G are usually not SFP form factors.** They are QSFP112, QSFP-DD/QSFP-DD800, or OSFP.
2. **Appearance is insufficient.** Similar housings can carry different speeds, fibers, wavelengths, FEC, and reach.
3. **DR means parallel and breakout-friendly; FR/LR means low fiber count through WDM; ZR/ZR+ means coherent DWDM.**
4. **400G modern arithmetic is four 100G PAM4 lanes; 800G is eight 100G PAM4 lanes.** Earlier 400G QSFP-DD commonly uses eight 50G lanes.
5. **Breakout is not guaranteed by lane count.** It requires a compatible host, module application, FEC mode, and cable map.
6. **Power and cooling are part of link design at 400G/800G.** A module that fits may exceed the port's thermal envelope.
7. **The label is only the beginning.** Always validate the platform compatibility matrix and the complete optical budget.

## Photo credits and reference scope

The SFP, DAC/AOC, QSFP100, QSFP112, QSFP-DD800, and OSFP product photographs are reproduced for identification and technical commentary from the linked Cisco data sheets: [1G SFP](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/gigabit-ethernet-gbic-sfp-modules/datasheet-c78-366584.html), [10G SFP+](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/transceiver-modules/data_sheet_c78-455693.html), [100G QSFP](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/transceiver-modules/datasheet-c78-736282.html), [400G QSFP112](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/400g-qsfp-transceiver-modules-ds.html), [QSFP-DD800](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/transceiver-modules/qsfp-dd800-transceiver-modules-ds.html), and [OSFP 800G](https://www.cisco.com/c/en/us/products/collateral/interfaces-modules/transceiver-modules/osfp-800g-transceiver-modules-ds.html).

The single-mode and multimode patch-cord photograph is by Christophe Finot, licensed CC BY-SA 3.0, via [Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Cordons_monomode-multimode_LC.JPG). The MPO end-face photograph is reproduced for identification and technical commentary from Fluke Networks' [Cleaning and Inspecting MPO/MTP Connectors](https://www.flukenetworks.com/knowledge-base/multifiber-pro/cleaning-and-inspecting-mpompt-connectors). Product identifiers are examples, not purchasing endorsements; current host support and specifications must be confirmed against the vendor's compatibility matrix.
