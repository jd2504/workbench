# Spec: Biological & Artificial Intelligence Research Workbench (v1: Learning)

Draft 2 · Nov 15, 2026 · revised for a local Claude Code build

## Changes in draft 2

Draft 2 fixes where the build runs and folds in the architecture review. Everything not listed here is unchanged from draft 1.

1. **Build environment.** Fable builds in Claude Code on the user's Arch machine, inside the `workbench/` repo (section "Note to Fable").
2. **Runtime project split.** Opus's Claude Code project (`CLAUDE.md`, skills, critic, `settings.json`, MCP config) moves from `workbench/.claude/` to `workbench/runtime/.claude/`. Every Opus session and `claude -p` call runs with `workbench/runtime/` as its working directory.
3. **Actor-checked write path.** Every write to `kb/` goes through one store function that takes an actor. Integrity rule 3 is enforced there, not by convention.
4. **Layer order.** Packages import in one direction only, checked by a test.
5. **Stable passage IDs.** Passage IDs are content hashes, so re-parsing a PDF does not orphan claims.
6. **Milestones split.** M5 becomes M5a, M5b, M5c. M6 becomes M6a, M6b. One session each.
7. **`wb init`.** It creates the `kb` repo and user config. The `workbench` repo already exists when `wb` first runs.
8. **Items to verify during the build,** listed in section "Items to verify during the build".

## Note to Fable (the builder)

You are building this system; Claude Opus will run it. Design every prompt, tool definition and context budget for Opus at runtime, not for yourself.

- **Build in milestones** (section "Repository layout, stack and build milestones"). Each milestone ends with passing tests and a short demo the user can run from Emacs.
- **Ask before inventing.** Where this spec is silent or ambiguous, record the question in `docs/DECISIONS.md` and pick the simplest option that keeps the data model intact.
- **Do not hard-code scientific claims.** Seed content (section "Bio-AI bridge map") is a starting set of entries with citations. It goes through the same ingestion and provenance path as everything else.
- **The user is an expert** in both neuroscience and machine learning. Optimize for precision, density and traceability, not explanation.

**Where the build runs.** Build in Claude Code on the user's Arch Linux machine, started in the root of the `workbench/` repo with Fable selected. Do not build in a cloud session: it has no Emacs 30, Docker, GROBID, Zotero or signed-in `claude`, and it cannot reach the paper APIs, so no acceptance check can run there. Run every test and demo on the machine itself before reporting a milestone as done.

**Session start.** There is no build-time `CLAUDE.md`, so the user opens each session with this prompt:

```text
Read docs/SPEC.md, docs/DECISIONS.md and docs/PROGRESS.md. Build milestone M<n> only.
Stop at its acceptance check and update docs/PROGRESS.md.
```

