### General

* Use @./docs/ai/OVERVIEW.md to understand the high-level project organization.
* Use @./docs/ai/CONCURRENCY_MODEL.md if understanding concurrency model of request and execution handling is important for the task.
* Ask clarifying questions from the user if details important for the task are missing.
* When working on a bug fix, follow the RED-GREEN-BLUE TDD approach.
* After building a mental model of a complicated flow, update or create a document in @./docs/ai/ to capture that knowledge so future sessions don't have to re-derive it.
* Never ignore error results `_ = smth.Close()`, errors need to bubble up OR be logged.
* After correcting a mistake based on user feedback, if it could have been avoided with better instructions, update the "AI learnings" section of @AGENTS.md with a rule that prevents the same class of mistake and notify the user about the update.

### Code Style

* When creating core types from a2a package use constructor functions from @./a2a/core.go (eg. `a2a.NewMessage(...)`, `a2a.NewStatusUpdateEvent(...)`).
* Do not leave comments in the code unless they explain a non-trivial implementation detail or highlight a suboptimally handled edge-case.
* Prefer early `return`-s and `continue` over deeply nested blocks.

### Test Writing

* Do not leave comments in tests unless they explain a non-trivial implementation detail. Test name and test case setup should be self-explanatory.
* ALWAYS use existing test files as a reference when generating new tests. Prioritize files in the same package if they exist.
* Write "table-driven tests" when logic can be shared across multiple test cases.
* Prioritize testing observable behavior of exported methods, not the internal state.
* Use `google/go-cmp/cmp` and `cmp.Diff(want, got)` for comparing structs, slices, maps, or complex types. Do NOT use `reflect.DeepEqual` or manual field checks.
* Use existing mocking utilities from @./internal/testutil or create a new utility in the package if needed.
* Use `t.Parallel()` at the start of test cases.
* Use `receiver.Operation() error = %v, want %v` as a template for printing test error check failures.
* Use `receiver.Operation() = %v, want %v` as a template for printing test error check failures.
* Use `receiver.Operation() wrong result (-want +got) diff = %s` as a template for printing test errors received when using `cmp.Diff`.
* Prefer using `t.Fatalf` over `t.Errorf` unless printing all the failed checks is justified or the method is called not from the main goroutine. 

### AI learnings

* ...


## 🛡️ Strict Embedding Separation, Zero-Fallback Law & 24/7 Dual-GPU Invariant
- **Reference**: 

### 1. Das Absolute Fallback-Verbot (Zero-Fallback Law)
Unter keinen Umständen, zu keinem Zeitpunkt und aus keinem Grund darf ein Fallback zwischen verschiedenen Embedding-Modellen stattfinden.
* **Geltende Aktion:** Fällt ein Embedding-Modell aus oder ist überlastet, MUSS die Operation sofort hart fehlschlagen () oder die Payload transaktional in einer Queue (NATS/SQLite) verharren, bis das exakte Modell bereit ist.
* **Verboten:** Kein stiller oder dynamischer Modellwechsel (weder Jina -> Gemma noch umgekehrt).

### 2. Warum ein Embedding-Fallback mathematisch & informationstheoretisch unmöglich ist
* **Topologische Inkompatibilität heterogener Vektorräume (Non-Isomorphism):**
  Jedes Modell 	heta: \mathcal{X} 	o \mathbb{R}^D$ projiziert Text in eine spezifische, gelernte Riemannsche Mannigfaltigkeit. Jina v5 (=256$) und EmbeddingGemma (=768$) spannen zwei völlig inkompatible geometrische Räume auf. Die Basisvektoren der semantischen Achsen sind ohne explizite Procrustes-Transformation nicht ausgerichtet.
* **Kollaps der Kosinus-Ähnlichkeit ($	ext{sim} pprox 0$):**
  Wird eine Suchanfrage mit Modell $ berechnet ( = f_B(q)$), während der Dokumentenkorpus mit Modell $ indiziert wurde ( = f_A(d)$), verhält sich das Skalarprodukt mathematisch wie das zweier rein zufälliger Vektoren auf einer hochdimensionalen Einheitssphäre:
  86306\mathbb{E}[	ext{sim}(u, v)] = 0 \quad 	ext{mit Varianz} \quad \sigma^2 = rac{1}{D}86306
  Der Nearest-Neighbor-Algorithmus (HNSW/k-NN) liefert stochastisches Rauschen. Das RAG-System erhält völlig falsche oder irrelevante Kontexte.
* **Irreversible Index-Vergiftung (Index Poisoning):**
  Wird auch nur ein einziger Vektor von Modell $ als 'Fallback' in den Index von Modell $ geschrieben, verunreinigt er die Distanzgraphen und Clusterzentren dauerhaft.
* **Das Gesetz des Fail-Fast:**
  Ein Ausfall muss hart abbrechen ().

### 3. Duale 24/7 Erfassungspflicht (GPU-Only)
* **GPU-Only Mandat:** Es läuft absolut nichts auf der CPU — GPU ONLY (NVIDIA GB10 CUDA) für ausnahmslos jedes Embedding-Modell.
* **24/7 Parallelität:** Sowohl  (256D, ~4,1 GB VRAM) als auch  (768D, ~1,2 GB VRAM) laufen dauerhaft 24/7 im VRAM (Summe ~5,3 GB VRAM).
* **Duale Erfassung:** Jeder zu indizierende Text/Chunk wird immer von beiden Modellen parallel eingebettet und getrennt persistiert.

### 4. Idioten- & Failsafe-Sicherung auf Datenbankebene
* **SQLite Schema CHECK-Constraints:**
   und  erzwingen atomare Abbrüche auf Engine-Ebene bei Modell-Mismatches.
* **Qdrant Collection Constraints:**
  Strikte Trennung in separate Collections ( vs ) mit fixierter Vektordimension.


## ⚡ High-Quality Systems Programming Languages Priority (No-Python Policy)
- **Reference**: `/home/m1st/.agents/rules/RULE_High_Quality_Systems_Programming_Languages.md`
- **Rule**:
  1. **Bevorzugte Sprachen:** High Quality **Golang (Go), Rust, C++, Zig, PowerShell, C** sind IMMER und AUSNAHMSLOS die bevorzugten Programmiersprachen.
  2. **Kein Python:** Python ist für neue Daemons, Watcher, Automatisierungen, APIs, CLI-Tools und Dienste strikt untersagt (GIL-Bottlenecks, Dependency-Drift, Speicherineffizienz).
  3. **Natives Systems-Engineering:** Alle Hintergrunddienste, Caching-Ebenen und Task-Runner müssen als native, speichersichere und nebenläufige Binaries kompiliert werden.
