---
name: research-gap-finder
description: Discovers confirmed-open research gaps from academic papers in any domain. Use when user says "find research gaps", "scan research gaps", "what's unsolved in", "open problems in", "research ideas for", or asks about academic frontiers and unsolved problems in a field. Requires the discovery-dashboard package.
compatibility: Requires discovery-dashboard (github.com/niyaznurbhasha/discovery-dashboard) with conda env "discovery"
metadata:
  author: niyaznurbhasha
  version: 1.0.0
  category: research
  tags: [research, academic, arxiv, semantic-scholar, gap-analysis]
---

# Research Gap Finder

Searches arxiv + Semantic Scholar for survey papers, open-problems papers, and research agendas in any domain. Extracts "Future Work" / "Limitations" sections from papers via ar5iv HTML. Validates that gaps are confirmed unsolved.

**Requires:** [discovery-dashboard](https://github.com/niyaznurbhasha/discovery-dashboard) installed with its conda environment.

## Instructions

### Step 1: Run the scan

```bash
conda run -n discovery python research_gaps.py --domain "<domain>" --max-papers 40
```

**Parameters:**
- `--domain` (required): 1-3 words works best. Long queries return fewer results.
- `--max-papers`: Max papers to process (default 30, recommended 40).
- `--max-extract`: Max papers for ar5iv full-text extraction (default 15).
- `--skip-ar5iv`: Skip full-text extraction, abstracts only (faster).

**Tips:**
- Short domains work best: "AI safety", "world models", "continual learning"
- Multiple runs on related terms build richer coverage
- If 0 results, try broader/shorter terms

### Step 2: Check results

The script outputs a summary with paper counts and validation breakdown.

Results saved to:
- `data/research_gaps/{domain-slug}_papers.json` — raw paper data
- `data/research_gaps/{domain-slug}.json` — dashboard display file

### Step 3: Re-render dashboard

```bash
conda run -n discovery python dashboard.py
```

Results appear in **Edges tab → Research Ideas**.

### Step 4: Report findings

Summarize: paper count, validation breakdown, top confirmed-open gaps, connections to user's work.

## How It Works

1. **DISCOVER**: 6 search strategies across arxiv + S2 (surveys, open problems, position papers, workshop papers, limitations). Filters by authority.
2. **EXTRACT**: Fetches ar5iv HTML, extracts Future Work / Limitations sections. Abstract fallback.
3. **VALIDATE**: Recency (last 2 years = confirmed), cross-referencing (2+ papers = confirmed), addressing-paper search.

## Troubleshooting

- **0 papers**: Domain query too long — try shorter terms
- **Mostly uncertain**: Papers older than 2 years with many addressing papers — normal for mature fields
- **Slow (5-10 min)**: S2 rate limiting — use `--skip-ar5iv` or `--max-papers 20`
