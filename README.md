# 45nm CTLE for a 12.5 Gb/s SerDes Link

A transistor-level continuous-time linear equalizer (CTLE) designed, simulated, and validated with real industry backplane channel data in Cadence Virtuoso, 45nm technology.

Methodology primarily follows Ankit Jain's MS thesis (*Equalization in Continuous and Discrete Time for High Speed Links Using 65 nm Technology*, UIUC, 2016), cross-referenced against Sam Palermo's SerDes lecture material (Texas A&M), a Gondi/Razavi (JSSC 2007) CTLE topology reference, and Poonam Agale's low-voltage CTLE thesis (SJSU, 2014, also 45nm).

**Status:** CTLE design complete and verified. DFE design in progress (see Section 9).


<p align="center">
  <img src="./Plots/Second%20Phase%20%28Double%20Stage%29/CTLE_Final_Schematic.png" width="600" alt="Double Stage Final Schematic">
</p>
<p align="center">
  <em>Two-stage Continuous Time Linear Equalizer Schematic, a fascinating melting pot of electronics</em>
</p>


---

## 1. Specifications

| Parameter | Value |
|---|---|
| Data rate | 12.5 Gb/s |
| Nyquist frequency | 6.25 GHz |
| Process | 45nm CMOS |
| Topology | Two-stage cascaded, source + capacitive degenerated differential pair |
| Simulator | Cadence Virtuoso / Spectre (ADE) |

---

## 2. Channel Characterization

Two real, industry-sourced channel models were used, both from a public IEEE P802.3ck contribution (Nathan Tracy & Arturo Pachon, TE Connectivity, Jan 2019):

**Notes:**

- The first backplane taken was a 4-inch one, but since the loss was too small, I chose to go for a 12 inch backplane.
- Only the traditional backplane was **initially used**, but because of certain problems faced later (eye diagram), **a switch was made to the orthogonal backplane**, allowing for a comparative study.

| Channel | File | Connector type | Loss @ 6.25GHz | Avg. ripple depth | Deepest notch |
|---|---|---|---|---|---|
| **Traditional** | `Std_BP_12inch_Meg7_Thru_B56.s4p` | Non-backdrilled | −9.73 dB | **7.83 dB** | −42.18 dB |
| **Orthogonal (DPO)** | `DPO_IL_28dB_DPO_12in_Meg7_THRU.s4p` | Direct-Plug Orthogonal | −10.15 dB | **0.03 dB** | −18.16 dB |



Both are 12-inch, Megtron-7 dielectric, same length and material, differing only in connector/via architecture. This pairing became the project's central comparison.

**Differential insertion loss (SDD21) extraction:** parsed directly from Touchstone `.s4p` files in Python (numpy), converting the 4-port single-ended S-parameters to mixed-mode differential insertion loss.
- Traditional channel: TX = ports {1,2}, RX = ports {3,4} → `SDD21 = 0.5×(S31−S32−S41+S42)`
- Orthogonal channel: TX = ports {1,3}, RX = ports {2,4} → `SDD21 = 0.5×(S21−S23−S41+S43)`

**Notes:**

- **A wrong port mapping assumption was caught and corrected mid project** (applying the wrong file's formula to the Traditional channel produced a smooth, artificially low-resonance curve with no physical meaning, and was resolved by verifying against each file's own header).

- **A significant causality violation was identified in the Traditional channel file** during transient simulation:

```
"after causality enforcement, the maximum in-band error is 134.6%, detected in S2_4 at 14.3 GHz"
```

An initial hypothesis was that Spectre's required causality correction might substantially alter the effective channel response, potentially explaining an unexpectedly open early eye result. **This hypothesis was tested and rejected** (See Section 5 and Section 8 (Pitfalls)).

- **Attempted, abandoned:** a channel-derived zero frequency (fz) extraction via upper-envelope peak fitting and −3dB corner detection. Failed because even the *envelope* of the Traditional channel's ripple peaks does not follow a smooth, monotonic trend, rather, the connector transition itself introduces broadband resonance, not just isolated notches. A rule-of-thumb (fz ≈ fNyquist / 2.8) was used instead.

<p align="center">
  <img src="./Plots/Channel%20Loss%20Graphs/Ripple_Peaks.png" width="600" alt="Ripple Peaks Envelope Extraction Attempt">
</p>
<p align="center">
  <em>Fig. 2.1 Attempted zero extraction via upper-envelope peak fitting on the Traditional channel's resonance profile.</em>
</p>

---

## 3. Device Characterization (gm/Id methodology)

Rather than hand square-law equations (which overestimate gm significantly for 45nm short-channel devices, confirmed empirically: assumed gm ≈ 5.33mS via `gm=2Id/Vov`, actual simulated gm ≈ 3.14mS at comparable current, a ~40% overestimate), transistor sizing used gm/Id-based design:

| Parameter | Value |
|---|---|
| Chosen operating point | gm/Id ≈ 9 V⁻¹ (moderate/strong inversion, balances gain efficiency vs. bandwidth) |
| Vgs at this point | 700.3 mV |
| Id/W (current density) | 69.75 µA/µm |

