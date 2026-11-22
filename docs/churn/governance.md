# Governance: model card and prod approval

## Model card

Every registered version gets `model_card/model_card.md` and `.json` in its MLflow run:

- **Intended and out-of-scope use.**
- **Data lineage:** source and DVC md5.
- **Test metrics.**
- **Per-group performance:** churn rate, predicted rate, recall, FPR and ROC AUC per group.
- **Fairness gaps** for sensitive attributes: the recall gap (equal opportunity) and the predicted-rate gap (demographic parity), with a 10% tolerance.
- **Automatically detected blind spots,** plus limitations and privacy notes.

## Prod approval

Promotion to `prod` is refused unless all three checks pass:

1. the version has a model card,
2. a named person approves it (`--approved-by`),
3. if the card flags a fairness gap, the reviewer acknowledges it (`--fairness-reviewed`).

```bash
uv run churn promote --from staging --to prod --approved-by "Your Name" --fairness-reviewed
```

The approver, a UTC timestamp and the review status are written onto the model version as tags, which gives an audit trail per prod model. A rollback to `prod-previous` needs a named approver too, but no new fairness review: that version was already approved.

## What the real model card showed

- **Fairness:** gender gaps were small (≤ 2.7%). Senior citizens are flagged 18.9 points more often, which largely follows their real churn rate (44% vs 23%), and their recall gap was 6.2%, within tolerance. The version is flagged, and a reviewer has to acknowledge it.
- **Blind spots:** the model almost never flags one- or two-year contract customers, even though 12% of one-year customers churn.
