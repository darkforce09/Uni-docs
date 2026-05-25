# NotebookLM Course Material Standardization Framework (Automation-Ready)

This framework is designed for both manual and automated (e.g., n8n) workflows to standardize university course materials for optimal NotebookLM performance.

---

## 1. Data Structure & Metadata Schema

For automation tools like **n8n** to parse these files correctly, we must use a strict, predictable YAML schema.

### Strict YAML Frontmatter
The YAML block must be the first thing in the file. Use the following keys precisely:
```yaml
---
course_code: "MV037G"
document_type: "Transcription | Slide Deck | Question Bank | Index"
subject_topic: "Specific Subject"
lecturer_name: "Name of Professor"
original_lecture_date: "YYYY-MM-DD"
source_format: "PDF | MP3 | TXT"
language_iso: "sv | en"
last_processed: "YYYY-MM-DD"
version_control: "v1.0.0"
---
```

### Predictable Header Hierarchy
Automated systems use headers to split data. Always use these exact H2 titles:
- `## Sammanfattning` (Brief summary for the AI base-prompt)
- `## Innehåll` (The primary body of the material)
- `## Nyckelord` (Comma-separated tagging)
- `## Referenser` (Optional: Links to other files)

---

## 2. Standardized Formatting Logic

### The "Prose-First" Rule
- **Logic**: Convert slide-based bullet points into full-sentence paragraphs. 
- **Why**: NLP models (as used in NotebookLM) derive better context from grammatical sentences than from isolated fragments.
- **Automation Tip**: When using an LLM in an n8n node, use the prompt: *"Rewrite the following bullet list into clinical, objective prose while maintaining all technical terms."*

### Semantic Weighting (Bolding)
- **Rule**: **Bold** the first occurrence of technical/medical terms per section.
- **Logic**: Signals importance to the vector embedding index.

### Tabular Data Rule
- **Rule**: For structured data, comparisons, or lab value sets, ALWAYS use Markdown pipe tables (`|`). 
- **Reason**: Tables provide explicit semantic boundaries that help LLMs associate headers with values more reliably than bulleted lists.
- **Automation Tip**: In an n8n LLM node, use the prompt: *"Convert any lists of clinical parameters or comparative data into a standard Markdown pipe table format."*

### Diagram & Visual Data
- **Rule**: Wrap visual descriptions in a dedicated block.
- **Syntax**: 
  ```markdown
  > [!NOTE] Diagrambeskrivning: [Name]
  > [Detailed prose description of the visual flow/relationship]
  ```

---

## 3. Automation-Ready Logic (n8n Implementation)

If you are building an automated pipeline in **n8n**, follow these logic gates:

### Step 1: Ingestion & Splitting
- **Node**: Read Binary File → Move to Text.
- **Action**: Use the `---` delimiters to extract YAML data for the n8n JSON object.

### Step 2: Content Transformation (LLM Node)
- **Prompt Directive**: *"Strictly follow the 'Prose-First' rule. Use the 'course_code' and 'subject_topic' from the YAML to provide context. Extract 15 unique 'Nyckelord' and place them at the end under a ## Nyckelord header."*

### Step 3: Automated Quality Control (The "QC Gate")
To prevent hallucinations or data loss during the LLM transformation, implement a secondary verification node.
- **Node**: LLM Chain (using a different model for "The Referee").
- **Prompt**: *"Compare the original source data with the rewritten prose. List any technical terms or specific medical values that were present in the source but are missing from the prose. Also, identify any 'new' medical facts added that were NOT in the source."*
- **Action**: 
  - **IF** discrepancies found: Route to `status: manual_review`.
  - **ELSE**: Proceed to final write.

### Step 4: Filename Normalization
- **Rule**: Standardize filenames before writing.
- **Regex Pattern**: `[CourseCode]_[Type]_[Topic].md`
- **Action**: Remove all spaces, special characters, and "Del1/Del2" tags.

---

## 4. Deep Citations & Numerical Data (Academic Precision)

For medical and scientific subjects, precision is mandatory. To achieve this in NotebookLM, we must standardize how we cite sources and record data.