---

## 4. Design Process & Key Corrections

- **DC gain formula correction:** an early derivation used `Peak/DC ratio = 1 + gm·RD` (missing a factor of 2). Correct form, verified against a Gondi (JSSC 2007) reference and Jain's own circuit: `DC Gain = gm·RL / (1 + gm·RD/2)`. This changed the computed degeneration resistor from 259Ω → 518Ω (design value).
- **Half-circuit → physical component scaling:** design equations use half-circuit values; the physical resistor/capacitor placed between the two source nodes must be **2×RD** and **0.5×CD** respectively (confirmed against Jain's own schematic figure).
- **fp2 target correction:** an initial claim that fp2 needed to sit at 1.5–2× the *data rate* (18–25 GHz) was checked and found to contain both a math error and a units error. Jain's own working design uses fp2 ≈ 1.33× Nyquist. fp2 was set to 10 GHz (~1.6× Nyquist) as a middle ground.
- **Missing CL bug:** an early schematic omitted the load capacitor (CL) entirely, causing fp2 to be governed only by parasitic Cgd/Cdb.
- **Real vs. ideal gain gap:** hand-calculated ideal peak gain (gm·RL) consistently exceeded simulated results, attributed to finite transistor output resistance (ro ≈ 2.2kΩ) and body effect (gmb ≈ 604µS), both confirmed via DC operating-point analysis.
- **Single-stage gain ceiling:** resolved by cascading two identical stages (precedented directly in Agale's SJSU thesis).

---

## 5. Final AC Results

| Configuration | DC Gain | Peak Gain | Peak Frequency |
|---|---|---|---|
| Single stage (final tuned) | ~0 dB | 5.81 dB | 7.6 GHz |
| Two-stage cascade (tuned for Traditional channel) | ~0 dB | **9.00 dB** | 6.25 GHz |
| Two-stage cascade (tuned for Orthogonal channel) | ~0 dB | **10.15 dB** | 6.25 GHz |

<p align="center">
  <img src="./Plots/Second%20Phase%20%28Double%20Stage%29/CTLE_Double_Stage_Gain_Trad.png" width="600" alt="Double Stage CTLE Gain Traditional">
</p>
<p align="center">
  <em>Fig. 5.1 Initial attempt, AC Differential Gain plotted for traditional backplane, and marked @ 6.25GHz.</em>
</p>
<p align="center">
  <img src="./Plots/Second%20Phase%20%28Double%20Stage%29/CTLE_Double_Stage_Gain_Orth.png" width="600" alt="Double Stage CTLE Gain Orthogonal">
</p>
<p align="center">
  <em>Fig. 5.2 AC Differential Gain plotted for orthogonal backplane, and marked @ 6.25GHz.</em>
</p>

---

## 6. Transient / Eye Diagram Results

PRBS7 differential input, 80ps bit period, 15ps rise/fall (within the channel file's stated edge-rate limit), real channel loaded via a 4-port `nport` block with passivity/causality enforcement enabled.

**These are the final, cache-verified results** (see Section 8, Pitfall #3, an earlier run on the Traditional channel was invalidated by a stale simulator cache and has been superseded by the numbers below):

| Metric | Orthogonal Pre-CTLE | Orthogonal Post-CTLE | Traditional Pre-CTLE | Traditional Post-CTLE |
|---|---|---|---|---|
| Eye Height | −15.96 mV (closed) | **+96.25 mV (open)** | −124 mV (closed) | **−317 mV (more closed)** |
| Eye Amplitude | 176.6 mV | 339.7 mV | 87.06 mV | 220.4 mV |
| Eye S/N | 2.751 | 4.186 | 1.237 | **1.23 (no improvement)** |
| Rise Time | 41.19 ps | 21.62 ps | 21.89 ps | 13.04 ps |
| Fall Time | 41.81 ps | 27.94 ps | 19.9 ps | 12.57 ps |

**Key finding:** the identical CTLE design, same tuning methodology, produces opposite outcomes on two channels with nearly identical raw loss (−10.15 vs. −9.73 dB) but wildly different resonance behavior (0.03 dB vs. 7.83 dB average ripple).

<p align="center">
  <img src="./Plots/Eye%20Diagram%20Analysis%20%28Traditional%20Backplane%29/CTLE_Trad_Trans.png" width="600" alt="Traditional Backplane Transient Waveform">
</p>
<p align="center">
  <em>Fig 6.1 Transient waveform for Traditional channel.</em>
</p>

<p align="center">
  <img src="./Plots/Eye%20Diagram%20Analysis%20%28Traditional%20Backplane%29/Eye_Diagram_Trad.png" width="600" alt="Traditional Backplane Eye Diagram">
</p>
<p align="center">
  <em>Fig 6.2 Eye diagram for Traditional channel.</em>
</p>

<p align="center">
  <img src="./Plots/Eye%20Diagram%20Analysis%20%28Orthogonal%20Backplane%29/CTLE_Orth_Trans.png" width="600" alt="Orthogonal Backplane Transient Waveform">
</p>
<p align="center">
  <em>Fig 6.3 Transient waveform for Orthogonal channel.</em>
</p>

<p align="center">
  <img src="./Plots/Eye%20Diagram%20Analysis%20%28Orthogonal%20Backplane%29/Eye_Diagram_Orth.png" width="600" alt="Orthogonal Backplane Eye Diagram">
</p>
<p align="center">
  <em>Fig 6.4 Eye diagram for Orthogonal channel.</em>
</p>

On the smooth Orthogonal channel, the eye opens cleanly, and eye height flips from negative (statistically overlapping logic levels) to positive (clean separation).

On the resonant Traditional channel, gain and edge speed both measurably improve (amplitude nearly triples, rise time nearly halves), but **eye height gets worse, not better**, after equalization. The mechanism: a linear CTLE amplifies everything in its passband uniformly, including the reflection-driven ISI components, not just the wanted signal. Since the Traditional channel's closure is dominated by resonance rather than simple attenuation, adding gain amplifies the interference right along with the signal, producing a net negative outcome despite every individual AC metric (gain, bandwidth, edge speed) improving. This is a structural limitation of single-zero/pole linear equalization, not a tuning deficiency, and directly motivates the DFE addition described in Section 9.

---

## 7. Tools & Methodology Notes

- **Cadence Virtuoso / Spectre**: gpdk045-class 45nm PDK
- **gm/Id sizing methodology** instead of square-law hand equations
- **ADE Parametric Analysis**: for manual iterative tuning of Rd/Cd/Rl/CL against real simulated AC response
- **Python (numpy/scipy in JupyterLab)**: Touchstone S-parameter parsing, mixed-mode SDD21 conversion, and channel resonance quantification

---

## 8. Known Limitations / Pitfalls Found and Corrected

- **Stale simulator cache produced a false result.** Spectre caches `nport` impulse responses in `~/.cadence/mmsim/*.bin` and can silently reuse a stale cached response from a previously-simulated channel file, even with the correct file loaded and correct port wiring. This produced a falsely clean/open eye on the Traditional channel in one run. **Diagnosed by checking the simulation log for a "Reuse impulse responses from..." message**, and resolved by clearing the cache directory and forcing a fresh computation.
- **A causality-correction hypothesis was proposed and rejected.** Given the Traditional channel file's significant causality violation (134.6% in-band error), it was hypothesized that Spectre's required correction might substantially improve the effective channel response. The cache-corrected re-simulation directly disproved this, the properly simulated eye is closed, in fact more so than the CTLE's input.
- **CL and fz remain placeholder/rule-of-thumb values**, not derived from a real downstream stage or a clean channel-derived extraction.
- **No PVT corner analysis** performed in this scope, on hold for now.
- **No physical layout** as of yet, schematic/simulation level only.

---

## 9. Current Phase: DFE Design (In Progress)

Following Jain's own thesis conclusion (Ch.8), a Decision Feedback Equalizer is being added specifically to address the Traditional channel's residual, resonance-driven ISI that the CTLE alone could not correct.

**Scope decision:** Jain's own suggested full pipeline places an FFE stage between the CTLE/driver amp and the DFE, with the DFE's tap coefficients set from whatever postcursors remain *after* the FFE has already reduced them. This project deliberately omits the FFE stage in this phase, meaning the DFE will need to cancel larger, more numerous postcursors on its own than it would in Jain's complete pipeline. This is a documented scope reduction, not an oversight; if postcursor extraction shows the **DFE alone is insufficient, an FFE stage is the natural follow-up**, exactly as Jain's own pipeline anticipates.

**Immediate next steps:**
1. Extract normalized postcursor values via a single-pulse response through the channel + CTLE (sampling at 80ps bit-period intervals from the main cursor), to determine required tap count from real data rather than assumption.
2. Design a 1-tap DFE first (behavioral, then transistor-level resistor-load summer topology per Jain Fig 8.3), targeting the Traditional channel specifically.
3. Extend to additional taps only if the postcursor data shows it's warranted.

---

## References

1. A. Jain, *Equalization in Continuous and Discrete Time for High Speed Links Using 65 nm Technology*, MS Thesis, University of Illinois at Urbana-Champaign, 2016.
2. P. V. Agale, *Low-Voltage Continuous-Time Linear Equalizer for Digital Video Applications*, MS Thesis, San José State University, 2014.
3. S. Gondi and B. Razavi, "Equalization and Clock and Data Recovery Techniques for 10-Gb/s CMOS Serial-Link Receivers," *IEEE Journal of Solid-State Circuits*, vol. 42, no. 9, 2007.
4. N. Tracy and A. Pachon (TE Connectivity), "Channel Simulations for 112G Backplane Analysis," IEEE P802.3ck Task Force contribution, Jan. 2019.
5. J. Baprawski, "SerDes System CTLE Basics," 2012.
6. S. Palermo, ECEN720/689 SerDes Circuit Design course material, Texas A&M University.
7. B. Razavi, "The Decision-Feedback Equalizer [A Circuit for All Seasons]," *IEEE Solid-State Circuits Magazine*, Fall 2017.
8. M. Aldacher, "SERDES Design of RX Decision Feedback Equalizer"
