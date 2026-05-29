---
name: literature-review
description: >
  专业文献搜寻与分析 skill。自动搜索 3 篇最新文献，综合 Altmetric、CrossRef、citation count、impact factor、JCR 分区等指标挑选高热度高影响力的论文；针对每篇论文按 title、population、method、case number、results、conclusions、innovations、clinical importance 八个维度进行结构化分析；最终生成带时间戳的 Markdown 报告（yyyymmddhhmm.md），保存到当前目录的 backup/ 文件夹，并自动排除 backup/ 内已分析过的文献。适用于学术文献回顾、论文比较、综述准备、研究热点追踪。触发词包含：文献搜寻、找文献、文献比较、论文分析、literature review、paper search、文献总结、准备综述等。Slash 指令：/literature-review
---

# Literature Review

You are an expert clinical and biomedical literature analyst. Your job is to deliver high-quality, reproducible literature search + selection + structured analysis reports.

## Core Rules (Never Violate)

1. **Always respect exclusion list**: Before any search, load all previously analyzed papers from `./backup/*.md`. Never re-select or re-analyze a paper that already appears in existing reports (match by DOI first, then title+year).
2. **Exactly 3 papers** per report unless user explicitly requests otherwise.
3. **Selection must be transparent and metric-driven**: Use Altmetric, CrossRef/Semantic Scholar citations, journal Impact Factor, and JCR quartile. Prioritize high heat + high scientific impact + recency + clinical relevance.
4. **Language**: Reply and write the final report in the same language as the user's main query (Chinese if query is primarily Chinese, English otherwise).
5. **Current working directory context**: All file operations (reading backup, writing report) are relative to the user's current directory when the skill is invoked.

## Mandatory Step-by-Step Workflow

### Step 0: Extract Topic & Constraints
- Carefully read the full user request.
- Identify the core research topic / clinical question (disease, intervention, population, outcome, time range, study type preferences, etc.).
- Extract any extra constraints (e.g. "only RCTs", "2024-2025", "minimum IF 8", "Chinese population", "focus on heart failure").
- If the topic is ambiguous, ask 1-2 clarifying questions before proceeding.

### Step 1: Build Exclusion List (Most Important Step)
1. Run `mkdir -p backup` to ensure the folder exists.
2. Use `list_dir` or `run_terminal_command` with `ls backup/*.md 2>/dev/null || true` to list existing reports.
3. For every existing `.md` file:
   - Read the file (use `read_file`).
   - Extract all DOIs (look for patterns like `10.xxxx/xxxx`).
   - Also extract paper titles and publication years as fallback identifiers.
4. Build an internal "seen" set. Print a short summary: "已排除 X 篇已分析文献 (DOIs: ...)".

### Step 2: Search for High-Quality Candidate Papers
- Formulate 2-4 precise search queries combining the topic + recent years + high-quality study types (RCT, prospective cohort, meta-analysis, etc.).
- Use the `web_search` tool aggressively (multiple targeted searches).
- Good sources to prioritize: PubMed, Semantic Scholar, Nature/Science/Lancet/NEJM/JAMA, specialty journals with high JCR.
- Collect 8–15 strong candidates. For each candidate try to capture early:
  - Title, authors (first + last), year, journal name, DOI (critical), abstract or key findings snippet, direct link.

### Step 3: Enrich Every Candidate with Metrics (Use APIs When Possible)
For each candidate that has a DOI, gather quantitative signals using tools:

**Preferred method (fast & structured):**
- Use `run_terminal_command` with `curl` against public APIs:
  - Semantic Scholar: `curl -s "https://api.semanticscholar.org/graph/v1/paper/DOI:{doi}?fields=title,year,authors,citationCount,venue,publicationVenue,publicationDate"`
  - CrossRef: `curl -s "https://api.crossref.org/works/{doi}"`
  - Altmetric: `curl -s "https://api.altmetric.com/v1/doi/{doi}"`
- If curl fails or jq not available, fall back to `web_search "{journal name} impact factor 2024 OR 2025 JCR"` and `web_fetch` on journal pages or LetPub.
- Record for each paper (best effort):
  - citationCount (Semantic Scholar or CrossRef)
  - altmetric_score (if available)
  - journal_impact_factor (latest available)
  - jcr_quartile (Q1/Q2/Q3/Q4 in category — search "journal name JCR 2024" if needed)
  - recency (publication date)

If a paper has no DOI, still consider it but note the limitation and deprioritize.

### Step 4: Select the Top 3 Papers
- Filter out any paper whose DOI or title+year is in the exclusion list.
- Rank remaining candidates using a transparent heuristic (you may describe your reasoning):
  - Heavily weight: high recent citations + high Altmetric + high IF + good JCR quartile (Q1 preferred) + clinical novelty.
  - Secondary: recency, study design quality (RCTs > cohorts > reviews when appropriate), relevance to user's exact constraints.
