# Figma Prototype Zoom

Open any Figma prototype as a clean, full‑screen presentation that behaves like a
real website — pinned header, responsive layout, and crisp zoom.

**Open it → https://loyegor.github.io/figma-zoom/**

## How to use

1. In Figma, on a prototype: **Share → Copy link**.
2. Open the app, paste the link, press **Load**.
3. Present it: zoom (100–140%), go fullscreen, and click through the prototype
   like a normal website. Use **Link** (bottom dock) to swap in another prototype.

The link you paste is **auto‑corrected** (loaded as a responsive
`embed.figma.com` prototype with the right scaling params) and **saved in your
browser** — nothing is uploaded anywhere, and the next visit reopens your last
prototype.

## Notes

- A single static `index.html` — no build, no dependencies, no tracking.
- Works with any Figma prototype whose share link the viewer can open.
- Zoom re‑flows the prototype to a narrower viewport (it doesn't just magnify the
  bitmap), so content stays crisp and nothing spills off the edges.
