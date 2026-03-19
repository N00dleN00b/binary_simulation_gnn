# Binary Similarity Detection with Graph Neural Networks

> Control-flow graph embedding via GNN to find functionally similar code across obfuscated binary variants — applied to patch diffing and vulnerability propagation.

---

## Goal

When a vulnerability is patched in one library, it often exists unpatched in dozens of forks, vendor copies, and embedded systems. This project builds a GNN-based binary similarity engine that can identify functionally equivalent code even across compiler optimizations, obfuscation, and architecture differences. The application is patch diffing and vulnerability propagation — finding where a known-vulnerable function appears in the wild.

---

## Research Questions

- Can GNN embeddings capture functional similarity better than traditional approaches (BinDiff, instruction counting)?
- How does performance degrade across: different compilers (gcc vs clang), optimization levels (O0–O3), and obfuscation methods (string encryption, control-flow flattening)?
- What graph features — basic block attributes, call graph structure, data flow edges — contribute most to embedding quality?
- At what granularity is comparison most useful: function-level, basic-block-level, or binary-level?
- Can a model trained on one architecture (x86) generalize to another (ARM)?

---

## Project Phases

### Phase 1 — Binary dataset construction
- Compile a set of open-source C/C++ projects (OpenSSL, zlib, curl) at multiple optimization levels and with multiple compilers
- For obfuscation testing: use OLLVM or similar to generate obfuscated variants
- For vulnerability testing: identify CVEs with known-patched functions, compile both vulnerable and patched versions

### Phase 2 — Disassembly and CFG extraction
- Use Ghidra (headless) or Binary Ninja to disassemble binaries and extract per-function control-flow graphs
- Node features per basic block: instruction count, opcode n-gram histogram, call targets, memory access patterns
- Export graphs to a format suitable for PyTorch Geometric (edge list + node feature matrix)

### Phase 3 — GNN model design
- Architecture: Graph Isomorphism Network (GIN) or Graph Attention Network (GAT) — both are strong baselines for this task
- Training objective: metric learning with triplet loss (anchor function, positive = same function different compilation, negative = different function)
- Embedding space: 128 or 256 dimensions, L2-normalized
- Evaluate: mean average precision (mAP) on a held-out function retrieval task

### Phase 4 — Embedding evaluation
- Retrieval task: given a query function embedding, rank a pool of 10k candidates, measure mAP@10
- Ablation: GNN vs simpler baselines (mean opcode histogram, random walk embeddings)
- Cross-architecture generalization: train on x86, evaluate on ARM (requires multi-arch disassembly)

### Phase 5 — Patch diffing application
- Given two binary versions of the same library, embed all functions from both
- Match functions by nearest neighbor in embedding space
- Highlight functions with high embedding distance as "changed" — i.e., potential patch locations
- Visualize as a diff report: matched pairs, unmatched (added/removed), changed

### Phase 6 — Vulnerability propagation demo
- Take a known CVE function (e.g. a heap overflow in OpenSSL)
- Embed it and search a corpus of other binaries for nearest neighbors
- Report which binaries/functions are most similar to the vulnerable function

---

## Repo Structure

```
binary-similarity-gnn/
├── binaries/
│   ├── raw/                  # Compiled binaries (gitignored)
│   └── build_scripts/        # Dockerized build scripts per project + opt level
├── ghidra_scripts/
│   └── export_cfg.py         # Headless Ghidra script to extract CFGs
├── data/
│   ├── graphs/               # Serialized CFG graph objects (.pt files)
│   ├── splits/               # Train/val/test function pair lists
│   └── embeddings/           # Precomputed embeddings for retrieval benchmarks
├── notebooks/
│   ├── 01_cfg_extraction_eda.ipynb
│   ├── 02_feature_engineering.ipynb
│   ├── 03_model_training.ipynb
│   ├── 04_retrieval_eval.ipynb
│   └── 05_patch_diff_demo.ipynb
├── src/
│   ├── disassembly/
│   │   ├── ghidra_runner.py  # Subprocess wrapper for headless Ghidra
│   │   └── cfg_parser.py     # Parse Ghidra JSON export → PyG Data objects
│   ├── features/
│   │   └── node_features.py  # Basic block feature extraction (opcodes, calls)
│   ├── model/
│   │   ├── gnn.py            # GIN / GAT implementation in PyTorch Geometric
│   │   ├── loss.py           # Triplet loss, contrastive loss
│   │   └── train.py
│   ├── eval/
│   │   ├── retrieval.py      # mAP@k computation
│   │   └── patch_diff.py     # Binary diff using embedding nearest neighbor
│   └── demo/
│       └── vuln_search.py    # Query a corpus with a known-vulnerable function
├── tests/
├── docker/
│   └── Dockerfile.build      # Reproducible compilation environment
├── requirements.txt
└── README.md
```

