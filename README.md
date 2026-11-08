# AI-assisted business matching: methods and selected excerpts

## Why this example exists
A concrete portfolio example of **using AI alongside SQL and business-matching skills**. Stephen provided the prompt/specification; AI generated a code draft and synthetic comparison data, and Stephen reviewed and refined the code. This small collection presents selected code and actual synthetic output—not a full implementation, product launch, trained machine-learning model or validated matching service.

## Four areas to inspect
1. **Normalization:** make business names, addresses and phone formats more comparable.
2. **Segmentation:** parse names/addresses into useful parts and group plausible candidate pairs through location blocking. This is text decomposition and candidate grouping, not implemented market segmentation or learned clustering.
3. **Individual component scoring:** compare names and street names while retaining separate phone, city, containment/shared-word and conflict evidence.
4. **Final scoring and interpretation:** combine components for a diagnostic composite, then apply priority-ordered evidence rules to produce tiers and readable reasons. **No single method or score tells the whole story.**

Start with [selected code and methodology](code_snippets.md). The excerpts cover all four areas but deliberately omit setup, most decision branches and the end-to-end implementation. They are partial and not runnable as supplied. Human judgment and business context remain important; no manual-review application or configurable policy engine is claimed.

## Observed demonstration, not an accuracy benchmark
On November 8, 2026 (America/Chicago, CST), Stephen manually ran the underlying Databricks demonstration with **20 embedded synthetic records per source**. The exported results contained **20 pairs: 19 DEFINITE and one BUSINESS_CHANGE**, with no duplicate ID pairs. This supports a working example of the methods, not measured accuracy, recall, speed or reduction rates.

[The labeled synthetic output](sample_output_synthetic.csv) retains all 20 rows in their original order but only **11 of the original 49 columns**: `id_a`, `id_b`, `name_raw_a`, `name_raw_b`, `biz_name_score`, `address_score`, `phone_match`, `city_match`, `composite_score`, `tier`, `tier_reason`. No retained values were recalculated or changed. Literal `null` denotes unavailable values; it is not zero. Address/phone values and other diagnostic fields are omitted to keep the example focused.

The synthetic fixture has identical normalized address scores, no observed phone/suite conflicts and only two tiers. It is not held-out evaluation data. Scores are not calibrated probabilities. Office Depot / Office Max being labeled DEFINITE is a useful edge case, not verified identity; BUSINESS_CHANGE is likewise a rule interpretation rather than a verified event.

## Scope and safety
Only this README, selected excerpts and a reduced synthetic CSV are included. No complete notebook, credentials, live workspace links, connections, automation or hosting are included or needed to inspect the example. The underlying run was performed by Stephen; the excerpts and data were subsequently reviewed statically. Keep this collection private while reviewing; public release is a separate choice. No license file has been added.
