# Link Budget And Site Survey

How far the radios will really reach, and how to prove it on site before freezing the design. Bottom line: at the legal 16 dBm EIRP in a Philippine street, expect **sub-kilometre links at street level**. Antenna height is the cheapest fix, and a physical survey is mandatory.

## Key points

- Design against the **legal +16 dBm EIRP**, not the radio's +20 dBm capability ([[Philippines-Regulatory-Context]]).
- Urban path loss falls off at **n ≈ 3.8**, not 2. A ~132 dB budget buys roughly **0.3–1.5 km**, not 10 km.
- **Design targets:** slave → device links ≤500 m; master → slave links ≤1.5 km with elevated antennas.
- **Height beats gain.** Getting slaves above the luminaire/tree/jeepney line is the cheapest dB available.
- SF11/SF12 are unusable here, so a link that doesn't close at SF10 needs height or placement, not more SF.
- **Survey with the real radios at real heights**, soak 24 h+, and demand ≥15 dB margin (25 dB in vegetation).

## The worksheet

```
EIRP_tx     = Ptx + G_tx − L_cable_tx
Prx         = EIRP_tx − PathLoss + G_rx − L_cable_rx
Link margin = Prx − Sensitivity          ← must exceed the fade margin
```

Worked example, master → slave at 916 MHz, SF9/125 kHz (sensitivity −129 dBm), 3 dBi antennas, −0.6 dB cable each end:

| Case | EIRP | Max tolerable path loss | After 15 dB fade margin |
|---|---|---|---|
| **Legal: +16 dBm EIRP** (PA backed off in firmware) | +16 dBm | 147.4 dB | **132 dB** ← design figure |
| Silicon capability, +20 dBm PA (for comparison only) | +22.4 dBm | 153.8 dB | 139 dB |

Antenna gain counts toward EIRP, so with a higher-gain antenna the PA output must come down further.

## Path loss: free space doesn't apply

Free-space loss at 916 MHz is 91.7 dB at 1 km and 97.7 dB at 2 km. That is wildly optimistic for a street canyon.

**COST-231 Hata** (valid 500–2000 MHz):

```
PL = 46.3 + 33.9·log10(f_MHz) − 13.82·log10(h_b) − a(h_r)
     + [44.9 − 6.55·log10(h_b)]·log10(d_km) + Cm       (Cm = 3 dB urban, 0 dB suburban)
```

For h_b = 10 m, h_r = 6 m, 916 MHz, urban: the distance term is ~**38 dB/decade**, a path-loss exponent of **n ≈ 3.8** against 2.0 in free space.

| Environment | Practical SF9 range, 16 dBm EIRP, pole-height antennas |
|---|---|
| Clear line of sight, both ends ≥10 m, no foliage | 2–5 km |
| Suburban, partial line of sight, low buildings | 0.8–2 km |
| Dense urban corridor / heavy foliage | 0.3–0.8 km |

⚠ Model error is large: one measurement campaign found Okumura-Hata **14 dB below** measurement at 100 m and **7 dB above** at 5 km in complex urban terrain. **±14 dB of model error is comparable to the entire fade margin.** No desk calculation replaces a survey.

## Height beats gain

Height enters the model as `−13.82·log10(h_b)`. Raising the gateway from 6 m to 12 m buys **4.2 dB**, plus a large unmodelled improvement in clutter clearance. That's cheaper than +1 SF (2× airtime for 2.5 dB). Two extra metres of mast typically beat 3 dB of antenna gain at street level.

## Fresnel zone: why every link here is obstructed

First Fresnel radius at midpoint: `r = 17.32 × sqrt(d_km / (4 · f_GHz))` m. At 916 MHz that's **9.0 m at 1 km** and 6.4 m at 500 m. Clearance should be ≥60% of that (80% preferred), so a 1 km link needs **5.4 m of clearance above the roadway at mid-span**.

A street-light-height antenna on a tree-lined street can't meet this. That is why n ≈ 3.8 and why measured ranges collapse to under a kilometre. Earth bulge is negligible at these distances.

## Antennas

- **Vertical polarisation, matched at both ends.** Cross-polarisation costs ~20 dB.
- Mount clear of the metal pole, at least λ/4 (~8 cm) and preferably 0.5 m above the luminaire arm, or the pole reflects and detunes the antenna.
- 3–5 dBi omni fibreglass collinear, with N-type (not SMA) connectors on outdoor bulkheads.
- **Each radio of a redundant pair gets its own antenna and coax.** Separating them vertically also improves in-box isolation ([[Dual-Radio-Redundancy-And-Coexistence]]).
- Coax loss, weatherproofing and lightning arrestors: [[Enclosure-Power-And-Surge]].

## Site survey protocol

1. **Survey with the actual radios (SX1276), at the actual heights, with the actual antennas.** Simulations lie.
2. Record RSSI/SNR/PER at every planned slave location and a sample of pole positions, at the working SF and at SF10. Record the **noise floor separately**, with all transmitters off.
3. **Soak each link 24 h+** to catch diurnal interference and thermal effects. Take at least one measurement in rain.
4. Acceptance: **≥15 dB margin at the working SF**. Design target ~25 dB in vegetated tropical terrain, because **foliage attenuation is seasonal**: a link commissioned in the dry season at 8 dB margin will fail in the wet season.
5. **Scan 915–918 MHz for existing occupancy** before fixing the channel plan. Only 3 MHz is available, and it's shared.
6. Document antenna type, height, polarisation, cable length and loss, and photograph each installation's orientation. **Connector water ingress is the most common slow-degradation failure in tropical installs**; it shows up as "the link got worse over six months".

⚠ No 915 MHz street-level propagation measurements specific to Philippine urban morphology were found. All range figures here are model-derived.

## Related

- [[LoRa-PHY-Fundamentals]]
- [[LoRa-Airtime-Budget]]
- [[Commissioning-And-Field-Swap]]
- [[Gateway-Observability]]

## Sources

- Research synthesis, 2026-09-09. COST-231: https://www.rfwireless-world.com/terminology/okumura-hata-model-cost-231-extension · measurement error: https://arxiv.org/pdf/2510.23152 · tropical/forest path loss at 923 MHz: https://pmc.ncbi.nlm.nih.gov/articles/PMC13210814/

## History

- 2026-10-14: The legal +16 dBm EIRP is now the design case (Philippine profile adopted); SX1276 radios; SF escalation capped at SF10; separate antennas for redundant radios.

Back to [[Index]]
