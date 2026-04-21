# How `gnfinder` Finds Taxonomic Names

`gnfinder` combines **dictionary lookups**, **character-pattern heuristics**, and a **trained naive Bayes classifier** in a three-stage pipeline. Entry point: `Find()` in `pkg/gnfinder.go:55-87`.

## 1. Tokenization & Heuristic Gating

Source: `pkg/ent/heuristic/heuristic.go:28-67`, `pkg/ent/token/token.go:94-147`

- Text is split on whitespace into tokens.
- Only **capitalized** tokens become name candidates.
- For each candidate, up to 5 following tokens are inspected to assemble a binomial or trinomial.
- A "Latin skeleton" is extracted from each token (punctuation stripped, internal non-letters replaced) and genus-abbreviation patterns like `A.` are flagged.
- Each token is assigned one of 9 `Decision` values: `NotName`, `Uninomial`, `PossibleBinomial`, `Binomial`, `Trinomial`, and their `Bayes*` variants.

## 2. Dictionaries

Source: `pkg/io/dict/dict.go`, CSV data embedded under `pkg/io/dict/data/`

Seven lookup tables drive rule-based acceptance:

| Dictionary | Role |
|---|---|
| `InGenera`, `InSpecies`, `InUninomials` | White lists of accepted names |
| `InAmbigGenera`, `InAmbigSpecies`, `InAmbigUninomials` | Context-dependent (also valid common words) |
| `NotInUninomials`, `NotInSpecies` | Black lists (known false positives) |
| `CommonWords` | European common words — veto |
| `Ranks` | Infraspecific markers: `var.`, `ssp.`, `f.`, … |

A candidate is promoted to a binomial when the genus appears in a genera dictionary **and** the following lowercase 3+ char token appears in `InSpecies`. Ambiguous entries require both parts to co-occur in `InAmbigGeneraSp`.

## 3. Naive Bayes Classifier

Source: `pkg/ent/nlp/bayes.go`, `pkg/ent/nlp/features.go:60-125`
Training data: `pkg/io/nlpfs/data/files/{eng,deu}/bayes.json`

- **Separate models for English and German** — trained from annotated corpora with a balanced 1:10 name-to-non-name prior.
- Features per token:
  - word length
  - last 3 characters (ending trigrams like `-us`, `-ii`, `-ana`)
  - presence of a dash
  - abbreviation flag
  - dictionary membership
  - preceding rank marker (for infraspecies)
- Posterior odds are computed independently for genus, species, and infraspecies positions.

## 4. How Heuristics and Bayes Combine

Source: `pkg/ent/nlp/bayes.go:40-122`

- Heuristics are the **authoritative filter** — they decide whether something is a name at all.
- Bayes can only **upgrade** a heuristic match (e.g. `Binomial` → `BayesBinomial`) when posterior odds exceed `BayesOddsThreshold` (default **80**).
- Bayes **never downgrades** a heuristic hit.
- Net effect: dictionary rules keep precision high; Bayes adds confidence scoring and rescues marginal candidates via word-shape evidence.

## 5. Language Detection

Source: `pkg/ent/lang/language.go:80-96`

- Uses the `whatlanggo` library on the first 40 KB of input.
- Maps to `eng` or `deu`; anything else defaults to English.
- Determines which Bayes model is loaded.

## 6. Nomenclatural Annotations

Source: `pkg/ent/output/process.go:143-188`

After a name is detected, up to 2 following tokens are scanned for:

- `sp. nov.` → `SP_NOV`
- `subsp. nov.` / `ssp. nov.` → `SUBSP_NOV`
- `comb. nov.` → `COMB_NOV`
- `nom. nov.` → `NOM_NOV`

Pure pattern matching — no Bayes involved.

## Summary

| Question | Answer |
|---|---|
| Uses a dictionary? | **Yes** — 7 CSV dictionaries (white/black/ambiguous lists, ranks, common words) |
| Uses character patterns? | **Yes** — capitalization, Latin skeleton, ending trigrams, abbreviation shape |
| Has training data? | **Yes** — naive Bayes models trained separately for English and German |
| How are they combined? | Heuristics gate acceptance; Bayes upgrades confidence above an odds threshold |