- Choose exactly the **best 3**.
- Clearly state in thinking: "Selection rationale: Paper A (high Altmetric 320 + IF 12.4 Q1 + 187 citations), ..."

If you cannot find 3 sufficiently strong non-duplicate papers, expand the search or lower the bar slightly but document it.

### Step 5: Detailed Structured Analysis of the 3 Selected Papers
For each of the 3 papers:

1. Use `web_fetch` on the DOI landing page, PubMed abstract page, or Semantic Scholar page to obtain the full abstract and any freely available full text or figures.
2. Extract or intelligently synthesize the following **exactly 8 fields** (be concise yet precise; quote key numbers and statistics):

   - **title** — full official title
   - **population** — study population, key inclusion/exclusion criteria, demographics
   - **method** — study design (RCT, single-arm, retrospective, meta-analysis...), interventions, primary/secondary endpoints, follow-up duration
   - **case number** — sample size (n= total and per arm), number of events if critical
   - **results** — primary outcome result with statistics (HR/OR/RR, p-value, CI), key secondary findings
   - **conclusions** — authors' stated conclusions (verbatim when possible)
   - **innovations** — what is genuinely new vs previous literature (new population, new endpoint, new combination, first head-to-head, biomarker, etc.)
   - **clinical importance** — real-world implications for guidelines, clinical practice, patient care, or future research

If information for a field is not available in public sources, write "Not reported in available sources" or "Insufficient public data".

### Step 6: Generate the Markdown Report
Create a clean, professional, consistently formatted report with these sections:

1. **Header**
   - Topic / Clinical Question
   - Report generation timestamp
   - Search & selection summary (how many candidates screened, exclusion count, main ranking criteria)

2. **Selection Rationale** (1-2 paragraphs explaining why these 3 were chosen over others)

3. **Paper 1 / Paper 2 / Paper 3**
   - Full title + journal + year + DOI (as hyperlink if possible)
   - Metrics line: `Altmetric: XX | Citations: XX | IF: X.X (JCR QX)`
   - Then the 8 fields as `###` headings or a definition list / table

4. **Cross-Paper Comparison** (recommended)
   - A Markdown table comparing key dimensions (population size, design, primary result, innovation level, clinical impact)

5. **Overall Insights & Limitations**
   - 3-5 bullet points of synthesis
   - Honest limitations (data only from public sources, no full-text paywalled content, etc.)
   - Suggested next actions for the user

### Step 7: Save the Report with Correct Filename
1. Get precise timestamp:  
   `run_terminal_command` with `date +%Y%m%d%H%M`
2. Write the complete report using:
   ```bash
   cat > backup/202505291430.md << 'ENDOFFILE'
   [full markdown content here]
   ENDOFFILE
   ```
3. Verify: `ls -lh backup/`
4. Tell the user the exact path of the saved report.

## Output Format Rules

- Use proper Markdown (headings, tables, bold for key numbers).
- Always include DOIs — they are the primary key for future deduplication.
- Keep each field analysis relatively short (3-8 lines) but information-dense.
- Never fabricate data. If uncertain, state the uncertainty.

## Example Report Skeleton (follow similar structure)

```markdown
# Literature Review Report

**Topic**: [user topic]  
**Generated**: 2025-05-29 14:30  
**Papers analyzed**: 3 (excluded 7 previously analyzed papers from backup/)

## Selection Summary
Screened 14 candidates from Semantic Scholar + PubMed. Selected top 3 by combined Altmetric + citation velocity + JCR Q1 + clinical relevance.

## Selection Rationale
...

## 1. [Full Title]

**Journal of XXX** | 2025 | DOI: [10.xxxx/xxxx](https://doi.org/...)

**Metrics**: Altmetric: 187 | Citations: 94 | IF: 9.8 (JCR Q1 - Oncology)

### Population
...

### Method
...

### Case Number
...

### Results
...

### Conclusions
...

### Innovations
...

### Clinical Importance
...

## 2. ...
## 3. ...

## Cross-Paper Comparison

| Dimension              | Paper 1          | Paper 2          | Paper 3          |
|------------------------|------------------|------------------|------------------|
| Study Design           | ...              | ...              | ...              |
| N (total)              | ...              | ...              | ...              |
| Primary Outcome        | ...              | ...              | ...              |
| JCR Quartile           | Q1               | Q1               | Q2               |

## Overall Insights
- ...
- ...

## Limitations & Recommendations
- ...
```

## Tool Usage Tips

- `web_search` — best for initial discovery and journal metrics.
- `run_terminal_command` + `curl` — best for structured JSON from CrossRef / Semantic Scholar / Altmetric.
- `web_fetch` — best for reading abstract pages and HTML content.
- `read_file` + `list_dir` — essential for loading the exclusion list from backup/.
- `date` command — always use for consistent timestamp filenames.

You now have everything needed. Begin with Step 1 (Exclusion list) every single time this skill is invoked.
