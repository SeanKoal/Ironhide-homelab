# Hardware Build

## The Concept

Ironhide started as an idea to build a homelab server using mostly thrifted and secondhand parts, keeping cost low while learning real virtualization and infrastructure skills.

## Parts List

| Component | Model | Source |
|-----------|-------|--------|
| Case | Fractal Torrent | Thrifted |
| CPU | AMD Ryzen 5 3600 | eBay, refurbished |
| Motherboard | MSI B450 Gaming Plus | eBay, refurbished |
| RAM | 32GB DDR4 3200MHz | eBay, refurbished |
| OS Drive | WD Blue SN570 250GB NVMe | New |
| VM Storage | Samsung 860 EVO 1TB SSD | eBay, refurbished |
| NAS Storage | Seagate IronWolf 2TB HDD | eBay, refurbished |
| PSU | Corsair CX550 | New |

## Build Notes

- Nearly the entire build was sourced secondhand through eBay (refurbished listings), with the case thrifted separately — only the PSU and OS drive were bought new, prioritizing reliability for the two components where used parts carry the most risk
- The IronWolf 2TB was intentionally left unformatted during initial setup, reserved for direct passthrough to a future TrueNAS VM
- RAM initially ran at 2133MHz until XMP was enabled in BIOS, bringing it up to the rated 3200MHz

## Naming Convention

Every device in the homelab follows a Transformers naming theme:

- **Ironhide** — the homelab server itself (the muscle)
- **Magnus** — daily driver desktop (the commander)
- **Soundwave** — portable ThinkPad for remote access
- **Hound** — repurposed laptop, planned as an always-on Wake-on-LAN agent
