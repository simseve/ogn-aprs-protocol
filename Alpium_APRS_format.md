---
title: OGNALP APRS message specification
description: APRS messages sent by the Alpium backend to OGN
date: 2026-09-19
version: 1.0.0
---

# OGNALP APRS message specification

Alpium (alpium.io) is a live-tracking and safety platform for free-flight
pilots, mostly paragliding and hang gliding. Positions come from the Alpium
app on the pilot's phone or from a GPS tracker connected to the Alpium
backend. The backend validates, filters and rate-controls those positions and
forwards them to OGN over one central persistent APRS-IS connection. Phones
and trackers never open connections to OGN themselves.

## 1 Identifiers and transport

Alpium uses the following fixed identifiers:

* APRS-IS login: `ALPIUM`
* TOCALL: `OGNALP`
* APRS source callsign: `ALP` followed by a six-hex-digit address
* software identifier in the APRS-IS login: `alpium-ogn-feed 1.0`

The backend login has this form:

```
user ALPIUM pass <passcode> vers alpium-ogn-feed 1.0
```

`OGNALP` identifies version 1 of this format. Alpium does not currently
append a version suffix to the TOCALL.

## 2 Position message

Alpium emits only complete TNC2 position lines in this form:

```
ALP<address>>OGNALP,qAS,ALPIUM:/<timestamp>h<latitude>/<longitude>'<course>/<speed>/A=<altitude> !W<precision>! id<identifier> <climb>fpm
```

Parameters:

* **address** — the six-hex-digit Alpium address described in section 3.
* **timestamp** — UTC time of the FIX in `HHMMSS` form, never the time of
  transmission.
* **latitude** — uncompressed APRS latitude in `DDMM.mmN` or `DDMM.mmS` form.
* **longitude** — uncompressed APRS longitude in `DDDMM.mmE` or `DDDMM.mmW`
  form. Alpium uses the `/` symbol table and the `'` aircraft symbol.
* **course** — three-digit ground track in degrees.
* **speed** — three-digit ground speed in knots. As specified for
  OGN-flavoured APRS, `000/000` indicates that neither value is available;
  about a fifth of the fixes from phones carry no ground speed or course.
* **altitude** — GNSS altitude in feet above mean sea level, six unsigned
  digits. Positions without a GNSS altitude are not transmitted at all.
  Barometric altitude is never substituted.
* **precision** — the `!W..!` extension carrying the third decimal digit of
  the latitude and longitude minutes.
* **identifier** — the OGN detail byte followed by the six-hex-digit Alpium
  address, as described in section 3.
* **climb** — signed vertical speed in feet per minute, computed between two
  consecutive source fixes, which are about one second apart.

Alpium does not send radio signal measurements, turn rate, satellite status,
device status, names, registrations or proprietary extensions. Absent optional
source values are omitted rather than synthesized.

## 3 Aircraft type and identity

The low two bits of the detail byte contain the OGN address type. Alpium uses
address type `0` (random): Alpium is not an ICAO, FLARM or OGN-tracker
registry, and claiming one of those types would collide with real addresses
in them.

The OGN aircraft type is taken from the sport the pilot is flying, as
recorded by the platform:

| Sport | OGN aircraft type | detail byte |
|---|---|---|
| paragliding, speed flying | `7` paraglider | `1C` |
| hang gliding | `6` hang glider | `18` |
| gliding | `1` glider | `04` |
| powered free flight | `8` powered aircraft | `20` |
| any other air sport | `0` unknown | `00` |

No stealth flag and no no-track flag are set. A position whose sport cannot
be established is not transmitted, so an unknown aircraft type reaches the
network only for an air sport outside the table above.

The 24-bit address is allocated sequentially from `000001` and is stable for
the life of the pilot's Alpium account. It is deliberately per pilot rather
than per device: an Alpium device identifier is a phone identifier that
changes when the app is reinstalled, and a pilot carrying both a phone and a
tracker is one aircraft, not two. Alpium transmits at most one device per
pilot at any moment. The address may optionally be registered in the OGN
Device Database; registration is not required for transmission.

## 4 Privacy and consent

A position is transmitted only while the pilot has live tracking switched on
in the Alpium app, which is an explicit action taken per flight. The
transmission is described to pilots in the Alpium privacy policy, which names
OGN and its redistribution.

Positions recorded for a company through Alpium Business, the platform's
workforce product, are never transmitted, and neither are simulated positions
used for demonstrations. Positions that Alpium itself received from OGN,
APRS-IS or another live-tracking provider are never re-transmitted.

Neither the address nor the message contains a name, email address, phone
number, account identifier or device identifier.

## 5 Rate control and freshness

The central backend implements the following transmission policy per
aircraft:

* the first valid position of a flight is transmitted immediately;
* after that, a position is transmitted when at least **20 seconds** have
  passed since the last transmitted position;
* between 10 and 20 seconds, a position is transmitted only when a consumer
  extrapolating from the last transmitted position, along its course at its
  ground speed and vertical speed, would now be wrong by more than **200
  metres horizontally** or **50 metres vertically**. This is what covers a
  rapidly changing flight path, such as a high sink rate or a sharp turn;
* no position is ever transmitted less than **10 seconds** after the previous
  one, whatever the divergence, so a continuously manoeuvring aircraft cannot
  become a high-rate source;
* positions older than 30 seconds, positions in the future, positions without
  a GNSS altitude and positions that are not newer than the last transmitted
  one are rejected;
* nothing is queued. A position that cannot be written when it arrives is
  discarded rather than transmitted late, so a reconnection cannot replay a
  historical track.

Measured against a recorded 75 km cross-country paragliding flight of 6679
source fixes over 153 minutes, this policy transmits 408 positions, one every
22.5 seconds on average, of which 54 are the extra messages permitted by the
divergence rule. While Alpium is silent, a consumer extrapolating from the
last transmitted position is a median of 7 metres and at most 198 metres from
the true position.

The backend reconnects with exponential backoff starting at five seconds,
doubling to a maximum of 120 seconds.

## 6 Examples

Both examples below are produced by the encoder itself and decode with
`python-ogn-client`. A paraglider climbing in a thermal:

```
ALP00001A>OGNALP,qAS,ALPIUM:/101530h4550.36N/00902.04E'090/015/A=003281 !W46! id1C00001A +059fpm
```

A hang glider whose phone reports no ground speed or course:

```
ALP0000B3>OGNALP,qAS,ALPIUM:/101612h4552.09N/00905.77E'000/000/A=004120 !W82! id180000B3 -250fpm
```

## 7 Related documents

* [OGN APRS messages](aprsmsgs.txt)
* [APRS Protocol Reference, Protocol Version 1.0](http://www.aprs.org/doc/APRS101.PDF)
* [Naviter APRS message specification](Naviter_APRS_format.md)
* [SkyBase APRS message specification](SkyBase_APRS_format.md)
