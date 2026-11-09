# Selected code and methodology

These are **partial, non-standalone excerpts**, not the complete implementation. They come from the reviewed Databricks demonstration source; SQL excerpts remove only Databricks `# MAGIC ` wrappers. Context, registrations, input/setup code, most rules and dashboard queries are intentionally omitted. The snippets are for inspection, not execution. No new matching logic has been introduced.

## 1. Normalization: address components and business-name comparison
Address and business-name preparation are the focus here, rather than phone formatting. Keep raw values alongside comparison fields so a reviewer can inspect what changed.

### Address indicators and street types
The existing code expands street-type and unit indicators (for example ST → STREET, BLVD → BOULEVARD and STE → SUITE), uppercases text and collapses whitespace:

```python
_STREET_SUFFIX_MAP = {
    'ST': 'STREET', 'STR': 'STREET',
    'AVE': 'AVENUE', 'AV': 'AVENUE',
    'BLVD': 'BOULEVARD',
    'RD': 'ROAD',
    'DR': 'DRIVE',
    'LN': 'LANE',
    'PKWY': 'PARKWAY', 'PKWAY': 'PARKWAY',
    'PL': 'PLACE',
    'CT': 'COURT',
    'CIR': 'CIRCLE',
    'TER': 'TERRACE', 'TERR': 'TERRACE',
    'JCT': 'JUNCTION',
    'HWY': 'HIGHWAY',
    'FWY': 'FREEWAY',
    'EXPY': 'EXPRESSWAY',
    'SQ': 'SQUARE',
    'APT': 'APARTMENT',
    'STE': 'SUITE',
    'DEPT': 'DEPARTMENT',
}

def _normalize_street_suffix(text):
    if text is None:
        return None
    text = text.upper().strip()
    for abbrev, full in _STREET_SUFFIX_MAP.items():
        text = re.sub(r'\b' + abbrev + r'\b', full, text)
    text = re.sub(r'\s+', ' ', text).strip()
    return text if text else None
```

### Separate street number, street text and suite evidence
These existing helpers extract the leading number and suite/unit identifier separately, and create street text without the leading number:

```python
def _extract_street_number(text):
    if text is None:
        return None
    m = re.match(r'\s*(\d+)', text)
    return m.group(1) if m else None

def _extract_street_name(text):
    if text is None:
        return None
    text = re.sub(r'^\s*\d+\s+', '', text.upper().strip())
    text = re.sub(r'\s+', ' ', text).strip()
    return text if text else None

def _extract_suite(text):
    if text is None:
        return None
    text = text.upper().strip()
    m = re.search(r'\b(?:STE|SUITE|APT|APARTMENT|UNIT|DEPT)\s*#?\s*(\w+)', text)
    if m:
        return m.group(1)
    m = re.search(r'\b#\s*(\w+)', text)
    if m:
        return m.group(1)
    return None
```

For `456 Oak Ave Ste 200`, the implemented functions yield normalized address `456 OAK AVENUE SUITE 200`, street number `456`, street text `OAK AVENUE SUITE 200`, and suite `200`. This is a traced example of the code, not an additional runtime test. **Suite extraction does not remove the suite from street text.** Keeping the suite as separate evidence allows the matching rules to detect suite conflicts; this is not a full postal-address parser.

### Business-name cleanup: comparison words, not identity
The actual stop-word list and function remove selected whole tokens, including THE, AND, OF and some business/industry terms such as RESTAURANT, GRILL and BAR:

```python
_STOP_WORDS = frozenset([
    'THE', 'AND', 'OF', 'CO', 'INC', 'LLC', 'CORP', 'CORPORATION', '&', 
    'COMPANY', 'LTD', 'LP', 'STORE', 'SHOP', 'RESTAURANT',
    'INC.', 'CORP.', 'LTD.', 'GRILL', 'BAR', 'GROUP', 'HOLDINGS',
    'ENTERPRISES', 'NO', 'NUM', '#',
])

def _remove_stop_words(text):
    if text is None:
        return None
    words = text.upper().strip().split()
    filtered = [w for w in words if w not in _STOP_WORDS]
    result = ' '.join(filtered) if filtered else ' '.join(words)
    return result if result else None
```

If removing every token would leave nothing, the existing function retains the original uppercased tokens. Raw business names also remain available for inspection. Stop words should be chosen for the industry and purpose: removing a meaningful name word can create a false match.

**Requested extensions not implemented in this source:** directional expansion (such as N → NORTH or SW → SOUTHWEST), hospitality stop words HOTEL/INN/SUITES/MOTEL, and general punctuation/special-character stripping. The current list removes `&` or `#` only as standalone tokens and explicitly lists a few punctuated business suffixes; it does not strip arbitrary punctuation inside words. Those additions would require a separately reviewed code change. They are not presented here as implemented or tested behavior.

## 2. Segmentation: parsed components, tokens and candidate groups
The address helpers above illustrate field decomposition. The notebook also extracts text after “at”/“@” and makes name tokens for comparison:

```python
def _name_words_3plus(text):
    if text is None:
        return []
    return [w for w in text.upper().split() if len(w) >= 3]
```

Here **segmentation means decomposing text and narrowing comparison groups**, not market segmentation, learned clusters or an implemented business-policy selector. The location block groups plausible candidate pairs:

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

The underlying example ran successfully in my Databricks test environment. These extracted fragments have not been executed independently. Their Python helpers require omitted context such as `re`; SQL requires prepared views, registered helper functions and the documented `jaro_winkler_similarity` built-in. No model training, production readiness or accuracy claim is made.
