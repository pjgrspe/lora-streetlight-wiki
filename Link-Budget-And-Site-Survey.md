# Link Budget And Site Survey

## The worksheet

```
EIRP_tx     = Ptx + G_tx − L_cable_tx
Prx         = EIRP_tx − PathLoss + G_rx − L_cable_rx
Link margin = Prx − Sensitivity          ← must exceed the fade margin
```

Worked, master → slave gateway at 916 MHz, SF9/125 kHz:

- Ptx +20 dBm, 3 dBi antenna, 1 m RG-316 (−0.6 dB) → EIRP ≈ **+22.4 dBm**
- Sensitivity SF9/125 kHz = **−129 dBm**
- Rx: 3 dBi, −0.6 dB → **max tolerable path loss ≈ 153.8 dB**
- Reserve **15 dB fade margin** → **budget ≈ 139 dB**

But if regulation caps EIRP at **+16 dBm** (the AS923 default, see [[Philippines-Regulatory-Context]]), recompute: 147.4 dB tolerable, **132 dB after fade margin**. Design against the legal figure, not the silicon's.

## Path loss — free space is a fantasy here

FSPL at 916 MHz: 91.7 dB at 1 km, 97.7 dB at 2 km. **Wildly optimistic for a street canyon.**

**COST-231 Hata** (valid 500–2000 MHz, so applicable):

```
PL = 46.3 + 33.9·log10(f_MHz) − 13.82·log10(h_b) − a(h_r)
     + [44.9 − 6.55·log10(h_b)]·log10(d_km) + Cm       (Cm = 3 dB urban, 0 dB suburban)
```

For h_b = 10 m, h_r = 6 m, 916 MHz, urban: the distance term is ~**38 dB/decade**, i.e. path-loss exponent **n ≈ 3.8** versus 2.0 in free space.

**Practical consequence: a 139 dB budget buys roughly 0.6–1.5 km in a Philippine urban corridor** — not the 10 km marketing figure.

| Environment | Practical SF9 range, 16 dBm EIRP, pole-height antennas |
|---|---|
| Clear LOS, both ends ≥10 m, no foliage | 2–5 km |
| Suburban, partial LOS, low buildings | 0.8–2 km |
| Dense urban corridor / heavy foliage | 0.3–0.8 km |

**Design target: slave-to-device links ≤500 m, master-to-slave links ≤1.5 km with elevated antennas.** Then you have real margin.

⚠ Model error is large: a recent measurement campaign found Okumura-Hata **14 dB below** measurement at 100 m and **7 dB above** at 5 km in complex urban terrain. **±14 dB of model error is comparable to your entire fade margin.** No desk calculation substitutes for a survey.

## Height beats gain

Height appears in the model as `−13.82·log10(h_b)`. Going 6 m → 12 m on the gateway buys **4.2 dB** plus a large unmodelled improvement in clutter clearance.

> **Raising the slave gateways above the luminaire / tree / jeepney line is the cheapest dB you will ever buy** — cheaper than +1 SF, which costs 2x airtime for 2.5 dB. Two metres of extra mast typically beats 3 dB of antenna gain at street level.

## Fresnel zone — why you are always NLOS here

First Fresnel radius at midpoint: `r = 17.32 × sqrt(d_km / (4 · f_GHz))` metres. At 916 MHz: **9.0 m at 1 km**, 6.4 m at 500 m. Clearance criterion is ≥60% of F1 (80% preferred), so a 1 km link needs **5.4 m of clearance above the roadway at midspan**.

**A street-light-height antenna on a tree-lined street physically cannot meet this.** You will operate deep in obstructed propagation — which is exactly why n ≈ 3.8 and why measured ranges collapse to sub-km. Earth bulge is negligible at these distances.

## Antennas

- **Vertical polarisation, matched at both ends.** Cross-polarisation costs ~20 dB.
- Mount clear of the metal pole — at least λ/4 (~8 cm), preferably 0.5 m above the luminaire arm — or the pole becomes a reflector and detunes the antenna.
- 3–5 dBi omni fibreglass collinear, N-type (not SMA) for outdoor bulkheads.
- See [[Enclosure-Power-And-Surge]] for coax loss, weatherproofing and lightning arrestors.

## Site survey protocol

1. **Survey with the actual radios, at the actual heights, with the actual antennas.** Simulations lie.
2. Record RSSI/SNR/PER at every planned slave location at both the working SF and the fallback SF — and record the **noise floor separately**, all transmitters off.
3. **Soak each link 24 h+.** Catch diurnal interference and thermal effects. Take at least one measurement in rain.
4. Acceptance: **≥15 dB margin at the working SF**; design target ~25 dB in vegetated tropical terrain, because **foliage attenuation is seasonal** — a link commissioned in dry season at 8 dB margin will fail in the wet season.
5. **Scan for existing 915–918 MHz occupancy.** Only 3 MHz is available and it is shared.
6. Document antenna type, height, polarisation, cable length/loss, and orientation photographs. **Connector water ingress is the most common slow-degradation failure in tropical installs** — it presents as "the link got worse over six months".

⚠ No 915 MHz street-level propagation measurements specific to Philippine urban morphology were found. All range figures here are model-derived. **The survey is mandatory, not optional.**

## Related

- [[LoRa-PHY-Fundamentals]]
- [[Commissioning-And-Field-Swap]]
- [[Gateway-Observability]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. COST-231: https://www.rfwireless-world.com/terminology/okumura-hata-model-cost-231-extension · measurement error: https://arxiv.org/pdf/2510.23152 · tropical/forest path loss at 923 MHz: https://pmc.ncbi.nlm.nih.gov/articles/PMC13210814/
