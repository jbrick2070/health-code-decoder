# Before You Pay

A standalone HTML page by Jeffrey A. Brick. Enter a generic medication and quantity, confirm the exact strength/form, and view the seller's API quote. Or enter a procedure/code and ZIP, confirm the matched procedure, and view hospital-published cash records with each record's date. Each result includes a question to copy and a print/save-PDF action.

Live: https://jbrick2070.github.io/health-code-decoder/before-you-pay/

## Run

Serve the repository with Python's http.server, then open /before-you-pay/. There is no build step, package dependency, backend, analytics, credential or API key. The existing Health Code Decoder remains at the repository root.

## Data

- [Cost Plus Drugs public API](https://costplusdrugs.github.io/apidocs/): generic tablet/capsule search by name and quantity. Requires user selection of exact form/strength. Uses requested_quote only when requested_quote_units matches the requested quantity; never multiplies unit price to invent a quote. Shipping and applicable tax are extra. One seller, not a market-wide comparison.
- [FairVisitHealth public API](https://fairvisithealth.com/developers/): query or CPT/HCPCS code plus ZIP. Confirm the returned procedure before viewing facility cash records. Dates and required attribution are shown; inclusions are unknown. Facility rows are alphabetical. No Medicare-to-hospital comparison, inferred savings, insured estimate, or price-based facility recommendation. Free service limit: 60 lookups/IP/day. HTTP429 is surfaced without retries.
- [NLM RxNorm](https://lhncbc.nlm.nih.gov/RxNav/APIs/api-RxNorm.findRxcuiByString.html): optional exact/normalized name lookup (search=2), followed by properties. Only a single product/pack concept is shown as a product name match; an ingredient concept is insufficient. This is not an NDC crosswalk.
- [CARTOS](https://cartos.healthit.gov/TerminologyBrowser/vpc/termsearch): optional FHIR R4 code lookup for the selected RxNorm concept. Terminology requests do not determine prices or infer procedures from diagnoses.

On first load the page shows an explicitly labeled real sample captured September 4, 2026: amitriptyline 10mg tablet ×30, $5.97 before shipping/tax. No network lookup happens automatically. Search inputs stay out of the page URL and persistent browser storage. Requests go directly to the named providers, which may keep their own logs and cache their responses.

## Attribution

Creator: Jeffrey A. Brick, UMLS license holder, approved September 3, 2026. The approval email and private credentials are not included. Public APIs are accessed without a private UMLS key. The creator's license does not transfer terminology rights to visitors. The repository's MIT license covers the application code only; data remains subject to its providers' terms. Independent project, not affiliated with or endorsed by ONC, HHS or NLM.

Initial visual consultation: one Fable 5.1 CLI take, implemented and evaluated by Codex. Its paper-receipt idea informs the navy/white/red interface, receipt edge and date stamp.