### Deep Citation Markers
- **Rule**: When converting a PDF/Slide deck, keep track of the original slide/page numbers. 
- **Syntax**: At the end of every significant H3 section, add a bracketed source marker.
- **Example**: `[Källa: Bild 14, Föreläsning_Hjärtsvikt]`
- **Reason**: This allows NotebookLM to give you the exact slide number when you ask for its source, making it much easier to verify facts in the original material.

### Numerical & Lab Value Standard
- **Rule**: Lab values and ranges must always include their units and an explicit "Normal" label. When presenting multiple values together (e.g., a blood gas panel), use a pipe table (see Section 2). For inline single values, use the syntax below.
- **Inline Syntax**: `[Parameter]: [Value] [Unit] (Normal: [Range])`
- **Example**: `S-Kalium: 5.2 mmol/L (Normal: 3.5-5.0)`
- **Reason**: Standardized formatting prevents the AI from confusing historical patient data with general medical facts.

---

## 5. Automation Error Handling (n8n Logic)

When using **n8n** for mass processing, your workflow must handle "Edge Cases" effectively.

### The "Dead PDF" Gate
- **Scenario**: The PDF is a scanned image (no text layer).
- **Trigger**: If `Read Binary File -> Text` returns `< 100 characters`.
- **Logic**:
  - **IF** Scan is detected: Route to an **OCR Node** (Optical Character Recognition).
  - **ELSE**: Proceed with standard extraction.
  - **IF** OCR fails: Post a notification to a "Manual Review" queue.

### The Metadata Matcher
- **Scenario**: The filename doesn't match the course code.
- **Logic**: Check the first 500 characters for the string `[CourseCode]`. If not found, tag the file with `status: needs_review_metadata` in the YAML.

---

## 6. RAG Optimization (AI Navigation)

### Contextual Cross-Linking
- **Rule**: At the end of every `Transcription` file, add a "Related Material" section.
- **Syntax**: `Referens: [Exact_Slide_Deck_Filename.md]`
- **Reason**: This creates a "web" of context that helps the RAG agent retrieve paired data (Theory + Slides).

### Sanity Testing (Verification)
After ingestion, ask NotebookLM:
1. *"Identify the 'Index / Study Map' and list the core modules of this course."*
2. *"Using only the 'Question Bank' files, generate a 5-question mock exam on [Topic]."*
3. *"Based on a 'Transcription' file, explain the clinical significance of [Term X]."*

---

## 7. Verification Checklist (The "QA" Gate)
- [ ] **YAML**: Valid and at the very top?
- [ ] **QC Pass**: Has the content been cross-referenced for data loss/hallucinations?
- [ ] **Tabular Data**: Are lab values and comparisons in pipe tables (`|`)?
- [ ] **Citations**: Are section-level source markers included?
- [ ] **Numbers**: Are lab values written in the `Value [Unit] (Normal: Range)` format?
- [ ] **OCR Check**: Was the file checked for a readable text layer?
- [ ] **Naming**: Does the filename follow the `[Course]_[Type]_[Topic].md` pattern?

---

## 8. Template Example
```markdown
---
course_code: "MV037G"
document_type: "Transcription"
subject_topic: "Syra-Bas"
lecturer_name: "Petter Berggrund"
original_lecture_date: "2025-09-09"
source_format: "MP3"
language_iso: "sv"
last_processed: "2026-03-28"
version_control: "v1.0.0"
---

# Föreläsning: Syra-basbalansen

## Sammanfattning
Denna föreläsning går igenom kroppens homeostas rörande pH-värden...

## Innehåll
### Respiratorisk Acidos
Vid respiratorisk acidos sjunker pH-värdet på grund av hypoventilation...
[Källa: Bild 12, Föreläsning_Syra_Bas]

### Tabell: Blodgasvärden
| Parameter | Värde | Enhet | Normalintervall |
| :--- | :--- | :--- | :--- |
| pH | 7.25 | - | 7.35 - 7.45 |
| pCO2 | 7.8 | kPa | 4.8 - 5.8 |
| HCO3 | 23 | mmol/L | 21 - 27 |

## Nyckelord
Acidos, alkalos, koldioxidtryck, bikarbonat...

## Referenser
Referens: MV037G_Föreläsning_Åhörarkopia_Syra_Bas.md
```
