# Functional & Technical Requirements

## 1 Project Goal

Enhance the existing **benches‑nearby** single‑page web app so a user can search for *one of four* OpenStreetMap (OSM) point‑of‑interest (POI) categories around any spot on the map:

* **Bänke** (benches)
* **Altglas‑Container** (glass‑only recycling containers)
* **Defibrillatoren** (AEDs)
* **Briefkästen** (post boxes)

Only one category is active at a time; the user chooses it via a modal overlay shown on first load. The map itself remains fully client‑side—no backend services other than the public Overpass API and external asset CDNs.

---

## 2 User‑level Requirements

| #      | Scenario                       | Expected Result                                                                                                                                                      |
| ------ | ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **U1** | Load page                      | Map fills the viewport; an opaque vertical modal lists the four categories as icon + label buttons.                                                                  |
| **U2** | Choose category                | Modal disappears; browser asks for geolocation permission.                                                                                                           |
| **U3** | Accept geolocation             | Map recentres to user position; **automatic** first POI search fires.                                                                                                |
| **U4** | Deny / unavailable geolocation | Modal closes; unobtrusive instruction appears: *„Standort nicht verfügbar – bitte in die Karte tippen, um hier zu suchen“*. User can still click anywhere to search. |
| **U5** | Click on map                   | Previous markers are cleared ➜ new Overpass request ➜ new markers shown.                                                                                             |
| **U6** | Click marker                   | Leaflet popup opens displaying either the OSM `name` or a default label (see §5.2).                                                                                  |

---

## 3 OSM Query Design

### 3.1 Tag Filters

| Category          | Overpass filter                                        | Default radius |
| ----------------- | ------------------------------------------------------ | -------------- |
| Bänke             | `node["amenity"="bench"]`                              | **1500 m**     |
| Altglas‑Container | `node["amenity"="recycling"]["recycling:glass"="yes"]` | **200 m**      |
| Defibrillatoren   | `node["emergency"="defibrillator"]`                    | **500 m**      |
| Briefkästen       | `node["amenity"="post_box"]`                           | **300 m**      |

`around:<radius>,<lat>,<lon>` is appended to each query.

### 3.2 Query Envelope

```txt
[out:json][timeout:25];
<filter line from table>
out body;
```

No result‑count limit or clustering; the app must gracefully render **all** returned nodes.

---

## 4 UI / UX Specification

### 4.1 Modal Overlay

* **Layout**: full‑viewport *fixed* `<div>` with a semi‑opaque backdrop; centered **vertical button list** (`flex-col` Tailwind).
* **Buttons**: left‑aligned 32 px SVG icon + right‑aligned text label, `gap-3 py-4 px-6 rounded-lg hover:bg-gray-100`.
* **Tailwind**: delivered via **Play CDN** script tag.
* **Accessibility**: *none added*—no focus‑trap or ARIA roles (per stakeholder decision).

### 4.2 Map Interaction

* Map initialises immediately (`setView([0,0],13)`), tile layer from OpenStreetMap standard.
* After category choice ➜ on successful geolocation: `map.setView([lat,lng],15)` plus small `L.circle` *“You are here”*.
* On every `map.on('click')`:

  * `layerGroup.clearLayers()` removes previous POIs.
  * New Overpass request fires with chosen radius.

### 4.3 Pop‑ups

* Primary text = OSM `name` tag.
* Fallback default labels: **Bank / Altglas‑Container / Defibrillator / Briefkasten**.

### 4.4 Visual Assets

| Category          | Icon URL (remote)                                                    |
| ----------------- | -------------------------------------------------------------------- |
| Bänke             | `https://wiki.openstreetmap.org/w/images/6/60/Amenity_bench.svg`     |
| Altglas‑Container | `https://wiki.openstreetmap.org/w/images/1/16/Recycling-16.svg`      |
| Defibrillatoren   | `https://wiki.openstreetmap.org/w/images/9/9f/Emergency_aed_svg.svg` |
| Briefkästen       | `https://wiki.openstreetmap.org/w/images/d/df/Post_box-12.svg`       |

All icons loaded via Leaflet `L.icon` with `iconSize:[32,32]`.

### 4.5 Loading Indicators

* **None** (per requirement). UI stays static until markers appear.

### 4.6 Error Feedback

* Overpass fetch failures are logged via `console.error()` only; no UI messaging.

---

## 5 Technical Architecture

### 5.1 File Structure

```
index.html   ← single self‑contained file
```

Everything (HTML, inline `<script>`, inline `<style>`) remains inside `index.html`.

### 5.2 External Libraries / CDNs

| Purpose               | Source                                                |
| --------------------- | ----------------------------------------------------- |
| Leaflet v1.x JS & CSS | `https://unpkg.com/leaflet/...`                       |
| Tailwind Play CDN     | `https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4` |
| Icons                 | OSM‑Wiki URLs (see §4.4)                              |

No build pipeline, package.json, or bundler is introduced.

### 5.3 Data Flow Diagram

```
User ► CategoryModal ► (choose) ► Geolocation API
      │                           │
      └─ Map click ───────────────┘
                           ▼
               Overpass API (POST)
                           ▼
                     JSON result
                           ▼
               Marker creation in Leaflet
```

### 5.4 Performance Notes

* No artificial Debounce/Back‑off; rely on user pacing and Overpass limits.
* Accept potential slowdowns with very high marker counts.

---

## 6 Error‑Handling Strategy

| Condition                      | Handling                                              |
| ------------------------------ | ----------------------------------------------------- |
| Geolocation denied             | Display inline hint encouraging manual click (§2 U4). |
| Overpass error / 429 / timeout | Log to console, no user alert.                        |
| Network offline                | Leaflet still loads; POI fetch will fail and log.     |

---

## 7 Testing Plan

### 7.1 Unit‑like (manual) checks

1. **Category Modal** – appears on every hard reload; disappears on button click.
2. **Geolocation Accept/Deny** – verify auto‑search vs manual‑click path.
3. **Radius Accuracy** – use dev console to confirm `around:` values match table §3.1.
4. **Marker Clearing** – verify previous POIs vanish before new ones render.
5. **Fallback Labels** – choose map spots where OSM nodes lack `name` tags.
6. **No‑Limit Stress** – test dense city centre (e.g. Berlin Alexanderplatz) for performance.

### 7.2 Cross‑browser Matrix

| Browser | Desktop OS | Mobile OS   |
| ------- | ---------- | ----------- |
| Chrome  | Win/Mac    | Android 11+ |
| Firefox | Win/Mac    | —           |
| Safari  | macOS      | iOS 15+     |

Geolocation and HTTPS restrictions should be tested on each.

### 7.3 Accessibility Smoke‑test

Although no focus‑trap/ARIA is implemented, verify that keyboard selection via **Tab → Enter** still activates category buttons.

---

## 8 Non‑functional Requirements

* **HTTPS only** – mandatory for Geolocation API.
* **Open‑source license compliance** – all external assets are CC‑0 or Public Domain.
* **No server‑side state** – purely static hosting (GitHub Pages, Netlify, etc.).

---

## 9 Future‑proofing & Known Limitations

* **Rate limiting** – heavy/fast clicks may trigger Overpass 429; current spec accepts this risk.
* **Accessibility** – minimal; upgrading to full ARIA & focus‑trap is recommended for WCAG compliance.
* **Scaling** – Consider MarkerCluster or result limits if user performance feedback is poor.