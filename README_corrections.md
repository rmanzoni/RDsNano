# MC corrections in RDsNano (`BsToDsPhiKKPiMuBuilder`)

Data are never corrected. Everything below applies to MC only.

| what | where | switch (cff / `run_crab.py` option) | default |
|---|---|---|---|
| track momentum scale + smearing (factor f per track) | momenta of mu, K, K, pi before the fits and the mass windows | `trkCalibFile` | on (v4) |
| **q/p covariance scaled by 1/f before the vertex fits** | covariance of the four tracks given to the fits | `calibCovBeforeFit` / `calibCovBeforeFit=` | **True** |
| **muon sigma_dxy quantile morph** | muon covariance (dxy row/column) before the fits and IPs | `applyDxyMorph` + `dxyMorphFile` / `dxyMorph=` | **True** |

Example: `cmsRun test/run_crab.py isMC=True globalTag=... calibCovBeforeFit=False dxyMorph=False`.
The job log prints the choice: `#### MC covariance options (README_corrections.md): calibCovBeforeFit=1 applyDxyMorph=1`.

## WARNING: these are two partial, interim corrections of the track covariance

The momentum calibration corrects the q/p *value*. The morph corrects the muon *dxy uncertainty*. Neither is
a consistent correction of the full 5x5 track covariance (momentum and impact parameters together, with
their correlations), and two half corrections are not necessarily better than none. A full covariance-flow
treatment (impact parameters and momentum at the same time) is the intended long-term solution; until then
both are kept **optional** and **on** by default for this version (decision of R. Manzoni, 10 Nov 2026).
Analyses that are sensitive to vertex / IP uncertainties in MC should check both settings.

## 1. `calibCovBeforeFit` (1/f scaling of the q/p covariance before the fits)

The calibration changes q/p -> (q/p)/f. Propagating this linear change to the covariance gives D C D with
D = diag(1/f, 1, 1, 1, 1): the q/p row and column are divided by f (C(0,0) by f^2). This keeps the
relative momentum resolution of each track unchanged. It is what Bmmm does (`TrackRecord.fit_track`).

- Only row/column 0 (q/p) changes: C(0,0) and the q/p covariances with lambda, phi, dxy, dsz.
  **The dxy and dsz variances are untouched**, so `dxyError()` and the kaon/pion/muon IP errors do not depend
  on this option (checked: identical branches).
- True: the fits (phi, Ds, Bs) get the scaled covariance. False: the fits get the original covariance with
  the calibrated momenta (the behaviour of RDsNano before the rds16 port).
- With |f - 1| < 1% (1-99% range on signal), the fitted quantities move by ~1e-5 to 1e-3 (relative, median)
  between the two settings.
- The mass errors (`ds_custom_m_err`, `kk_m_err`, `phiPi_m_err`, `phiPi_m_err_cal`) use the 1/f-scaled
  covariance **exactly once** in both settings: identical up to the order of the positive-definite fix
  (`correctCovMat`, max 1e-3 relative on `phiPi_m_err`).

Bug fixed on 10 Nov 2026: with the scaling before the fit, `scaleAndCalibrateCovMat` (helper.h) applied 1/f a
second time for `ds_custom_m_err` (q/p variance / f^4 instead of / f^2). Effect: `ds_custom_m_err` off by
0.2% (median), up to 1%. Productions with the bug: 20261109 MC (sig, sig_ext, hb, hb_ext, QCD), RDsNano
<= 7b71fc7. No other branch was affected. Data unaffected.

## 2. `applyDxyMorph` (muon sigma_dxy quantile morph)

MC underestimates the muon impact-parameter uncertainty. The morph maps the MC sigma_dxy (w.r.t. the beam
spot, track and beam-spot errors in quadrature) to data by quantile, binned in muon pT and |eta|. The map
was derived on data with sPlot weights (`data/sigma_dxy_morph_splot_fixed_muonid_fixed_decay_length.json`,
correctionlib, copied from the Bmmm ntuplizer: `--cov-scale <morph> --cov-scale-muon-only`).
It matters because the parking trigger and the trigger scale factors depend on the muon dxy significance.

- Only the track part is rescaled: s = sqrt(sigma_morph^2 - sigma_bs^2) / sigma_trk; the dxy row and
  column of the muon covariance are multiplied by s (C(3,3) by s^2), the correlations are kept.
- Applied before the fits: the Bs vertex (muon + Ds) and every muon IP use the morphed muon.
- Stored: `mu_cov_scale_dxy` (s, = 1 when off), `dxy_mu_err_pv_raw`, `dz_mu_err_pv_raw`,
  `dxy_mu_err_beamSpot_raw` (unmorphed errors). The Bs fit results have no unmorphed copy.
- On signal: s median 1.08 (5-95%: 1.01-1.38); `dxy_mu_err_pv` / raw median 1.07.
- False: no morph; Bs vertex quantities move (median relative ~1e-5 to 1e-3, Bs fitted mass error ~0.2%).
