# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a mobile-first quiz/exam preparation PWA for Chinese nursing certification exam preparation. The main application is a single HTML file (`quiz_app_v3.6.html`) that runs entirely in the browser with localStorage persistence.

## Commands

### OCR Pipeline (for processing photographed question bank pages)

```bash
# Work in project directory: /Users/chenlian/Quiz_Minimax/

# Convert HEIC to PNG
mkdir -p .ocr_work/full_png
for f in *.heic; do
  base=$(basename "$f" .heic)
  sips -s format png "$f" --out ".ocr_work/full_png/${base}.png"
done

# Run OCR with Swift Vision
swift scripts/vision_ocr.swift .ocr_work/full_png/*.png > .ocr_work/vision_ocr.json

# Extract explanations and generate quizpack (one command)
# Output is saved next to input CSV
python3 scripts/extract_quiz_content.py \
  .ocr_work/vision_ocr.json \
  第二篇/questions.csv \
  --deck-name "第二篇 外科护理学" \
  --chapter-name "第一章 尿路结石患者的护理"
```

### Importing Quizpacks

Open `quiz_app_v3.6.html` in a browser and use the import button to load `.quizpack` files.

## Architecture

### Quizpack Format (JSON)

```json
{
  "version": "1.0",
  "name": "Deck Name",
  "chapters": [{
    "id": "ch01",
    "name": "Chapter Name",
    "questions": [{
      "id": "ch01q001",
      "type": "single" | "multi",
      "stem": "Question text",
      "options": ["A. Option 1", "B. Option 2", ...],
      "answer": ["A"],
      "explanation": "Answer explanation",
      "questionPage": "P.1",
      "answerPage": "P.2"
    }]
  }]
}
```

### Practice Modes

- **练习 (Practice)** - Questions never attempted
- **易错题 (Hard)** - Questions with error rate above threshold
- **重练 (Redo)** - All questions in chapter
- **复习 (Review)** - Attempted but not mastered
- **冲刺 (Sprint)** - Random cross-chapter sampling

### State Management (localStorage)

- `quiz_decks` - Deck metadata array
- `quiz_deck_{id}` - Individual deck data
- `quiz_stats_{id}` - Per-question statistics (attempts, errors, mastery streak)
- `quiz_activity` - Daily activity tracking
- `cfg_*` keys - User configuration (theme, thresholds, etc.)

### Question Types

- `single` - Click to answer immediately after selection
- `multi` - Select all correct options, then submit

## OCR Edge Cases

The `extract_quiz_content.py` script handles these OCR edge cases automatically:
- **OCR numbering errors**: Detects jumps (e.g., 14 → 5) and corrects via answer matching
- **Combined answer format**: Parses "5~6B、D【解析】" as Q5(B) + Q6(D) sharing explanation
- **Cross-page explanations**: Concatenates text split by page numbers
- **Reference resolutions**: "参见 A2 型题第2题解析" → actual explanation text
- **Whitespace variations**: Handles "5. A【解析】" vs "5.A【解析】"

## Question Bank Structure

The question bank contains **5篇 (parts) with 52章节 (chapters)**. The `第二篇/` directory holds source images and OCR work for one part. The `目录/` directory contains the table of contents. Additional parts will be added as they are processed.

## OCR Processing

The main script is `scripts/extract_quiz_content.py` (also available as system skill `~/.claude/skills/quiz-extraction/`).

Input CSV should have columns: question_type, question_number, shared_stem, question, option_A-E, answer, explanation (explanation can be empty initially).
