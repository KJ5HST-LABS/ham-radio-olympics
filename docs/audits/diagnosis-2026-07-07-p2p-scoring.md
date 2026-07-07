# Diagnosis — P2P (Park-to-Park) scoring "not calculating"

**Date:** 2026-07-07
**Session:** 37
**Reported by:** N5VYS (relayed by operator) — *"the Olympic score on my behalf is not calculating correctly … I have several P to P QSOs but no change."*
**Method:** Read-only SELECT queries against the production DB (`kd5dx:/data/ham_olympics.db`, opened `mode=ro`), tracing N5VYS's park QSOs through the real `scoring.py` matcher. No writes, no recompute.

---

## TL;DR

Two separate things are in play. Neither zeroed N5VYS out — he has **117 points across 22 matches**. The complaint is that **recent P2P QSOs don't *add*.**

1. **Immediate cause (working as designed): unconfirmed QSOs don't score.** N5VYS has **1031 unconfirmed QSOs** (of 2451), including **34 of his 92 P2P QSOs**. Every recent P2P contact that failed to score was `is_confirmed=0`. Park sports all have `show_live_results=0`, so unconfirmed contacts are excluded until the *other* operator QSLs (QRZ/LoTW) and N5VYS re-syncs.
2. **Deeper defect (real bug): "P2P" earns nothing beyond a normal activation.** `should_award_pota_bonus()` awards the +2 "Park-to-Park" bonus for *any* activation in a park-targeted match — it **never checks whether the DX station is at a park** (`dx_sig_info`). So a genuine P2P QSO and a plain activation QSO score identically, and the bonus is a **flat per-match** value, not per-QSO. Making more P2P contacts at a park you've already activated in a match changes the score by **zero**.

---

## Evidence (N5VYS, production)

| Metric | Value |
|---|---|
| Total QSOs | 2451 (1420 confirmed, **1031 unconfirmed**) |
| Park-involved QSOs | 850 |
| P2P QSOs (both `my_sig_info` and `dx_sig_info`) | 92 (**58 confirmed, 34 unconfirmed**) |
| Medal rows | 22 |
| **Total Olympic points** | **117** |
| Sports entered | 9 (park sports: 3, 5, 8, 9; plus continent/country + Winter Fieldnic) |

**Trace of recent park QSOs (`matches_target` replayed):**
- Confirmed P2P → **scored** as an activation of the operator's own park, e.g. `2026-05-28 KD2YIO my=US-0756 dx=US-6532 conf=1 → scores in m106 (park=US-0756)`.
- Unconfirmed P2P → **scored nowhere**, e.g. `2026-05-28 NC2Y my=US-0756 dx=US-0232 conf=0 → NONE (blocked: UNCONFIRMED)`.

The confirmed P2P contacts *do* count — but only toward the park N5VYS was activating (`US-0756`), and they contribute no more than his non-P2P activation contacts at that same park.

---

## Code analysis

**Gate 1 — confirmation** (`scoring.py:get_matching_qsos`, ~L303-321; `validate_qso_for_mode` L222):
QSOs are included only if `is_confirmed = 1 OR live_mode`. `live_mode = match.show_live_results`, which is `0` for every park sport. → unconfirmed P2P is invisible to scoring.

**Gate 2 — the "P2P" bonus is a misnomer** (`scoring.py:517-544`):

```python
def should_award_pota_bonus(target_type, role, competitor_at_park):
    is_pota_target = target_type in ("park", "pota")
    if is_pota_target and competitor_at_park:   # "Park-to-Park": +2
        return 2
    elif is_pota_target or competitor_at_park:  # +1
        return 1
    else:
        return 0
```

`competitor_at_park` is just `bool(my_sig_info)` (`compute_medals` L469: `has_pota = any(q.has_pota …)`, and `q.has_pota = bool(my_sig_info)`). **`dx_sig_info` is never consulted.** So "POTA target + I'm at a park" is treated as P2P — but that is simply *an activation during a park-targeted match*, not park-to-park.

- On the **hunter/work side** the +2 is legitimately P2P: a park-target *work* match matches on `dx_sig_info`, so the DX is provably at a park and `my_sig_info` proves the competitor is too.
- On the **activator/combined side** the +2 is awarded even when the worked station is **not** at a park — the bug. And because `pota_bonus` is computed once per `(callsign, role)` group and added once to `total_points` (`compute_medals` L508-512), it is **flat per match**: 1 activation QSO and 30 activation QSOs (P2P or not) both yield the same single +2.

**Contrast — the Triathlon scorer does it correctly** (`scoring.py:1397-1402`):

```python
has_my_park = bool(qso["my_sig_info"])
has_dx_park = bool(qso["dx_sig_info"])
pota_bonus = 100 if (has_my_park and has_dx_park) else 50   # P2P vs single park
```

The medal path and the triathlon path disagree on what "P2P" means. The triathlon definition (both stations at parks) is the correct one.

---

## Impact

- **User-visible:** operators who work many P2P contacts see no score movement, matching N5VYS's report. The activation bonus they already earned caps out at +2 per match regardless of P2P volume, and unconfirmed contacts don't count at all.
- **Fairness:** genuine P2P work is not rewarded above ordinary activation in the medal standings.
- **Scope:** the bonus logic is global — any fix changes standings for **all** competitors in park sports and requires a full medal recompute.

---

## Recommended fixes (NOT implemented — needs a product decision + operator approval)

This is a scoring-semantics change affecting every competitor's standings, so it belongs in a separate, approved session with a full recompute. Options:

1. **Make +2 mean true P2P** (align with the triathlon definition): award +2 only when **both** `my_sig_info` and `dx_sig_info` are present; +1 for single-park involvement. Requires threading `dx_sig_info` into `MatchingQSO`/`compute_medals` (today only `my_sig_info` is carried as `has_pota`). *Trade-off:* plain activations drop from +2 → +1, lowering many scores.
2. **Keep flat bonus but document it** as "park-activation bonus," and separately reward P2P volume (e.g., a per-P2P-QSO tally or a dedicated P2P event) so P2P work visibly moves the needle.
3. **Confirmation UX only** (no scoring change): surface "N unconfirmed QSOs pending QSL" on the dashboard so users understand why fresh contacts haven't scored yet.

The intended behavior is a call for the operator. Recommend deciding between (1) and (2), then planning the change (with recompute) as its own session.

---

## Plain-language answer for N5VYS

> Two things. First, contacts only count once they're **confirmed** — the other station has to QSL them (via QRZ or LoTW), then your log re-syncs. Right now a chunk of your recent P2P contacts are still unconfirmed, so they haven't scored yet; they'll come in as they confirm. Second, we found that the app currently gives a park-to-park contact the **same** credit as a normal activation at that park — it isn't giving P2P any extra. That's a bug on our side, and we're going to fix how P2P is scored. Thanks for flagging it. 73!
