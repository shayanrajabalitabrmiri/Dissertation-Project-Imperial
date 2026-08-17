# Dissertation-Project-Imperial
CO2 Comparison of a Bartlett-Lewis stochastic model with an AI tool for rainfall generation : Supervisor Dr Christian Onof 

**Comparison of a Bartlett–Lewis Stochastic Model with an AI Tool for Rainfall Generation**
Shayan Rajabali · MSc Environmental Engineering, 2025–2026
Supervisor: Prof. Christian Onof

This repository contains all the code, data and outputs behind my dissertation.
Below is a guide to what is where, and what each notebook actually shows.

---

## How to navigate this repository

```
/Data           the three rainfall records
/notebooks      all the Jupyter notebooks (start here)
/outputs        every figure and results table each notebook produced
/docs           the dissertation and reference papers
```

Each notebook writes its figures and CSVs into its own folder inside `/outputs`,
so you can look at any result without running anything.

---

## Part 1 — The Bartlett–Lewis model

The mechanistic benchmark: storms arrive, storms contain rain cells, cells rain.
Six parameters per month, calibrated to match the observed statistics.

### `Heathrow_BL_Complete_Workflow_v9.ipynb`
**The main calibration.** Heathrow, 1949–2022.
Quality control → observed statistics → calibration → validation → extremes.

What it shows:
- The record has no missing-value codes: gauge outages were written as zeros.
  I find them by looking for calendar months with exactly 0 mm, which is impossible
  in London. Thirteen months get masked.
- The fit reproduces the observed statistics to about 2%.
- Storm rate goes from ~20 storms/month in January to ~7 in July, while cell
  intensity moves the opposite way. That is the frontal/convective contrast, and
  it came out of the data, not from anything I imposed.
- The model produces **10–15% too few storms**, which is also why it leaves too
  many completely dry days. One fault, two symptoms — this is its main limitation.

→ `/outputs/heathrow_v9/`

### `Ringway_BL_Complete_Workflow_v5.ipynb`
**Same pipeline, second station.** Manchester, 1949–2001.

What it shows:
- This file is a *wet-day register* — dry days were simply left out. Scaling it
  naively gives 1,470 mm/year against Manchester's real ~810 mm. The fix is to
  restore every missing day as dry, and the rebuilt series lands on 803 mm.
- Manchester gets 36% more rain than Heathrow, but **not by raining harder**:
  the wet-hour intensity is nearly identical. It rains more often and for longer.

→ `/outputs/ringway_v5/`

### `Bochum_BL_Complete_Workflow_v15.ipynb`
**Third station, and the literature check.** Germany, 1931–1999.

What it shows:
- Bochum resolves 0.01 mm where the British gauges resolve 0.1 mm, so a fifth of
  its wet hours are below what a UK gauge could even record. This is why I round
  all simulated rainfall to the gauge resolution before comparing anything.
- Kaczmarska et al. (2014) calibrated the same model to this same gauge, so I run
  a second fit under their conditions. It moves towards their published values,
  which shows the differences come from the *fitting recipe*, not from the rainfall.
- **The identifiability problem, in one example.** In November, the data cannot
  separate the storm rate from the rain per storm — 0.0020 storms/h × 46.1 mm and
  0.0100 storms/h × 9.2 mm give the same rainfall to three decimals. Only the
  product is constrained. That is why the storm rate needs a bound.

→ `/outputs/bochum_v15/`

### `Comparison_3Stations_v9.ipynb`
**The three calibrations side by side.**

What it shows:
- The storm shortfall repeats at all three stations: −16%, −15%, −15%. Two
  countries, two instrument resolutions — so it is a property of the model,
  not of any one record.
- Extremes are fine at Heathrow and Ringway but underestimated by 19% at Bochum.

→ `/outputs/comparison_v9/`

---

## Part 2 — TimeGAN (the model that did not work)

### `TimeGAN_Heathrow.v1.ipynb`
**First full attempt.** Four seasonal models, trained from scratch.

What it shows:
- TimeGAN **mode-collapses**: it produces almost the same 4-day window every time.
  Two generated windows correlate at 0.67, where two real stretches of rainfall
  correlate at 0.003.
