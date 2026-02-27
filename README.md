# I Built a PWA to Find Sunshine Near Me — Here's How It Went

*A weekend project that turned into a surprisingly deep rabbit hole of weather APIs, sun scoring algorithms, and the question: "why does my phone think it's night when the sun is blazing outside?"*

---

There's a very specific frustration that hits when you're sitting indoors in Munich on a grey afternoon and you're not sure if it's worth getting out — is it just cloudy here, or everywhere? Is there sun 20 km south? Should I drive?

I couldn't find an app that answered this simply. Every weather app gives you too much: hourly precipitation probability, wind direction, UV index, dew point. I don't need any of that. I need one answer: **is there sun near me right now, and if not, where's the closest sunny spot?**

So I built [SunChaser](https://sunchaser.pages.dev).

---

## What It Does

SunChaser is a progressive web app (PWA) that does three things:

1. **Shows your current sun status** — a single, honest answer: Sun now / Partly sunny / Mostly cloudy / Overcast / Night time.
2. **Shows today's hourly sun forecast** — a scrollable strip so you can see when things improve.
3. **Finds the nearest sunny spot within 50 km** — it samples 24 points around you (8 compass directions × 3 distances: 15, 30, and 50 km) and surfaces the sunniest ones.

No sign-up. No API key. No ads. Works offline. Installable as a home screen app on both iOS and Android.

---

## The Tech Stack (Deliberately Boring)

I made a deliberate choice to keep the tech stack as minimal as possible: **one HTML file, zero dependencies, zero build steps**.

- **Weather data**: [Open-Meteo](https://open-meteo.com) — free, no API key, genuinely excellent API
- **Geocoding**: [Nominatim](https://nominatim.openstreetmap.org) (OpenStreetMap) — free reverse geocoding
- **Hosting**: [Cloudflare Pages](https://pages.cloudflare.com) — free, instant deploys from GitHub, automatic HTTPS
- **Fonts**: Syne (display) + Instrument Sans (body) via Google Fonts
- **Offline**: A service worker registered via an inline Blob URL — a neat trick that means no separate `sw.js` file is needed

The whole thing is ~60 KB uncompressed. There's no bundler, no npm, no framework. Just HTML, CSS, and vanilla JS.

---

## The Sun Score Algorithm

The core of the app is a simple scoring function that translates raw weather data into a 0–100 "sun score":

```js
function sunScore(cloudCover, precip = 0) {
  let score = 100 - cloudCover;  // cloud cover is 0–100%
  if (precip > 40) score -= 30;  // heavy rain penalty
  return Math.max(0, Math.min(100, score));
}
```

That's it. Cloud cover drives almost everything. A heavy rain penalty kicks in when precipitation probability exceeds 40%. The score then maps to a status:

- **75–100**: ☀️ Sun now
- **50–74**: ⛅ Partly sunny
- **25–49**: 🌥️ Mostly cloudy
- **0–24**: ☁️ Overcast (or 🌙 Night time if `is_day = 0`)

---

## Finding Nearby Sun: The Multi-Ring Approach

The first version of the nearby search placed exactly 8 points in 8 compass directions, all at exactly 50 km. This worked but produced a silly result: every card said "~50 km away," even if sun was available 15 km north.

The fix was to sample **three concentric rings**: 15 km, 30 km, and 50 km — 24 total points. Then for each compass direction, keep the closest point that scores best. This means if there's sun 15 km north, you see "N · ~15 km away" instead of being sent 50 km unnecessarily.

Open-Meteo supports **batch requests** — you can pass multiple lat/lng pairs in a single API call and get an array back. So all 24 points are fetched in one request. The whole scan takes about 1–2 seconds.

---

## The Bugs That Taught Me Things

### `is_day` is not always right

Open-Meteo returns an `is_day` flag with current conditions. Early on I used this as the primary check — if `is_day = 0`, show Night time. Simple.

But users started seeing "Night time" with a sun score of 100% and 0% cloud cover. Turns out `is_day` can lag around civil twilight — the sun is technically above the horizon, cloud cover is zero, but the flag says night.

The lesson: **trust measured data over derived flags**. The sun score is calculated from actual cloud cover. `is_day` is a metadata flag that can be wrong at the edges. The fix was to check score first, `is_day` only as a tiebreaker for low scores.

*(Edit: then I reverted this because it caused a different problem — "Sun now" showing at actual midnight. The real fix was keeping `is_day` correct and instead making the night screen more useful, with a "Best sun tomorrow" chip showing when to come back.)*

### Scroll fade masks and dynamic backgrounds

The hourly forecast strip needed edge fades to hint it was scrollable. My first approach: absolutely-positioned `::before`/`::after` pseudo-elements with a `rgba(0,0,0,0.5)` gradient overlay.

This looked terrible. The dark overlay didn't match the warm orange sunny gradient at all.

The correct solution: **CSS `mask-image`**. Instead of painting a colour on top, you apply a transparency mask to the element itself:

```css
.hourly-outer {
  mask-image: linear-gradient(
    to right,
    transparent 0%,
    black 48px,
    black calc(100% - 48px),
    transparent 100%
  );
}
```

This fades the content to truly transparent, which works on any background, any gradient, any colour. JavaScript toggles `.at-start` and `.at-end` classes to remove the relevant side mask when you've scrolled to an edge.

---

## What I Like About It

**The gradient transitions.** The background smoothly transitions between warm orange (sunny), blue (partly cloudy), dark slate (overcast), and deep navy (night) as conditions change. A 1.3-second CSS transition on `body.background` makes it feel like the sky is actually changing.

**No empty states without meaning.** At night, instead of just showing a moon and stopping, the app shows:
- When the sun rises
- Tomorrow's best sun window ("Best sun tomorrow: 9am – 1pm")
- A Find Sun Nearby button that, if everything within 50 km is dark, shows a quote and explains why rather than a list of moon icons

**It's a real PWA.** Full offline support via service worker, installable on iOS and Android home screens, theme-color adapts to the current conditions, proper Open Graph / Twitter card / JSON-LD structured data for SEO.

---

## What I'd Do Differently

**More sampling points.** 24 points is a good start but weather can vary significantly within 15 km in mountainous terrain (hello, Alps). A hexagonal grid or Voronoi-based sampling would give better coverage.

**Push notifications.** "Sun is breaking through in 30 minutes" would be genuinely useful. This requires a backend, which I deliberately avoided, but it's the natural next step.

**The `og-image.png` placeholder.** The SEO meta tags reference a social share image that doesn't exist yet. Every developer's eternal TODO.

---

## Try It

[**sunchaser.pages.dev**](https://sunchaser.pages.dev)

Works best installed as a PWA — add to your home screen and it opens full-screen with no browser chrome. On iOS: Share → Add to Home Screen. On Android: the install banner appears automatically after a few seconds.

The source is a single `index.html` file. If you want to see how any of it works, just View Source.

---

*Built in Munich, where checking whether to go outside is a genuine daily need from October through April.*