**Before M0** (the user's job; `wb-doctor` checks the same list from M0 onward):

- Claude Code installed and signed in with the Pro plan, with usage credits enabled for Fable
- `ANTHROPIC_API_KEY` not set in the shell
- Emacs 30.2, Docker, `uv` and `git` on the path
- A private GitHub repo for `workbench`, cloned to `~/wb/workbench`, with this file committed as `docs/SPEC.md`

**Build cost.** The user is on the Pro plan, where Fable runs on pay-as-you-go usage credits instead of plan limits ([source](https://support.claude.com/en/articles/15424964-claude-fable-models-on-your-plan)). Build one milestone per session, keep context lean, and stop at each milestone's acceptance check so the user can review before more credits are spent. The finished system runs on Opus, which Pro includes.

**Build files.** This spec lives in the repo as `docs/SPEC.md`. Record decisions in `docs/DECISIONS.md` and milestone status in `docs/PROGRESS.md`, and read both at the start of each session. Keep build-time notes out of every `CLAUDE.md`, and do not create a `CLAUDE.md` or a `.claude/` directory at the repo root: Claude Code loads those files automatically, so anything in them could also reach Opus at runtime. The only `CLAUDE.md` in the repo is `runtime/.claude/CLAUDE.md`, which holds the runtime contract described in section "Opus agent workflows".

## Purpose and goals

The workbench helps one researcher build computational models of biological intelligence, informed by artificial neural network architectures, using only published research and public data.

It has three goals of equal weight:

1. **Cross-disciplinary synthesis.** Read ML/AI papers and cognitive neuroscience papers into one knowledge base. Map phenomena, mechanisms and architectures across the two fields, always traceable to source passages.
2. **Simulation.** Implement candidate mechanisms as small runnable models and test whether they reproduce published behavioral findings.
3. **Model investigation.** Probe trained ML models (for example JEPA-family encoders, transformers, networks trained with biologically plausible learning rules) for how they encode information. Compare their representations and behavior with biological data and ask which architectural features are biologically plausible.

The system must run locally from Emacs with no ongoing cost beyond the user's existing Claude subscription.

## Scope of v1

Version 1 covers **learning**, at the **behavioral and computational** levels (Marr's computational and algorithmic levels). It has two strands: how brains learn representations without labels, and how they assign credit when learning. Learning rules are studied as algorithms, not as biophysics. The architecture must not be learning-specific: memory (phase 2, kept in the appendix), attention and later circuit-level work are added as new seed content and benchmarks, not new subsystems.

**In scope**

- **Representation and self-supervised learning:** latent-prediction (JEPA-style), contrastive and redundancy-reduction objectives; learning from temporal continuity; statistical learning of sequences; how unsupervised exposure shapes later learning.
- **Credit assignment and plasticity:** backprop as the reference; feedback alignment; predictive coding; three-factor rules with eligibility traces; dendritic and burst-based error signals; local self-supervised rules; whether a learning rule can be identified from observable data.
- Benchmarks from published behavioral findings, animal and human learning curves (including public training data), and representational alignment with public neural and behavioral datasets.
- User-supplied data: CSV or NWB files the user adds to a project.

**Not in v1**

- Memory as a focus area. Its seed content is kept in the appendix for phase 2.
- Collecting new experimental data, or any feature that assumes access to imaging hardware.
- Biophysical or spiking-neuron simulation. Learning rules run in rate-based networks.
- Training large models from scratch. v1 trains small models on CPU or a single cloud GPU, and uses pretrained public checkpoints.
- A web UI, multi-user support, or cloud hosting beyond the optional GPU runner.

## Key decisions and constraints

At runtime, Opus is reached only through the official Claude Code CLI, signed in with the user's Claude plan. That keeps ongoing cost at zero beyond the subscription. Everything else is free, open-source and local.

| Area | Decision | Why |
| --- | --- | --- |
| Build environment | Fable in Claude Code on the user's Arch machine, in the `workbench/` repo root | Acceptance checks need Emacs, Docker, Zotero, the paper APIs and a signed-in `claude`, which only the user's machine has. |
| Runtime model | Claude Opus (`claude-opus-5-5`), pinned with `--model` | The user's requirement. Pro includes Opus ([source](https://claude.com/pricing)). Record the model ID and prompt version on every LLM output. |
| How Opus is called | Interactive: Claude Code sessions opened from Emacs. Background jobs: `claude -p` subprocess calls. Both run with `workbench/runtime/` as the working directory. | Both run under the user's own Claude plan with no API key. Claude Code's support docs say `claude -p` currently draws from plan usage limits ([source](https://support.claude.com/en/articles/15036540)). The separate working directory keeps Opus's permissions and contract out of build sessions. |
| Backend abstraction | One `LLMBackend` interface with two implementations: `ClaudeCodeCLI` (default) and `AnthropicAPI` (off by default) | Anthropic announced, then paused, a move of `claude -p` usage to a separate monthly credit ([source](https://code.claude.com/docs/en/headless.md)). If billing changes again, switching is one config line. |
| Credential hygiene | Never read, copy or proxy OAuth tokens. Unset `ANTHROPIC_API_KEY` when calling the CLI backend. | If `ANTHROPIC_API_KEY` is set, Claude Code uses it instead of the plan and bills API rates ([source](https://support.claude.com/en/articles/11145838-)). |
| Usage budget (Pro plan) | Background jobs default to about 4 Opus calls a week, run overnight Sunday. Extraction is one call per paper. Every job has hard caps and logs to `kb/usage.org`. | Pro has a five-hour limit and a weekly limit, and chat and Claude Code draw from the same pool ([source](https://claude.com/pricing)). Opus 5.5 has a 1M-token context window in Claude Code, so a whole paper fits in one call ([source](https://support.claude.com/en/articles/8606394-how-large-is-the-context-window-on-paid-claude-plans)). |
| Source of truth | Plain-text files in a git repo (Org + JSONL). SQLite and a vector index are derived and can be rebuilt. | Diffable history of every claim and hypothesis; readable and editable in Emacs; free backup to any git remote. |
| Write path | One store function writes to `kb/`, and it takes an actor (`user`, `opus`, `sweep`, `system`) | Makes "only the user can accept" a code-level rule. Section "Data model" has details. |
| Interface | Emacs 30, with Org-mode as the main surface | The user runs Emacs 30.2. Section "Emacs interface" has details. |
| Local compute | Python 3.12, `uv`, PyTorch; CPU only, on Arch Linux | The user has no local GPU. GROBID, embeddings and the small v1 learning models all run on CPU; full-size video self-supervised training uses the optional GPU runner. |
| Remote GPU (optional) | `wb run --remote gcp` starts a Spot VM with one GPU, runs the job, copies results back, then deletes the VM | The only cost outside the plan. Off by default; every run shows a cost estimate and needs the user's confirmation (section "Modeling sandbox"). |
| Paper metadata | arXiv, bioRxiv/medRxiv, PubMed E-utilities, Crossref, Semantic Scholar, OpenAlex | All free. OpenAlex now needs a free key and gives $1 of usage per day; lookups by DOI are free ([source](https://developers.openalex.org/api-reference/authentication)). |
| PDF parsing | GROBID (local Docker) primary; Docling as fallback for tables and figures | Both open-source and local. GROBID is built for scholarly PDFs and was much faster than Docling on scholarly PDFs in one comparison ([source](https://github.com/docling-project/docling/discussions/622)). |
| Embeddings | Local scientific-text embedding model via `sentence-transformers`, on CPU. Choose by benchmark at M1 between SPECTER2 and at least one model trained for query-to-passage retrieval. | Free and offline; no LLM calls needed for search or sweep ranking. SPECTER2 is tuned for paper-level similarity, so it may not be the best choice for passage search. |

## System architecture

The system is a local Python core with two front doors: Emacs for the user, and an MCP server for Opus. Opus is reached in only two ways, interactive Claude Code sessions and bounded `claude -p` jobs, and both go through the official CLI.

```text
 Emacs (user)                Claude Code session          Weekly scheduler
 wb.el + Org buffers         claude-code.el in Emacs      systemd user timer
 reads kb/ files directly    cwd: workbench/runtime/      bounded Opus budget
        |                            |                            |
        | --json calls               | interactive                | claude -p
        v                            v                            v
 wb core (Python)  --extract (claude -p)-->  Opus (claude-opus-5-5)
 CLI + MCP server  <------MCP tools-------   via Claude Code CLI only,
 ingest, extract,                            user's plan, no API key
 bridge, sandbox, sweep
        |                     |                         |
        | read, write         | query                   | parse, index, run
        v                     v                         v
 kb/ git repo  --rebuild-->  Derived index          Local compute
 Org + JSONL                 SQLite + sqlite-vec    GROBID, embeddings,
 (source of truth)           (rebuilt from kb/)     PyTorch sandbox
        ^                                               ^      (optional GCP
        | metadata, OA PDFs                             |       GPU runner)
 Free paper APIs + Zotero                        Datasets: THINGS, IBL,
 arXiv, bioRxiv, PubMed,                         Brain-Score, Allen, NSD,
 Crossref, S2, OpenAlex                          user CSV / NWB / BIDS
```

The core calls Opus for extraction, and Opus calls back into the core through MCP tools. Shaded boxes are external, free sources; everything else runs on the user's machine.

**Layer order.** Packages under `src/wb/` import in one direction only. A test (`import-linter` or equivalent) fails the build on any import that runs the other way.

1. `core`
2. `store`
3. `ingest`, `llm`, `bridge`
4. `extract`, `sweep`, `sandbox`
5. `cli`, `mcp_server`

Each layer imports only from the layers above it in this list. `wb.el` talks only to the CLI's `--json` output and reads `kb/` files; it never imports or calls Python directly.

**Two Claude Code projects in one repo.** The repo root is Fable's build project and has no `.claude/` directory. `workbench/runtime/` is Opus's project: its `.claude/` holds the runtime contract, skills, critic subagent and permissions, and its `.mcp.json` registers `wb mcp`. `wb-ask`, `wb-synthesize` and the `ClaudeCodeCLI` backend always start `claude` with `workbench/runtime/` as the working directory. Extraction calls also disable tools, since they need none.

## Data model

Every statement the system makes must trace back to a source passage, a simulation run or a user note. The model below enforces that. Define each entity as a Pydantic schema in `src/wb/core/schemas.py`. Each schema maps to a file format on disk and to tables in the derived SQLite index.

| Entity | Key fields | Stored as |
| --- | --- | --- |
| Paper | citekey, DOI/arXiv/PMID, title, authors, year, venue, field (`neuro`, `ml`, `both`), open-access status, PDF path, parse status, added by (`user`, `sweep`) | `kb/papers/<citekey>.org` (property drawer + notes) |
| Passage | ID (citekey + hash of the normalized text), paper, section, page, character span, text hash | `kb/passages/<citekey>.jsonl` |
| Claim | type (`empirical`, `modeling`, `theoretical`, `method`), normalized statement, passage IDs, quoted span, linked phenomena/mechanisms/architectures, quantitative fields (measure, value, uncertainty, n, species, paradigm), extractor (model + prompt version), review status (`unreviewed`, `accepted`, `edited`, `rejected`) | `kb/claims/<citekey>.jsonl` |
| Phenomenon | name, definition, level (`behavioral`, `computational`, `neural`), paradigms, aliases | `kb/ontology/phenomena.org` |
| Mechanism | computational principle, such as prediction in latent space or random-feedback credit assignment | `kb/ontology/mechanisms.org` |
| Architecture | an AI model family, such as JEPA, predictive coding networks or feedback-alignment networks | `kb/ontology/architectures.org` |
| Bridge link | from → to, relation (`analogy`, `formal-equivalence`, `empirically-compared`, `proposed`), claims for, claims against | `kb/bridge/links.jsonl` |
| Hypothesis card | statement, bridge links, testable predictions, falsifiers, status (`proposed`, `testing`, `supported`, `refuted`, `parked`), experiments, change log | `kb/hypotheses/H-NNNN.org` |
| Benchmark | phenomenon, data source (claims or dataset), metric, target values with uncertainty | `kb/benchmarks/<id>.yaml` |
| Experiment run | hypothesis, model config, benchmarks, code commit, seed, metrics, figures, Opus critique | `kb/experiments/<id>/` (README.org, config, results) |
| Dataset | source, license and access terms, local path, format (NWB, BIDS, CSV) | `kb/datasets/<id>.org` |

**Passage IDs.** A passage's ID is its citekey plus a hash of its whitespace-normalized text, not its position in the parse. Re-parsing a PDF with a newer GROBID or with Docling then keeps the ID of every passage whose text is unchanged. When a re-parse drops a passage that claims point to, `wb` keeps the old passage record, marks it `superseded` and reports the affected claims; it never deletes a passage that a claim cites.

**Provenance labels.** Every sentence the system writes into a report, digest or answer carries one of four labels. Org exports render each label as a tag or link.

- `REPORTED`: from a paper; links to the passage.
- `INFERRED`: the system's synthesis; links to the claims it rests on.
- `SIMULATED`: from a run; links to the experiment.
- `USER`: the user's own note or data.

**Integrity rules**, enforced in code and tested:

1. A claim without at least one passage ID is rejected.
2. A claim's quoted span must appear in its passage text, after whitespace normalization.
3. Only the user can mark a claim `accepted` or move a hypothesis to `supported` or `refuted`. Opus can propose these changes; it cannot apply them.
4. Ontology entries (phenomena, mechanisms, architectures) are merged by alias. Opus proposes merges; the user approves them.

**Write path and actors.** All code that changes `kb/` calls one function in `src/wb/store/`, and every call names an actor: `user`, `opus`, `sweep` or `system`. Rule 3 is enforced there, in four parts.

- **State machines.** Claim review status and hypothesis status each have a transition table keyed by actor. The store rejects any transition the actor is not allowed to make. Only `user` may set `accepted`, `edited`, `rejected`, `supported` or `refuted`, or approve an ontology merge or bridge link.
- **Fixed actor per front door.** The MCP server hard-codes `opus`. The sweep hard-codes `sweep`. The CLI commands that Emacs calls for review use `user`. No tool argument or environment variable can change the MCP server's actor.
- **Proposals.** What Opus proposes is stored as a proposal (`unreviewed` claims, `proposed` links and merges, a `proposed_status` field on a hypothesis card) for the user to apply in `wb-review` or the card's Org buffer.
- **Permissions.** `runtime/.claude/settings.json` denies Opus direct edits to files under `kb/` and denies shell calls to `wb`, so the MCP tools are its only write route. Each `kb` commit records the actor in its message, so `git log` is the audit trail.

## Paper ingestion pipeline

A paper enters the system in one command (`wb add <DOI | arXiv ID | URL | path.pdf>`, or `M-x wb-add-paper`). It comes out as passages, local embeddings and Opus-extracted claims waiting for the user's review. Steps 1–5 use no LLM and cost nothing. Step 6 is the only Opus call.

1. **Resolve metadata.** Look up the identifier in arXiv, Crossref, PubMed, Semantic Scholar or OpenAlex, in that order of preference. Assign a citekey such as `lillicrap2016random`. Skip duplicates by DOI or arXiv ID.
2. **Get the full text.** Use the local PDF if given. Otherwise fetch a legal open-access copy (arXiv, bioRxiv, PubMed Central, Unpaywall-style OA links from OpenAlex). If none exists, ingest the abstract only and mark the paper `abstract-only`, so the user can drop in a PDF later.
3. **Parse.** GROBID turns the PDF into sections, references and figure captions as TEI XML, with coordinates turned on so each paragraph carries its page. Docling is the fallback when GROBID fails or tables matter.
4. **Split into passages.** Use paragraph-level passages with section, page and character offsets into the section's normalized text. Hash each passage's normalized text and derive its ID from that hash (section "Data model"). These are the provenance anchors.
5. **Embed and index.** Embed passages locally. Store the vectors in SQLite with `sqlite-vec` and add the paper's references to a citation graph.
6. **Extract claims (Opus).** Send the whole paper's passages in one call, with the extraction prompt and the current ontology. Split by section only if a paper exceeds a configured token budget. Validate the returned JSON against the Claim schema, check every quoted span against its passage, and retry once on failure. Link claims to existing phenomena, mechanisms and architectures, or propose new ontology entries.
7. **Queue for review.** Write `kb/papers/<citekey>.org` with a summary, the extracted claims and the proposed ontology changes. The user accepts, edits or rejects them in Emacs.

**Zotero integration.** Zotero stays the user's reference manager; `wb` reads from it and never edits its database file.

- **Read (default):** Zotero's desktop app serves a local copy of its web API on `localhost:23119/api/`, and read requests need no authentication ([source](https://www.zotero.org/support/dev/web_api/v3/local_api)). `wb zotero sync` pulls items from chosen collections and ingests them. It reuses the PDFs already in Zotero storage instead of downloading them again.
- **Citekeys:** if Better BibTeX is installed, use its citekeys so `wb` and the user's existing `.bib` files agree. Otherwise generate `authorYEARword` keys.
- **Write back (optional):** papers added through `wb` or the weekly sweep can be pushed into a `wb-inbox` Zotero collection. Local-API writes need Zotero 10+ and a local API key the user grants through a confirmation dialog (same source). Off until the user turns it on.
- **Sync rule:** a Zotero item and a `wb` paper are matched by DOI, then arXiv ID, then title and year. Metadata edits in Zotero flow into `wb`; `wb` never changes Zotero metadata.

**Other inputs.** Bulk import from a BibTeX file. A watched `inbox/` folder for dropped PDFs. Papers the weekly sweep picks (section "Opus agent workflows").

**User data.** `wb data add <path>` registers a CSV, NWB or BIDS dataset with a short Org description. The sandbox can then load it like any public dataset.

## Bio–AI bridge map and learning seed content

The bridge map is the system's core knowledge structure. It links biological phenomena to computational principles to AI architectures, and every link carries a relation type and evidence. v1 starts with the twelve learning links below: six on representation learning and six on credit assignment. They are **seed entries, not facts**. At milestone M4, Fable resolves each reference through the metadata APIs and ingests it through the normal pipeline. The relation label is then confirmed or corrected from the extracted claims. Flag any reference that cannot be resolved; do not write claims from memory.

| Strand | Biological phenomenon | Computational principle | AI analogue | Seed relation | Seed references |
| --- | --- | --- | --- | --- | --- |
| Representation | Invariant object recognition learned from temporal continuity | Slowness and temporal-contiguity objectives | Video self-supervised learning (V-JEPA); time-contrastive objectives | empirically-compared | Wiskott & Sejnowski 2002; Li & DiCarlo 2008; Zhuang et al. 2021 |
| Representation | Cortex predicts its sensory input | Prediction in latent space rather than reconstruction | JEPA (I-JEPA, V-JEPA 2); predictive coding networks | analogy | Rao & Ballard 1999; LeCun 2022; Assran et al. 2023 |
| Representation | Ventral-stream representations form without labels | Self-supervised objectives predict neural data about as well as supervised ones | Self-supervised vision models scored on neural benchmarks | empirically-compared | Zhuang et al. 2021; Konkle & Alvarez 2022 |
| Representation | Redundancy reduction in sensory coding | Decorrelation; variance and covariance regularization | Barlow Twins; VICReg; LeJEPA's SIGReg | analogy | Barlow 1961; Zbontar et al. 2021; Bardes et al. 2022; Balestriero & LeCun 2025 |
| Representation | Statistical learning of sequences in infants | Learning transitional probabilities by prediction | Next-token prediction; simple recurrent networks | empirically-compared | Saffran, Aslin & Newport 1996; Elman 1990 |
| Representation | Unsupervised exposure shapes later learning | Pretraining, then task learning | Self-supervised pretraining with fine-tuning | empirically-compared | Zhong et al. 2025 (mouse visual learning); Orhan et al. 2020 (child headcam video) |
| Credit assignment | No symmetric feedback weights in cortex | Random feedback can still train deep networks | Feedback alignment; direct feedback alignment | empirically-compared | Lillicrap et al. 2016; Nøkland 2016 |
| Credit assignment | Error-like signals in cortical activity | Predictive coding approximates backprop | Predictive coding networks; prospective configuration | formal-equivalence | Whittington & Bogacz 2017; Millidge, Tschantz & Buckley 2022; Song et al. 2024 |
| Credit assignment | Neuromodulator-gated plasticity | Three-factor rules with eligibility traces | e-prop; node perturbation; REINFORCE | analogy | Frémaux & Gerstner 2016; Bellec et al. 2020 |
| Credit assignment | Apical dendrites and bursting | Error signals carried in separate dendritic compartments | Dendritic and burst-dependent approximations of backprop | proposed | Guerguiev, Lillicrap & Richards 2017; Sacramento et al. 2018; Payeur et al. 2021 |
| Both | Local plasticity builds deep representations | Local self-supervised contrastive prediction | CLAPP; Greedy InfoMax | empirically-compared | Illing, Ventura, Bellec & Gerstner 2021; Löwe, O'Connor & Veeling 2019 |
| Both | Which learning rule does cortex use? | Learning rules leave signatures in activity and weight statistics | Classifiers trained to identify learning rules | proposed | Nayebi et al. 2020; Richards et al. 2019; Lillicrap et al. 2020 |

**Seed hypothesis cards.** Three cards make the full loop testable end to end, one per strand plus one that joins them:

1. **H-0001 (representation).** A small video encoder trained with latent prediction (JEPA-style) shifts its position tolerance after "swapped" object-identity exposure in the same direction as monkey IT did in Li & DiCarlo 2008. A frame-shuffled control does not shift. *Test:* recreate the swap-exposure manipulation in silico and compare the direction and relative size of the change. *Falsifier:* no difference from the control, or a shift in the opposite direction.
2. **H-0002 (credit assignment).** Backprop, feedback alignment, predictive coding and a three-factor rule leave learning-dynamics signatures that stay classifiable when only a few hundred noisy units are observed, extending Nayebi et al. 2020. *Test:* train small networks with each rule, subsample and add noise, then classify the rule. *Falsifier:* classification falls to chance at realistic subsampling.
3. **H-0003 (both).** A network trained with a local self-supervised rule (CLAPP-style, or predictive coding with a variance regularizer) matches a size-matched, backprop-trained JEPA in alignment with THINGS human similarity judgments. *Test:* odd-one-out accuracy and RSA against THINGS-data, against an untrained network as the floor. *Falsifier:* the local rule's alignment sits closer to the untrained network than to the backprop model.

**H-0003 at CPU scale.** A JEPA trained on CPU is very small, and both trained models may sit near the untrained floor. The M5c run must report the gap between the backprop model and the floor with its uncertainty across seeds. If that gap is not clearly above zero, the run is recorded as uninformative, the card stays `testing`, and H-0003 is rerun at a larger size on the GPU runner at M6b.

**Seed reading list.** All references in the table, plus 10–15 reviews chosen by Opus from the citation graph after the first ingestion pass. The user approves the list before extraction runs.

## Modeling sandbox, benchmarks and datasets

The sandbox turns a hypothesis card into a run whose result can be checked against a benchmark. Models are small, seeded and reproducible on a CPU-only laptop. Jobs that need a GPU (mainly video self-supervised training and model investigation at M6) go to an optional, pay-per-use GCP runner, described at the end of this section.

**Component library** (`src/wb/sandbox/components/`, PyTorch modules with one shared interface):

- **Learning rules**, behind one optimizer-like interface: backprop (the reference), feedback alignment and direct feedback alignment, predictive coding (iterative inference plus local updates), three-factor rules with eligibility traces and node perturbation, Hebbian and Oja-type rules, local contrastive rules (CLAPP-style), and difference target propagation
- **Self-supervised objectives:** JEPA-style latent prediction with a LeJEPA-style regularizer, contrastive (InfoNCE), redundancy reduction (Barlow Twins, VICReg), slowness, next-step prediction
- **Small architectures:** MLPs, small CNNs, a small ViT and simple recurrent networks, all instrumented the same way
- **Locality declaration:** each rule states what each weight update reads (local activity, a global scalar, a separate feedback pathway, transported weights, a separate phase). An automated audit reports these in every run.
- **Training-dynamics recorder:** saves activations, weights and update statistics at checkpoints, so representational change over learning can be analyzed

**Reference reproductions.** Every rule and objective is compared with a backprop-trained version of the same architecture at matched compute. Two published results must be reproduced, within stated tolerance of their source papers, before any new comparison is trusted. One is feedback alignment learning a standard task close to backprop (Lillicrap et al. 2016). The other is a next-step predictor telling words from part-words in a Saffran-style syllable stream.

**Task simulators:**

- Statistical-learning streams (syllables and visual shape sequences) with word and part-word tests
- Temporal-contiguity exposure: normal and "swapped" object identity across time, after Li & DiCarlo 2008
- Few-shot category learning
- A visual decision task with the same structure as the IBL mouse task, for comparing learning curves
- Standard small tasks for learning rules: MNIST, Fashion-MNIST, CIFAR-10

All tasks share one trial-and-response format, so model and animal or human data use the same analysis code.

**Benchmarks for v1:**

- **Behavioral alignment:** THINGS odd-one-out accuracy relative to the human noise ceiling
- **Neural alignment:** encoding and RSA scores on public Brain-Score benchmarks, THINGS-fMRI/MEG and the Allen Brain Observatory
- **Statistical learning:** word versus part-word discrimination, and how learning curves compare with published infant and adult results
- **Invariance learning:** direction and relative size of the tolerance change after swap exposure
- **Learning-stage alignment:** representational change from unsupervised exposure versus task training, compared with the Zhong et al. 2025 mouse data
- **Credit-assignment gap:** accuracy and sample efficiency relative to backprop at matched architecture and compute, plus the locality audit
- **Rule identifiability:** accuracy when classifying the learning rule from subsampled, noisy activity
- **Learning curves:** trials to criterion and psychometric changes over training, compared with IBL mice

Each benchmark stores its target values with uncertainty. Targets come from accepted claims or from a public dataset.

**Fitting and comparison.** Fit with a summary-statistic loss plus Bayesian optimization (Optuna). Report parameter uncertainty across seeds. Compare models on held-out subjects or lists, with a penalty for free parameters.

**Model-investigation toolkit** (goal 3):

- Activation hooks for PyTorch and Hugging Face models
- Representational similarity analysis (`rsatoolbox`) and CKA, applied across training checkpoints as well as between models
- Linear and nonlinear probes; ablation and lesion utilities
- Ridge-regression encoding models to map model features onto neural data
- Brain-Score's public benchmarks run locally as an extra scorer
- Starting checkpoints: the distilled V-JEPA 2.1 encoders, and public self-supervised and supervised vision models of matching size

**Public datasets** (adapters at M5c–M6a; access terms checked and recorded in `kb/datasets/`):

| Dataset | Use in v1 | Access |
| --- | --- | --- |
| [THINGS-data](https://vicco-group.github.io/THINGS-data) | About 4.7M odd-one-out similarity judgments over 1,854 object concepts, plus fMRI and MEG on the same images | Open: behavior on OSF, raw fMRI and MEG on OpenNeuro, derivatives on Figshare |
| [Brain-Score vision](https://github.com/brain-score/vision) | Standard neural and behavioral alignment benchmarks | Open source; running locally gives the public benchmarks only, the full set needs a website submission |
| [IBL behavior release](https://int-brain-lab.github.io/iblenv/notebooks_external/data_release_behavior.html) | Mice learning a visual decision task from novice to expert (198 mice) | Open, through the ONE API |
| Zhong et al. 2025 mouse visual learning | Recordings of up to 80,000 neurons in visual cortex at different stages of visual learning ([dataset guide](https://instructor.compneuro.neuromatch.io/projects/neurons/README.html)) | Open; confirm the host at M5c |
| Allen Brain Observatory | Mouse visual cortex responses for neural alignment | Open |
| Natural Scenes Dataset | 7T fMRI to natural images for neural alignment | Data-use agreement |
| SAYCam (child headcam video) | Developmentally realistic training data for self-supervised models | Restricted; access request needed; optional |

**Code safety.** Opus writes experiment configs freely. New component code goes only in `sandbox/contrib/` on a git branch. Runs execute in a subprocess with time and memory limits. The user reviews the diff before merging into the main library.

**Remote GPU runner (GCP, optional).** `wb run <config> --remote gcp` runs the same experiment on a short-lived GCP GPU VM. Every run works the same way:

1. Check the config is marked `needs_gpu` or that the user passed `--remote`. Remote runs are never automatic, including when Opus requested the run.
2. Show the machine type, GPU, region, maximum runtime and a cost estimate from current GCP pricing, then wait for the user's confirmation in Emacs.
3. Upload the config, code at a pinned commit and the needed data subset to a GCS bucket. Start a Spot VM from a Deep Learning VM image with one GPU (L4 by default, T4 as the cheaper option) using `gcloud`.
4. Run the job in a container built from the same lockfile as the local environment, so results are identical apart from hardware nondeterminism.
5. Copy results to `kb/experiments/<id>/`, then delete the VM. A startup-script watchdog shuts the VM down at the maximum runtime or after 15 idle minutes, even if the local machine goes offline.
6. Log the actual runtime and estimated cost to `kb/usage.org` alongside Opus usage.

If a Spot VM is preempted, the runner resumes from the last checkpoint once, then reports the failure. Configuration lives in `~/.config/wb/gcp.toml`: project, region, bucket, default GPU and a monthly spend cap that the runner refuses to exceed. Setting up the GCP project and billing is the user's job; the runner only uses credentials already set up with `gcloud auth`.

## Opus agent workflows

Opus does six jobs. Each one runs either inside a Claude Code session started in `workbench/runtime/`, or as a bounded `claude -p` call from the same directory. Opus can read and propose anything. It never applies status changes the user owns.

| Workflow | Trigger | How Opus runs | Output | User gate |
| --- | --- | --- | --- | --- |
| Extract | A paper is added | `claude -p`, one call per paper, tools disabled, JSON validated against the schema | Claims and proposed ontology entries | Review in the paper's Org file |
| Ask | User question | Interactive Claude Code session using the `wb` MCP tools | Answer with provenance labels and links | None; answers are not stored unless the user saves them |
| Synthesize | `M-x wb-synthesize <topic>` | Interactive session or `claude -p` | `kb/reports/<topic>.org`: cross-field review with REPORTED / INFERRED labels, open questions and proposed bridge links | Accept the proposed links |
| Hypothesize and simulate | From a bridge link or report | Interactive session: draft card, write config, call `run_experiment`, read results | Hypothesis card, experiment folder, critique | Approve contrib code; set the card's status |
| Critique | After any synthesis or run, or on demand | A separate subagent with fresh context that searches the knowledge base for contrary claims | A critique section: overclaiming, missing counter-evidence, alternative explanations, the next experiment that would discriminate | None |
| Weekly sweep | systemd user timer, weekly | `claude -p` with a hard cap on calls | `kb/digests/YYYY-Www.org` plus an Atom feed | Choose which papers to ingest fully |

**Weekly sweep, step by step.** Steps 1–2 are free and local; only steps 3–5 use Opus.

1. **Fetch** the last seven days of candidates. Sources: arXiv (`q-bio.NC`, `cs.NE`, `cs.LG`, `cs.AI` with keyword filters), bioRxiv (neuroscience category), PubMed saved queries, and OpenAlex citations to papers already in the knowledge base.
2. **Rank** candidates locally by embedding similarity to an interest profile. The profile is built from accepted claims, open hypothesis cards and user-pinned keywords and authors.
3. **Triage** the top 25 abstracts with Opus in one batched call. Score relevance and novelty against the knowledge base, with a one-line reason each.
4. **Extract** claims from the top 3 open-access papers, one call each. Mark them `unreviewed`.
5. **Write the digest.** It opens with a short summary of the week, then each paper with its key claims, which hypotheses it bears on (supports, contradicts or is relevant to), and suggested bridge links.
6. **Log usage** to `kb/usage.org`. The Pro defaults (25 triaged, 3 extracted) come to about 4 Opus calls a week. The sweep runs overnight Sunday so it doesn't compete with daytime use. All caps are config values.

**Stopping early near the weekly limit.** The sweep should stop before it uses the last of the week's plan allowance. At M2, check whether remaining plan usage can be read from a headless `claude` call. If it can, the sweep checks it before each Opus call. If it cannot, the sweep relies on its own ledger in `kb/usage.org` and its call cap, and it stops at the first limit error from the CLI, writing a partial digest that says so.

**Starter sweep profile.** The profile below is a first suggestion; the user will refine it later. It lives in `kb/sweep.org`, so it can be edited in Emacs, and every digest shows which rule surfaced each paper so noisy rules are easy to find and remove.

| Filter | Starter entries |
| --- | --- |
| arXiv categories | `q-bio.NC` and `cs.NE` in full; `cs.LG`, `cs.AI` and `stat.ML` only when a keyword below matches |
| Preprint servers | bioRxiv Neuroscience and Animal Behavior and Cognition; PsyArXiv through the OSF API (for statistical learning) |
| Representation keywords | self-supervised learning, representation learning, predictive learning, statistical learning, unsupervised learning, perceptual learning, invariance learning, temporal contiguity, slowness, developmental learning |
| Credit-assignment keywords | credit assignment, synaptic plasticity, learning rule, biologically plausible learning, backpropagation and the brain, feedback alignment, predictive coding, three-factor rule, eligibility trace, dendritic learning, burst-dependent plasticity, local learning, Hebbian learning, neuromodulation |
| AI keywords | JEPA, latent prediction, contrastive learning, redundancy reduction, VICReg, Barlow Twins, world model, target propagation, equilibrium propagation, forward-forward |
| Bridge boost | Rank higher when a paper matches a representation or credit-assignment keyword and an AI keyword at once |
| Authors | Authors of the seed references (for example Lillicrap, Richards, Bogacz, Gerstner, Sacramento, Nayebi, Yamins, DiCarlo, Konkle, Saffran, Pachitariu, Balestriero, LeCun, Assran). Track them by OpenAlex author ID, not name, to avoid false matches. |
| Citations | New papers that cite any seed reference or any paper the user has accepted claims from (via OpenAlex) |
| Venues | Neuron, Nature Neuroscience, Nature Human Behaviour, eLife, PNAS, Neural Computation, PLOS Computational Biology, Journal of Neuroscience, Trends in Cognitive Sciences, Nature Machine Intelligence; NeurIPS, ICLR, ICML, TMLR, CCN |

**Claude Code runtime project** (Fable builds and versions all of it under `workbench/runtime/`):

- `runtime/.claude/CLAUDE.md`: the runtime contract for Opus (provenance labels, integrity rules, when to say evidence is missing, the user's expertise level). It contains nothing about how the system was built.
- `runtime/.mcp.json`: registers the MCP server (`wb mcp`, Python), which exposes `search_passages`, `get_paper`, `get_claims`, `query_bridge`, `propose_claim`, `propose_link`, `create_hypothesis`, `run_experiment`, `get_run`, `list_datasets`, `load_dataset_summary`. Every write tool runs as actor `opus`.
- `runtime/.claude/skills/`: `extract`, `synthesize`, `hypothesize`, `sweep-triage`.
- `runtime/.claude/agents/critic.md`: the `critic` subagent.
- `runtime/.claude/settings.json`: permissions that allow the MCP tools and `sandbox/contrib/` writes; deny direct edits under `kb/`; deny shell calls to `wb`; and limit Claude's web tools to the paper API domains. These rules govern Claude's own tools. They do not by themselves stop a shell command from reaching the network, so shell access for Opus is limited to an allowlist of commands the sandbox needs, and any broader sandboxing is recorded in `docs/DECISIONS.md` at M3.
- Prompts in `prompts/`, versioned. The version is recorded on every output.

## Emacs interface

The interface is a small Emacs package, `wb.el`, layered on Org-mode. Because the knowledge base is plain Org and JSONL files, most reading and editing is ordinary Emacs. `wb.el` adds commands that call the `wb` CLI asynchronously (`make-process`, `--json` output) and open Claude Code sessions. There is no server to keep running.

**Claude sessions.** Integrate with an existing Claude Code package rather than writing a new terminal layer. Support [claude-code.el](https://github.com/stevemolitor/claude-code.el) (on MELPA; eat, vterm or ghostel backends) by default, and [claude-code-ide.el](https://github.com/manzaltu/claude-code-ide.el) as an option, since it can expose Emacs itself to Claude through MCP. `wb-ask` opens a session in `workbench/runtime/`, pinned to Opus, and pre-loads context from the current buffer (paper, claim, hypothesis or region). The user's Fable build sessions are separate: they start in the repo root and are not opened by any `wb` command.

| Command | What it does |
| --- | --- |
| `wb-dispatch` | Transient menu for everything below |
| `wb-add-paper` | Ingest a DOI, arXiv ID, URL or PDF; also works from the URL at point in eww or elfeed |
| `wb-search` | Semantic and keyword search over passages and claims, with `completing-read` (works with vertico and consult) |
| `wb-review` | Step through unreviewed claims: accept, edit or reject with one key each; edits are written back to JSONL as actor `user` |
| `wb-goto-source` | Jump from any claim or REPORTED sentence to the passage, opening the PDF at its page in pdf-tools, or in the built-in DocView if pdf-tools is not installed |
| `wb-ask` | Open a Claude session with the context at point |
| `wb-synthesize` | Run a synthesis on a topic or ontology entry and open the report |
| `wb-bridge` | Browse the bridge map as an Org outline (phenomenon → principle → architecture) with evidence counts; export to Graphviz SVG |
| `wb-hypothesis-new` / `wb-run` | Create a hypothesis card; launch or inspect an experiment run, with figures inline in Org |
| `wb-digest` | Open the latest weekly digest; mark papers for full ingestion |
| `wb-usage` | Show the Opus usage ledger for the week |

**Compatibility.** Every Org entry gets an `:ID:` property, so org-roam users get backlinks for free. `wb export-bib` writes `kb/library.bib` for citar and org-cite. Digests are also published as an Atom file for elfeed. The package targets Emacs 30+ (the user runs 30.2), byte-compiles cleanly and passes `checkdoc`.

**Setup.** The user doesn't use org-roam, citar, pdf-tools or elfeed yet, so every command must work without them. The README gives one `use-package` block, using Emacs 30's built-in `:vc` keyword, that installs:

- **Required:** `wb.el`, and claude-code.el with its default eat backend (a terminal written entirely in Elisp, so nothing to compile). Org and transient already ship with Emacs 30.
- **Recommended:** pdf-tools, for exact page jumps and passage highlighting from `wb-goto-source`; vertico and consult, for faster search.
- **Optional:** citar (citations), elfeed (digest feed), org-roam (backlinks).

`M-x wb-doctor` reports which of these are installed, what each one adds, and whether the `wb` CLI, `claude`, Docker and `gcloud` can be found. It also reports whether `claude` is signed in and whether `ANTHROPIC_API_KEY` is set.

## Repository layout, stack and build milestones

There are two git repos. The **workbench** repo holds the code. The **kb** repo holds the user's research data, so the knowledge base can be backed up, shared or reset without touching the code. Default locations are `~/wb/workbench` and `~/wb/kb`; the `kb` path is a value in `~/.config/wb/config.toml`.

```text
workbench/                  # Fable's build project: no CLAUDE.md, no .claude/ at this level
  pyproject.toml            # uv; Python 3.12
  src/wb/
    core/                   # Pydantic schemas, config, provenance labels
    store/                  # single write path with actors, file store, SQLite + sqlite-vec index, rebuild
    ingest/                 # sources/ (arXiv, bioRxiv, PubMed, Crossref, S2, OpenAlex), parse/ (GROBID, Docling), passages, embed
    llm/                    # LLMBackend, claude_cli.py, anthropic_api.py, usage ledger
    extract/                # claim extraction + validation
    bridge/                 # ontology, links, merges
    sandbox/                # components/, models/, tasks/, benchmarks/, fit/, probe/, datasets/, contrib/
    sweep/                  # fetch, rank, triage, digest, atom
    mcp_server/             # `wb mcp`
    cli.py                  # `wb` (Typer)
  runtime/                  # Opus's Claude Code project; cwd for wb-ask and every claude -p call
    .claude/                # CLAUDE.md (runtime contract), skills/, agents/critic.md, settings.json
    .mcp.json               # registers `wb mcp`
  prompts/                  # versioned prompt files
  emacs/                    # wb.el, wb-org.el, wb-transient.el, tests/
  deploy/                   # docker-compose for GROBID; systemd timer + cron examples
  tests/                    # unit, golden papers, recorded LLM fixtures
  docs/                     # SPEC.md, DECISIONS.md, PROGRESS.md
kb/                         # separate repo: papers/ passages/ claims/ ontology/ bridge/ hypotheses/ benchmarks/ experiments/ datasets/ reports/ digests/ usage.org library.bib
```

**Backup.** The `kb` repo pushes to a private GitHub repository by default. Only text goes into git. Large and re-creatable files are kept out by `.gitignore` and backed up elsewhere:

| Content | Where it lives | Backup |
| --- | --- | --- |
| Org, JSONL, YAML, configs, small figures | `kb/` git repo | Private GitHub repo, pushed after each `wb` write session |
| PDFs | Zotero storage, referenced by path | Zotero's own sync or backup; never committed |
| Public datasets | `~/.cache/wb/datasets/` | None needed; re-downloadable, with the source and version recorded in `kb/datasets/` |
| Large run outputs (checkpoints, activations) | `~/.cache/wb/runs/` | The GCS bucket the GPU runner already uses |
| SQLite and vector index | `~/.cache/wb/index/` | None needed; rebuilt from `kb/` |

This split keeps the repo far below GitHub's limits, which hard-block single files over 100 MiB and recommend keeping repositories ideally under 1 GB ([source](https://fast.io/resources/github-file-size-limit-guide.md)). A pre-commit hook rejects any file over 5 MB. The remote is a config value, so GitLab, Codeberg or a bare repo on another disk works the same way.

**Stack:** Python 3.12, `uv`, Typer, Pydantic, SQLite + `sqlite-vec`, `sentence-transformers`, GROBID (Docker), Docling, PyTorch, NumPy/SciPy/pandas, Optuna, `rsatoolbox`, `pynwb`, `mne-bids`, matplotlib, the official MCP Python SDK, `pytest`, `import-linter`. Emacs 30+, Org, transient, and claude-code.el or claude-code-ide.el. The optional GPU runner uses the gcloud CLI and Cloud Storage. No paid service is required; GCP GPU runs are opt-in and paid per use.

**Build milestones.** Eleven sessions, one milestone each. Each ends with passing tests and a demo the user can run from Emacs. Acceptance criteria are in the next section.

1. **M0, Skeleton.** Schemas, the actor-checked write path and state machines, file store, derived index with rebuild, the layer-order test, `wb init` (creates the `kb` repo and user config) and `wb status`, `wb-dispatch` showing status, `wb-doctor`.
2. **M1, Ingestion and search.** Metadata resolution, Zotero sync (read), open-access fetch, GROBID parse, passages with content-hash IDs, the embedding-model benchmark, local embeddings, `wb-add-paper`, `wb-search`, `wb-goto-source`.
3. **M2, Extraction and review.** The `LLMBackend` with `ClaudeCodeCLI`, extraction prompt, validation, the usage ledger, the plan-usage check, `wb-review`.
4. **M3, Claude Code integration.** The `runtime/` project: MCP server, `CLAUDE.md`, skills, critic subagent, `settings.json`; `wb-ask`, `wb-synthesize`.
5. **M4, Learning seed and bridge map.** Resolve and ingest the seed list; build the twelve bridge links and the seed hypothesis cards; `wb-bridge`.
6. **M5a, Learning rules.** The learning-rule library behind one interface, small architectures, locality declarations and audit, training-dynamics recorder, experiment runner, and the feedback-alignment reproduction.
7. **M5b, Objectives and tasks.** Self-supervised objectives, the task simulators with the shared trial format, and the Saffran-style reproduction.
8. **M5c, Data and benchmarks.** THINGS, IBL and Zhong et al. adapters, CSV and NWB user-data adapters, benchmarks, fitting, `wb-run`. Run H-0002 and H-0003 end to end on CPU.
9. **M6a, Model investigation.** Probing toolkit, checkpoint loaders, BIDS adapter, Brain-Score local scoring, and H-0001 on a small video sample on CPU.
10. **M6b, GPU runner.** The GCP GPU runner, H-0001 at full size, and the H-0003 rerun if M5c recorded it as uninformative.
11. **M7, Weekly sweep.** Fetch, rank, triage, digest, Atom feed, timer install, `wb-digest`.

## Testing and acceptance criteria

The test suite must run offline and use none of the user's plan. LLM calls are replayed from recorded fixtures. A separate `pytest -m live` marker runs a small number of real Opus calls, and only when the user asks.

**Cross-cutting tests**

- **Golden papers.** Five papers (at least two neuroscience and two ML) with hand-checked claims. Extraction must recover at least 80% of the hand-checked claims, and no claim may fail provenance.
- **Provenance invariants.** Every stored claim has passages, and every quoted span is found in its passage. Every REPORTED sentence in a report resolves to a passage.
- **Actor rules.** Every status transition is tried as every actor. Only `user` can reach `accepted`, `edited`, `rejected`, `supported` or `refuted`; the same calls as `opus`, `sweep` or `system` are rejected and leave the files unchanged. No MCP tool input can change the server's actor.
- **Layer order.** No package imports from a later layer.
- **Passage stability.** Re-parsing a golden paper keeps the ID of every passage whose text is unchanged, and no claim loses its passage.
- **Index rebuild.** Deleting the SQLite index and rebuilding it from `kb/` produces the same search results.
- **Credential check.** The CLI backend refuses to run if `ANTHROPIC_API_KEY` is set, unless the API backend is selected explicitly.
- **Runtime isolation.** The repo root has no `CLAUDE.md` and no `.claude/` directory, and the CLI backend always starts `claude` in `runtime/`.
- **Emacs.** ERT tests run in batch mode; the package byte-compiles with no warnings and passes `checkdoc`.

| Milestone | Accepted when |
| --- | --- |
| M0 | `wb init` creates the `kb` repo and config; `wb status` and `wb-dispatch` work on a clean Emacs 30 machine following the README; `wb-doctor` reports every dependency correctly; the actor-rule and layer-order tests pass |
| M1 | 20 mixed papers ingest with no LLM calls, including a Zotero collection whose PDFs are reused from Zotero storage; the expected paper appears in the top 5 for 9 of 10 test queries with the chosen embedding model, and the benchmark result is recorded in `docs/DECISIONS.md`; `wb-goto-source` opens the right PDF page |
| M2 | Golden-paper thresholds pass on recorded fixtures; one live run on a new paper is reviewed in `wb-review`; the plan-usage question is answered in `docs/DECISIONS.md` |
| M3 | `wb-ask` answers three set questions with correct provenance labels, and says so when the knowledge base lacks the evidence; in a live session, Opus's attempt to edit a file under `kb/` or to call `wb` from the shell is denied; build sessions in the repo root load none of the runtime contract |
| M4 | Every seed reference is resolved or flagged; the bridge map shows twelve links with evidence counts |
| M5a | The feedback-alignment reproduction falls within stated tolerance; the locality audit flags weight transport in backprop and nothing in feedback alignment |
| M5b | The Saffran-style word-segmentation reproduction falls within stated tolerance; every objective trains a small model on CPU from one config format |
| M5c | H-0002 and H-0003 produce runs, critiques and figures in Org; the H-0003 run reports the backprop-to-floor gap and is marked informative or uninformative |
| M6a | RSA and probe results reproduce on a fixed seed; H-0001 runs on a small video sample on CPU |
| M6b | H-0001 runs at full size on the GPU runner; one confirmed GCP run completes, returns results and deletes its VM; the runner refuses a run that would exceed the monthly cap |
| M7 | A dry-run sweep produces a digest within the configured Opus call cap; the timer installs and runs once |

## Items to verify during the build

Each item is checked on the user's machine at the milestone shown, and the result goes in `docs/DECISIONS.md`.

| Milestone | Question | If the answer is no |
| --- | --- | --- |
| M1 | Does SPECTER2 meet the search target for passage queries? | Use the best query-to-passage model from the benchmark |
| M2 | Can remaining plan usage be read from a headless `claude` call? | Use the ledger, the call cap and limit-error detection |
| M3 | Does a session started in `runtime/` load only `runtime/.claude/`, with nothing from the repo root? | `wb init` installs the runtime project to `~/wb/runtime/`, outside the repo, from templates kept in `workbench/` |
| M3 | Do the `settings.json` rules stop Opus's shell commands from reaching the network? | Narrow the shell allowlist, or turn on Claude Code's sandboxing if available |
| M5c | Is a CPU-scale JEPA clearly above the untrained floor on THINGS? | Record H-0003 as uninformative and rerun at M6b |

## Open questions for the user

Where an answer is missing, Fable uses the default shown.

- [ ] **Repo locations (needed by M0).** *Default: `~/wb/workbench` and `~/wb/kb`.*
- [ ] **Zotero details (needed by M1).** Which collections to sync, whether Better BibTeX is installed, and whether to turn on write-back to a `wb-inbox` collection. *Default: sync all collections, read only.*
- [ ] **Your data (needed by M5c).** Format depends on the dataset. *Default: CSV and NWB adapters first; others added when a dataset needs them.*
- [ ] **GCP details (needed by M6b).** Project ID, region, preferred GPU (L4 or T4) and a monthly spend cap for GPU runs. *Default: L4, us-east1, $20/month cap. Nothing runs until you confirm each job.*
- [ ] **Sweep profile (needed by M7).** Refine the starter profile in the "Opus agent workflows" section, or keep it as it is.
- [ ] **Backup remote.** Confirm a private GitHub repo for `kb`, or name another remote.

**Answered**

- **v1 focus:** learning, in two strands: representation/self-supervised learning, and credit assignment/plasticity. Memory moves to phase 2 (appendix).
- **Claude plan:** Pro. Background Opus budget is set to about 4 calls a week; the build itself runs on Fable usage credits.
- **Build location:** Claude Code on the user's Arch machine, in the `workbench/` repo.
- **Hardware:** no local GPU. CPU-only locally, with an optional GCP GPU runner.
- **OS and Docker:** Arch Linux with Docker, so GROBID is the primary PDF parser and the sweep uses a systemd user timer.
- **Emacs:** 30.2. Not yet using org-roam, citar, pdf-tools or elfeed; the spec makes them optional and recommends pdf-tools.
- **Library:** Zotero, integrated through its local API.
- **Project name:** `wb`.

## Appendix: phase 2, memory

Memory was the focus area in earlier drafts. Its seed content is kept here so phase 2 can start without new design work. Fable builds none of it in v1, but the data model, sandbox and sweep must be able to take it on without structural changes.

**Memory bridge map (seed).** The same rules apply as for the v1 bridge map: these are seed entries, verified when ingested.

| Phenomenon | Computational principle | AI analogue | Seed relation | Seed references |
| --- | --- | --- | --- | --- |
| Recency, primacy and temporal contiguity in free recall | Slowly drifting temporal context used as a retrieval cue | Induction heads and in-context retrieval in transformers | empirically-compared | Howard & Kahana 2002; Polyn, Norman & Kahana 2009; Ji-An et al. 2024 (in-context learning and human episodic memory) |
| Pattern completion from partial cues | Content-addressable attractor retrieval | Modern Hopfield networks; transformer attention | formal-equivalence | Hopfield 1982; Krotov & Hopfield 2016; Ramsauer et al. 2021 |
| Separate codes for storing and finding memories | Key–value memory | Attention keys and values; memory-augmented networks | proposed | Gershman, Fiete & Irie 2025 |
| Fast episodic and slow semantic learning; interference | Complementary learning systems | Replay buffers; retrieval-augmented models; catastrophic forgetting | analogy | McClelland, McNaughton & O'Reilly 1995; Kumaran, Hassabis & McClelland 2016; McCloskey & Cohen 1989 |
| Prioritized hippocampal replay | Replay ordered by expected utility | Prioritized experience replay | empirically-compared | Mattar & Daw 2018; Schaul et al. 2016 |
| Relational memory; place and grid cells | Factorized structure and sensory codes | Tolman–Eichenbaum Machine; transformers with recurrent position codes | formal-equivalence | Whittington et al. 2020; Whittington, Warren & Behrens 2022 |
| Predictive maps in hippocampus | Successor representation | Successor features in RL; latent prediction in JEPA | analogy | Dayan 1993; Stachenfeld, Botvinick & Gershman 2017; Assran et al. 2025 (V-JEPA 2) |
| Event boundaries organize memory | Segmentation driven by prediction error | JEPA latent prediction error as a boundary signal | proposed | Zacks et al. 2007; Baldassano et al. 2017; Mur-Labadia et al. 2026 (V-JEPA 2.1) |
| Pattern separation | Sparse, expanded coding | Sparse distributed memory; attention as an approximation of it | formal-equivalence | Kanerva 1988; Yassa & Stark 2011; Bricken & Pehlevan 2021 |

**Memory hypothesis cards (seed),** numbered after the learning cards:

1. **H-0004.** A small transformer trained on list-recall tasks shows a forward-asymmetric lag-CRP only when its position or context code drifts across the list. *Test:* compare lag-CRP curves with PEERS free-recall data. *Falsifier:* the same asymmetry appears with static position codes.
2. **H-0005.** The inverse temperature (β) of modern Hopfield retrieval controls list-length and fan effects the way CMR's retrieval parameters do. *Test:* fit β to published effect sizes. *Falsifier:* no β fits both effects at once.
3. **H-0006.** Peaks in JEPA latent prediction error line up with human-annotated event boundaries in naturalistic video. *Test:* a distilled V-JEPA 2.1 encoder (ViT-B scale) on a public video set with boundary annotations, to be identified in phase 2. *Falsifier:* alignment is no better than a low-level visual-change baseline.

**Memory sandbox additions.**

- A CMR reference model that reproduces published serial-position and lag-CRP curves.
- Components: temporal context, Hebbian association, modern Hopfield retrieval, key–value memory and sparse expansion.
- Task simulators: free recall, serial recall, paired associates (AB–AC), recognition, continuous recognition and lure discrimination.
- Benchmarks: serial-position curve, probability of first recall, lag-CRP, semantic CRP, list-length and presentation-rate effects, interference, forgetting and spacing, recognition ROC and the mirror effect.

**Memory datasets.** [PEERS](https://memory.psych.upenn.edu/PEERS) behavioral free recall (direct download; EEG on OpenNeuro). Penn RAM intracranial free-recall data ([request required](https://memory.psych.upenn.edu/Data)).

**Memory sweep keywords:** episodic memory, free recall, serial position, temporal contiguity, recognition memory, pattern separation and completion, consolidation, hippocampal replay, event segmentation, temporal context model, retrieved context, CMR, successor representation, complementary learning systems, Hopfield network, key–value memory, induction heads.

**Memory reading list.** All references in the memory table, plus 10–15 reviews chosen from the citation graph and approved by the user.

## Sources

- [Claude plans and pricing](https://claude.com/pricing): Opus on Pro; five-hour and weekly limits shared across chat and Claude Code.
- [Claude Fable models on your plan](https://support.claude.com/en/articles/15424964-claude-fable-models-on-your-plan): Fable runs on usage credits on Pro.
- [Context windows on paid plans](https://support.claude.com/en/articles/8606394-how-large-is-the-context-window-on-paid-claude-plans): Opus 5.5 has a 1M-token context window in Claude Code.
- [Claude Help Center: Agent SDK and claude -p usage](https://support.claude.com/en/articles/15036540): the paused June 2026 change; these calls still use plan limits.
- [Claude Code docs: headless / non-interactive mode](https://code.claude.com/docs/en/headless.md)
- [Claude Help Center: Using Claude Code with your Pro or Max plan](https://support.claude.com/en/articles/11145838-): the `ANTHROPIC_API_KEY` precedence warning.
- [Claude Code docs: legal and compliance](https://code.claude.com/docs/en/legal-and-compliance): ordinary individual use; OAuth credentials.
- [OpenAlex authentication and pricing](https://developers.openalex.org/api-reference/authentication)
- [GROBID vs Docling comparison (Docling discussion #622)](https://github.com/docling-project/docling/discussions/622)
- [claude-code.el](https://github.com/stevemolitor/claude-code.el) and [claude-code-ide.el](https://github.com/manzaltu/claude-code-ide.el)
- [PEERS dataset](https://memory.psych.upenn.edu/PEERS) and [Computational Memory Lab data](https://memory.psych.upenn.edu/Data)
- [V-JEPA 2.1 (arXiv 2603.14482)](https://arxiv.org/html/2603.14482): released checkpoints, including distilled ViT-B/L.
- [Zotero local API](https://www.zotero.org/support/dev/web_api/v3/local_api): read without authentication; writes in Zotero 10+ with a local API key.
- [GitHub file and repository size limits (summary)](https://fast.io/resources/github-file-size-limit-guide.md)
- [THINGS-data](https://vicco-group.github.io/THINGS-data): behavior on OSF, fMRI and MEG on OpenNeuro and Figshare.
- [IBL behavior data release](https://int-brain-lab.github.io/iblenv/notebooks_external/data_release_behavior.html): mouse learning data through the ONE API.
- [Brain-Score vision](https://github.com/brain-score/vision): local runs cover the public benchmarks only.
- [Neuromatch dataset guide](https://instructor.compneuro.neuromatch.io/projects/neurons/README.html): Zhong et al. 2025 mouse visual learning recordings.