---

## Stack

| Purpose | Library |
|---|---|
| Disassembly | Ghidra (headless), optionally Binary Ninja |
| Graph ML | `torch_geometric` (PyG) |
| Deep learning | `pytorch` |
| Graph utilities | `networkx` |
| Embeddings / ANN search | `faiss-cpu` |
| Visualization | `matplotlib`, `plotly` (UMAP of embedding space) |
| Build automation | `docker`, `make` |

---

## Datasets and Source Binaries

- **OpenSSL** — well-studied, many CVEs, compiled at O0/O1/O2/O3 with gcc and clang
- **zlib, curl, libpng** — common embedded dependencies, useful for propagation demo
- **CRONOS dataset** (if accessible) — purpose-built binary similarity benchmark
- **BinaryCorp** — open binary similarity dataset from academic papers

To compile your own dataset:

```bash
# Example: build OpenSSL at all opt levels
docker run --rm -v $(pwd)/binaries:/out binary-builder \
  bash build_scripts/build_openssl.sh
```

---

## Key Metrics to Track

| Metric | Baseline target |
|---|---|
| mAP@1 (same function retrieval) | > 0.85 (same compiler, different opt level) |
| mAP@1 (cross-compiler) | > 0.70 |
| mAP@1 (obfuscated) | > 0.55 |
| mAP@1 (cross-architecture) | > 0.50 |
| Embedding throughput | > 1000 functions/sec on CPU |

---

## Baseline Comparisons

Implement at least one non-GNN baseline to quantify the GNN's contribution:

- **BinDiff heuristic** — matched/unmatched by name, hash, or structural hash
- **Mean opcode histogram** — average of one-hot opcode vectors over all basic blocks
- **Random walk embedding** — node2vec on the CFG (no function semantics)

---

## Stretch Goals

- [ ] Cross-architecture: train on x86 + ARM jointly using architecture-agnostic node features
- [ ] Lifted IR approach: compare against a model trained on VEX IR or LLVM IR instead of raw assembly
- [ ] Interactive embedding visualizer: UMAP plot where hovering shows function name and binary origin
- [ ] Integration with Ghidra plugin: run similarity search directly from the Ghidra UI
- [ ] Evaluate on real-world CVE propagation: find a CVE, scan a set of IoT firmware images

---

## References

- Xu et al., "Neural Network-based Graph Embedding for Cross-Platform Binary Code Similarity Detection" (CCS 2017) — the paper that started this direction
- Ding et al., "Asm2Vec: Boosting Static Representation Robustness for Binary Clone Search" (S&P 2019)
- PyTorch Geometric docs: https://pytorch-geometric.readthedocs.io
- Ghidra headless mode: https://ghidra-sre.org/CheatSheet.html

---

## Environment Setup

```bash
# Python dependencies
pip install -r requirements.txt

# Ghidra (headless) — download separately
export GHIDRA_HOME=/path/to/ghidra_10.x

# Run CFG extraction on a binary
python src/disassembly/ghidra_runner.py \
  --binary binaries/raw/openssl-1.1.1-O2 \
  --output data/graphs/
```

---

## Status

- [ ] Phase 1 — Dataset construction and compilation
- [ ] Phase 2 — CFG extraction pipeline
- [ ] Phase 3 — GNN model training
- [ ] Phase 4 — Retrieval evaluation
- [ ] Phase 5 — Patch diffing application
- [ ] Phase 6 — Vulnerability propagation demo
