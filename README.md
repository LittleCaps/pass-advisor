# Pass Advisor · Pass Decision Simulator

**[▶ Live demo](https://littlecaps.github.io/pass-advisor/)** — runs entirely in the browser, nothing to install.

Given a frozen snapshot of both teams' **static positions**, the app estimates, for every point on the pitch:

1. **Pass success probability P** — can the ball get there?
2. **Advantage gain ΔV (reward − risk)** — how worthwhile is it once the pass succeeds?
3. **Expected value EV = P × ΔV** — the combined recommendation.

Research question it illustrates: *with both formations frozen at one instant, where is the best place to pass the ball?*

![App preview](figures/app_preview.png)

---

## Quick start

Open `pass_advisor.html` in any modern browser (or use the live demo above). **Zero dependencies, works offline.** The model and the UI live in this single HTML file — no Python, no build step.

---

## Usage

### Arranging positions

- **Drag** any red/blue dot to move a player; touch drag works on mobile.
- **Formation presets**: the "Formation presets" panel re-arranges either team with one click
  (4-3-3, 4-4-2, 4-3-1-2, 4-2-3-1, 4-1-4-1, 5-3-2, 5-4-1).
  Red attacks to the right; Blue is mirrored automatically. The other team stays put, and you can keep dragging to fine-tune.
- "Reset positions" restores the initial example scene.

### Jersey numbers

- Each dot shows a jersey number; both teams default to 1–11.
- **Click** a player to select it (yellow ring), then edit the number (1–99) in the "Selected player" panel.

### Choosing the passer (yellow circle)

Any of:

1. **Double-click** a red player → they become the passer;
2. Click to select a red player, then press "Set as passer";
3. **Drag the ball** (yellow circle) to within 2.5 m of a red player and release.

There is also a "Passer = nearest red" button.

### Choosing the target (star)

- **Auto (default)**: the app evaluates the selected surface on an 84×54 grid; the white ★ marks the maximum and a yellow arrow points to it from the passer. Switching between the three surfaces (P / ΔV / EV) moves the ★.
- **Custom**: **click any empty spot on the pitch** to pin the pass target there (cyan ◎ star + dashed arrow); the side panel shows its values. "Clear custom target" restores the automatic ★.

### Readouts and parameters

- Hover anywhere to see P / reward / risk / ΔV / EV at that point in real time.
- Three sliders: λ (risk weight — higher is more conservative), interception sensitivity, control softness.

---

## Repository layout

```
pass_advisor/
├── pass_advisor.html      # the app (single-file web page: model + UI)
├── index.html             # identical copy served by GitHub Pages
├── README.md              # this file
├── LICENSE                # MIT
└── figures/
    ├── app_preview.png    # three-surface output preview
    ├── fig1_soccermap.png # literature sketch ①: SoccerMap pass probability surface + 5×5 optimum
    ├── fig2_unet_epv.png  # literature sketch ②: U-Net EPV reward/risk/net triptych
    └── fig3_obpv.png      # literature sketch ③: OBSO vs OBPV pitch-wide space value
```

---

## Model (current version)

Coordinates: 105 × 68 m pitch; red attacks to the right (increasing x), toward the goal at (105, 34).

- **Pass success** = control probability × lane openness × distance decay
  - Control: sigmoid of the difference between the nearest teammate's and nearest defender's distance to the target (a soft Voronoi)
  - Lane openness: sigmoid of the minimum perpendicular distance from any defender to the passer→target segment
  - Distance decay: exp(−distance/26)
- **Reward** = control probability × proximity to goal (exp(−distance-to-goal/20), weighted toward the central channel)
- **Risk** = Gaussian density of defenders around the target × (higher further upfield)
- **Advantage ΔV** = reward − λ × risk (λ = 0.85 by default)
- **Expected value EV** = P × max(0, ΔV)

> ⚠️ **Model status**: the formulas are a **hand-tuned analytic approximation for teaching purposes** — the framework follows the literature below, but the coefficients are **not fitted to real tracking data**. Good for illustrating the ideas, trying out formations, and interactive prototyping; serious analysis requires calibration on real data (see "Next steps").

---

## Literature

| # | Paper | Key contribution | Used here for |
|---|-------|------------------|---------------|
| ① | **SoccerMap** (Fernández & Bornn 2020) | Predicts a pass-success-probability surface for every pitch location; searches a 5×5 grid around expected teammate runs for the optimal pass | Success surface + grid search for the optimum |
| ② | **U-Net EPV / OJN-Pass-EPV benchmark** (Overmeer, Janssen & Nuijten 2025) | Splits each pass target's value into reward vs risk; 15-second evaluation window; adds ball-height features; first EPV benchmark | Advantage ΔV = reward − λ×risk |
| ③ | **OBPV** (Ogawa, Umemoto & Fujii 2025) | Extends OBSO (accurate mainly near goal) to pitch-wide space value, especially for transitions | Pitch-wide (not only box) evaluation |
| ④ | **Physics-Based Pass Probabilities** (Spearman et al. 2017) | Physics-based motion model of pass completion; the "control + lane interception" structure of our success formula descends from it | Structure of the success formula |
| ⑤ | **OBSO / Beyond Expected Goals** (Spearman 2018) | Off-Ball Scoring Opportunity: the classic pitch-control × space-value framework | Soft-Voronoi control probability |

### Citations

```
[1] Fernández, J., & Bornn, L. (2020). SoccerMap: A Deep Learning Architecture
    for Visually-Interpretable Analysis in Soccer. ECML-PKDD 2020.
    arXiv:2010.10202.

[2] Overmeer, P., Janssen, T., & Nuijten, W. (2025). Revisiting Expected
    Possession Value in Football: Introducing a Benchmark, U-Net Architecture,
    and Reward and Risk for Passes. arXiv:2502.02565.

[3] Ogawa, R., Umemoto, R., & Fujii, K. (2025). Pitch-wide Space Evaluation
    for Transitions in Soccer (OBPV). arXiv:2505.14711.

[4] Spearman, W., Basye, A., Dick, G., Hotovy, R., & Pop, P. (2017).
    Physics-Based Modeling of Pass Probabilities in Soccer.
    MIT Sloan Sports Analytics Conference 2017.

[5] Spearman, W. (2018). Beyond Expected Goals.
    MIT Sloan Sports Analytics Conference 2018.
```

> Method details of ①②③ (SoccerMap's 5×5 grid optimum search, U-Net EPV's reward/risk split and the OJN-Pass-EPV benchmark, OBPV's pitch-wide extension of OBSO) were checked against the original PDFs during development.

---

## Next steps (smallest investment first)

1. **"Pass to whom" mode**: evaluate only at the 11 teammates instead of every pitch location — closer to real decision-making and to SoccerMap's original setting.
2. **Player velocities**: positions are currently static. Adding velocity vectors and using "expected position one second ahead" in the success model (the core of pitch control) gets much closer to the real game.
3. **Calibrate on real data**: with event data (pass success/failure) or StatsBomb 360 freeze frames, fit the formula coefficients by logistic regression so the numbers are grounded.
4. **Python backend**: only needed to plug in trained deep models (SoccerMap / U-Net EPV) — then consider Streamlit / Dash.

---

## License

MIT — see [LICENSE](LICENSE).