- Its heaviest hour in an entire winter is 0.9 mm against 10.8 mm observed.
- **The most useful thing in this notebook**: across four training configurations,
  the ones that scored *best* on the usual statistics were the *most broken*. The
  standard statistics cannot see mode collapse. That is why I added three
  diagnostics, which are in every notebook after this one.

→ `/outputs/timegan_heathrow_v1/`

### `TimeGAN_Supervisors_Feedback.ipynb`
**My supervisors' suggestions, one experiment each.**

What it shows: five variants, each a single change to the same code — the wet/dry
test, dropping windows with no rain, separating the losses, and weighting the large
values. None of them fixes the collapse.

→ `/outputs/timegan_supervisors/`

### `TimeGAN_Results.ipynb`
**Everything in one notebook, trained live.**
If you only run one TimeGAN notebook, run this one. It also simulates Bartlett–Lewis
for comparison, so the two models are side by side on identical data.

→ `/outputs/timegan_results/`

### `TimeGAN_Binary_QuickFix.ipynb`
**The experiment that explains the failure.**

Prof. Onof suggested setting every positive rainfall value to 1 and asking whether
the network can learn just the wet/dry pattern — with no amounts left to get wrong.

What it shows:
- It still fails. **P(rain | it rained last hour) = 1.000** against 0.610 observed:
  once it starts raining in a generated window, it never stops.
- So the problem is not the zeros or the intensities — it is the adversarial
  training itself.
- It also tests a **hybrid**: TimeGAN decides *when* it rains, Bartlett–Lewis
  supplies *how much*. The amounts are fixed completely (heaviest hour 0.9 → 8.7 mm)
  but the timing is not, because the hybrid inherits TimeGAN's blocks.

→ `/outputs/timegan_binary/`

---

## Part 3 — The AI model that works

### `Autoregressive_Heathrow.v1.ipynb`
**One small network, trained by maximum likelihood.**

At every hour it answers two questions: *will it rain next hour?* and *if so, how much?*
Then it rolls forward, feeding each generated hour back in. No adversary, no competition
— so mode collapse is impossible by construction.

What it shows:
- It passes all three collapse diagnostics on the first attempt, with no tuning.
- It matches the observed wet/dry transition probabilities to the second decimal
  (0.611 against 0.610 observed; TimeGAN gives 1.000).
- It beats Bartlett–Lewis on four of the five statistics, and is the closest of the
  three models on storm counts — which neither model was fitted to.
- It does this with **3,654 weights against TimeGAN's 17,211**, in five minutes.
- Its weakness is the extreme tail, which I improve with a two-component gamma
  mixture — but at a measurable cost to the mean. That trade-off is discussed
  honestly in the notebook.

→ `/outputs/autoregressive_heathrow_v1/`

---

## The three checks I use everywhere

After nearly reporting a broken model as a success, I stopped trusting summary
statistics alone. Every generator in this repository is also checked with:

1. **Is any hour of the window special?** The chance of rain should be flat across
   the 96 hours. Real: 0.025–0.132. Collapsed TimeGAN: 0.000–0.239.
2. **Are two generated windows different?** Real rainfall: 0.003. TimeGAN: 0.667.
3. **Is the rain shared out realistically?** A spike at zero means the model
   produces all-or-nothing windows.

---

## The data

| Station | Period | Native step | Annual | Dry hours | Gauge |
|---|---|---|---|---|---|
| Heathrow (London) | 1949–2022 | 1 h | 590 mm | 91.5% | 0.1 mm |
| Ringway (Manchester) | 1949–2001 | 1 h | 803 mm | 87.8% | 0.1 mm |
| Bochum (Germany) | 1931–1999 | 5 min | 793 mm | 86.3% | 0.01 mm |

---

## Running the code

```bash
pip install numpy pandas matplotlib scipy torch jupyter
```

The Bartlett–Lewis notebooks also need `pyBL`. All notebooks use fixed random seeds,
and the generated rainfall is saved as `.npz` next to the figures, so you can rebuild
any plot without retraining a model.

---
