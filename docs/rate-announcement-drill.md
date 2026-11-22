# Rate Announcement drill (B-05)

CMS publishes the CY2028 Rate Announcement in early April 2027 (the CY2027 one
came out April 6, 2026). Bids are due the first Monday in June. The drill turns
the announcement into a re-priced assumption pack in one sitting.

## Before the announcement

1. Run the baseline pack for each contract under `ma_payment_2027`, which uses
   CY2027 factors plus illustrative CY2028 ranges:

   ```bash
   uv run vantage bid-pack --dataset <dataset> --contract <contract> \
       --county <ssa_code>=<enrollees> ... --bid <bid_pmpm> --seed 2028 \
       --stars-contract <H-number> --net-risk-run <net_risk run, forward_year 2028>
   ```

2. Keep the `run_id`. It is the baseline the drill compares against.

## On announcement day

1. Copy `policy/templates/ma_payment_2028.yaml.example` to
   `policy/sets/ma_payment_2028.yaml`.
2. Fill in these values from the announcement, each with `basis: rule_text` and
   the page in its note:
   - the normalization factor for the 2024 CMS-HCC model
   - the coding pattern adjustment
   - NPCMAGP
   - FFS growth
3. Collapse the `bid.*` ranges onto those values.
4. Load the new set: `uv run vantage load-policy policy/sets/ma_payment_2028.yaml`
   (add `VANTAGE_DB=md:vantage` in front to load it straight into the hosted site,
   keeping its saved runs). `scripts/init_db.py` also picks it up on the next rebuild.
5. Run the drill:
   `uv run vantage bid-drill <baseline run_id> --policy ma_payment_2028 --memo drill.md`.
   It re-runs the pack with the same inputs and seed. It returns the
   assumptions that moved (P50 before and after, and the basis change from
   illustrative to rule_text), plus the headline comparison.
6. When the CY2028 rate book is published, add `ratebook-2028` to
   `reference-data/sources.json`. Run the refresh workflow, then re-run the
   drill with `--rate-year 2028` so the pack reads the new county rates instead
   of growing the 2027 ones. Set the growth range to 0 in that set.
7. Export the memo for the bid team:
   `uv run vantage bid-export <drill run_id> --format memo --out drill.md`.

## Rehearsal

`uv run python scripts/rehearse_rate_drill.py` runs all of the above on a local database,
with the CY2027 Rate Announcement values (April 6, 2026) standing in for CY2028: it fills
the template exactly as on the day (under the id `rehearsal_2028_mock`, status
hypothetical), loads it, drills a Bayside baseline and writes the memo with its "what
moved" table to `outputs/drill-rehearsal/`. The first rehearsal (Nov 22, 2026) ran in
under five seconds of compute; it found that `status` must be final, hypothetical or
proposed, that the plain `init_db.py` route was slower and riskier than loading one set
(hence `load-policy`), and that the memo didn't say what moved (hence `--memo`).
Re-run it after any change to the bid code or the template.
