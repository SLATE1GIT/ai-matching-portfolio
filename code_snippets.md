# Selected code and methodology

These are **partial, non-standalone excerpts**, not the complete implementation. They come from the reviewed Databricks demonstration source; SQL excerpts remove only Databricks `# MAGIC ` wrappers. Context, registrations, input/setup code, most rules and dashboard queries are intentionally omitted. The snippets are for inspection, not execution. No new matching logic has been introduced.

## 1. Normalization: comparable representations
Phone formatting is normalized before comparing digit strings:

```python
def _standardize_phone(text):
    if text is None:
        return None
    digits = re.sub(r'[^0-9]', '', text)
    if len(digits) == 11 and digits.startswith('1'):
        digits = digits[1:]
    return digits if digits else None
```

Other implemented normalization includes business stop-word removal, street-suffix expansion, uppercasing and alternate name forms. Formatting cleanup supports comparison; it does not by itself establish business identity.

## 2. Segmentation: parse fields and group candidate pairs
Here **segmentation means decomposing text and narrowing comparison groups**, not market segmentation, learned clusters or an implemented business-policy selector. The code extracts leading street numbers separately from street names, extracts suites and text after “at”/“@”, and tokenizes names. Two examples:

```python
def _extract_street_number(text):
    if text is None:
        return None
    m = re.match(r'\s*(\d+)', text)
    return m.group(1) if m else None

def _name_words_3plus(text):
    if text is None:
        return []
    return [w for w in text.upper().split() if len(w) >= 3]
```

The location block groups plausible candidate pairs:

```sql
  FROM pmdemo_norm_a a
  JOIN pmdemo_norm_b b
    ON a.zip_a = b.zip_b
    OR (a.city_a = b.city_b AND a.state_a = b.state_b)
```

Additional pre-filters use SOUNDEX phonetics, prefixes, containment, shared words, phone/street-number agreement and acronyms. These are complementary signals, not a guarantee that every true match survives blocking. Some derived fields are exposed for inspection rather than driving every decision.

## 3. Individual component scoring: different evidence answers different questions
Name similarity and street-name similarity are distinct components. The address comparison below is gated by agreement on the extracted street number:

```sql
  jaro_winkler_similarity(name_norm_a, name_norm_b) AS biz_name_score,

  CASE WHEN street_num_a IS NOT NULL AND street_num_b IS NOT NULL
    AND street_num_a = street_num_b AND street_num_a != ''
    THEN jaro_winkler_similarity(street_name_norm_a, street_name_norm_b)
    ELSE 0.0
  END AS address_score,
```

These are separate SELECT-list fragments, not a complete query. Other implemented evidence includes city agreement/similarity, standardized-phone agreement, suite conflict, shared distinctive words and name containment. Missing phone information is treated as unavailable rather than a confirmed disagreement. Raw names are retained for review; separate raw-name scoring/fallback is not implemented.

## 4. Final scoring: composite plus ordered tier rules
This SELECT-list excerpt computes a composite from prepared components and penalties:

```sql
    ROUND(
        name_component  * 0.30
      + address_score  * 0.35
      + city_component * 0.15
      + phone_component * 0.20
      - CASE WHEN phone_mismatch  = 1 THEN 0.10 ELSE 0 END
      - CASE WHEN suite_conflict = 1 THEN 0.15 ELSE 0 END,
      4
    ) AS composite_score
```

The omitted preparation chooses among available normalized-name comparisons and supplies city/phone components. The composite helps inspect and rank pairs; it is not a calibrated probability and does not alone determine the tier.

One illustrative branch from the ordered tier CASE expression:

```sql
      WHEN biz_name_score >= 0.90
       AND address_score  >= 0.85
       AND phone_match = 1
        THEN 'DEFINITE'
```

Other branches and their thresholds are intentionally omitted, rather than publishing the full decision system. They combine different evidence for DEFINITE, LIKELY, PROBABLE, NAME_LOCATION, BUSINESS_CHANGE, POSSIBLE and NO_MATCH. Rules are evaluated in priority order; human-readable reasons accompany the result. Pair deduplication retains the best composite per ID pair. **Tiers are not composite-score bands.**

In the synthetic output, Joe's Garage / Smith Auto Repair has composite 0.8686 and BUSINESS_CHANGE, while Burger King / BK Burger has composite 0.8087 and DEFINITE. Different combinations of evidence—not merely the larger number—explain the distinction. These are illustrative rule outputs, not verified real-world events or identity judgments.

## Context and limits
The earlier three-part discussion framed business purpose → data preparation → layered decision methods. Saved summaries of comment replies emphasized purpose-dependent error tolerances and human review for remaining uncertainty. Full post/comment text was not recovered; no verbatim reproduction is claimed. Configurable policies and a manual-review workflow are broader process ideas, not implemented notebook features.

The underlying example ran successfully in Stephen's Databricks test environment. These extracted fragments have not been executed independently. Their Python helpers require omitted context such as `re`; SQL requires prepared views, registered helper functions and the documented `jaro_winkler_similarity` built-in. No model training, production readiness or accuracy claim is made.
