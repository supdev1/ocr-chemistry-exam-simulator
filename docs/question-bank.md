# Chemistry Revision — Supabase question bank

Shared by every app in the **Chemistry revision** project. Supabase project **Chemistry Revision** (`xnwdrahlkndisnxsfdji`, eu-west-1, `https://xnwdrahlkndisnxsfdji.supabase.co`). Last updated 6 Nov 2026.

## Apps using this database

| App | Where | Reads | Writes |
|---|---|---|---|
| Exam simulator | GitHub `supdev1/ocr-chemistry-exam-simulator`, branch `main`; live at https://ocr-chemistry-exam-simulator.onrender.com | modules, papers, topics, questions (incl. `part_text`), question_topics, question_contexts, question_context_links, storage `question-images` | `attempts` (insert, then update when a mark is overridden) |
| Chemistry Revision practice app | GitHub `supdev1/OCR-A-Level-Revision-Quiz`, branch `main`; live at https://ocr-a-level-revision-quiz.onrender.com | topics (`scheme = 'plan'`), question_topics, questions | — |

Each app has its own repository and Render site. Upload each app's `index.html` only to its own repository — uploading to the other one replaces that live site.

## Tables

- **papers** — 27 OCR H432 papers, 2017–2025 (`year`, `series`, `paper_number` 1–3, `paper_title`).
- **modules** — the six OCR modules (`number` 1–6).
- **questions** — 1,093 question parts. `question_ref` is like `Q18(a)(ii)`; the number after `Q` groups parts of one question within a paper. `question_type` (mcq, short_answer, calculation, extended_response, structure_drawing, mechanism_or_pathway, graph, spectroscopy, apparatus_drawing); `delivery_mode` = `online` or `download_only` (needs a drawn answer). `mark_scheme` (MCQ answer letter first), `revise_note`, `image_path` / `mark_scheme_image_path` (objects in the private `question-images` bucket — use signed URLs).
- **topics** — two schemes: `ocr` (45 specification topics, codes like `5.1.3`) and `plan` (96 revision topics with `section_title` and `revision_guide`). Codes overlap between schemes but mean different things — always filter by `scheme`.
- **question_topics** — every question has exactly one primary topic in each scheme (`is_primary = true`) plus optional secondary topics.
- **attempts** — a student's mark on a question (`user_id` = `auth.uid()`). Students can insert, read and update their own rows.
- **module_status**, **marking_log**, views **question_topics_flat**, **module_performance** — added by the practice app.

## Shared question context (new, 6 Nov 2026)

Every part of a multi-part question repeats the question's context in `question_text`. That context is now also stored once:

- **question_contexts** (`id`, `paper_id`, `question_number`, `sort_order`, `context_text`) — each repeated context paragraph, once per question.
- **question_context_links** (`question_id`, `context_id`, `position`) — the contexts a part needs, in display order.
- **questions.part_text** — the part's own text without those contexts.
- **questions.question_text is unchanged.**

**Which to use**

- Showing **one part on its own** (flashcards, a single practice question): use `question_text` as before. Nothing needs to change.
- Showing **several parts of the same question together** (an exam paper, a whole question), in two steps:
  1. **Contexts:** show each linked context once, before the first displayed part that needs it, then each part's `part_text`.
  2. **Sentence tidy-up:** context often grows from part to part ("Step 1 …", then "Step 1 … Step 2 …") or is restated. Within one question, for every paragraph except a part's final one (its actual question):
     - drop it if an already-shown paragraph contains it word for word;
     - otherwise drop sentences at its start that were already shown (any length), and later sentences of 40+ characters that were already shown, or whose every word, number and formula appears in one sentence already shown (ignore little words such as "the", "of", "is");
     - only drop a table if an identical table was already shown.

  The reference implementation is `contextSplit` / `tidyContextPara` in the simulator's `index.html`. On the 60 questions that needed it, step 2 cuts what is shown by a further 7%, with no part left empty and no part's question changed.

Fetch with supabase-js: `from('questions').select('id, question_ref, part_text, question_context_links(position, question_contexts(id, context_text))')`.

**How it was split.** A paragraph counts as shared context when it is in the opening run of a part, is not the part's last paragraph, and appears word for word (ignoring spacing and capitals) in another part of the same question. Reworded restatements are never merged, so no different data is hidden. Rebuilding text as contexts + `part_text` reproduces `question_text` exactly for all 1,093 questions. A full multi-part question now shows 69% of the stored text.

**Review (done 6 Nov 2026).** 133 parts in 60 questions restated context in different wording. All were reviewed and `context_needs_review` cleared:
- 80 are handled by the sentence tidy-up above.
- 43 look similar but carry different content (another compound, other data, a different route) and must stay.
- 10 restate context with a formula or detail added or dropped. These were left as they are: a sentence of repetition is harmless, and editing would mean rewriting exam text.

The view `questions_needing_context_review` lists anything flagged in future.

**After adding or editing questions,** rebuild that paper only, so reviewed papers are not overwritten:

```sql
select * from public.rebuild_question_contexts(<papers.id>);   -- one paper
select * from public.rebuild_question_contexts();              -- everything (overwrites hand corrections)
```

The function is not callable by `anon` or `authenticated` users; run it from the SQL editor.

## Access rules (row-level security)

- Read for everyone (signed in or not): modules, papers, topics, question_topics, questions, question_contexts, question_context_links, storage bucket `question-images`.
- `attempts`, `module_status`: each signed-in student only sees and changes their own rows.
- The public (anon) key is safe in a web page; never put the service-role key in a page.

## Changes made on 6 Nov 2026

1. Signed-in students can read `questions` (previously only signed-out visitors could).
2. Students can update their own `attempts`.
3. Added `question_contexts`, `question_context_links`, `questions.part_text`, `questions.context_needs_review`, view `questions_needing_context_review` and function `rebuild_question_contexts(paper_id)`.
