# Police Report Bot — Fix Summary and Future Fixes

This document summarizes the fixes handled in the current pass and the items intentionally kept on the future list. Items that were skipped entirely are not included.

## Current-Pass Fixes

### Priority 1 — Product blockers and correctness fixes

#### Escape HTML in browser grading display
**Source:** I found  
**Files:** `src/ui.py`  
Added HTML escaping before user/model-generated text is inserted into `gr.HTML`. This protects the grading display from broken markup or unintended HTML rendering in officer messages, narrative excerpts, grammar excerpts, and style suggestions.

#### Use post-validation coverage in notes
**Source:** You found + I confirmed  
**Files:** `src/grader.py`  
Updated the notes builder to reference post-validation coverage instead of original pre-validation flags. This prevents rows from claiming an element was asked or supported after validation downgraded it.

#### Hide stale or irrelevant references in UI and PDF
**Source:** You found + I confirmed  
**Files:** `src/ui.py`, `src/session_logger.py`, `src/grader.py`  
Changed browser and PDF rendering so interview references appear only when final interview status is `full` or `partial`, and narrative references appear only when final narrative status is `full` or `partial`. Final grading rows no longer copy stale references after validation downgrades coverage.

#### Prevent out-of-order click crashes
**Source:** I found  
**Files:** `src/ui.py`  
Added guard clauses so early clicks on Send, End Interview, or Submit Report do not crash the app when session state is missing or incomplete.

#### Synchronize final grading output with validated coverage
**Source:** You found + I confirmed  
**Files:** `src/grader.py`  
Adjusted final grading rows so stale interview messages are not copied into `grading_results["content"]` when validation downgrades interview coverage. Raw debugging data can still remain in source coverage structures, but final UI/PDF output uses validated references.

#### Fix PDF table word wrapping
**Source:** You found + I confirmed  
**Files:** `src/session_logger.py`  
Converted PDF table values to wrapped ReportLab `Paragraph` cells and adjusted table width so long coverage, reference, grammar, and style text does not overlap adjacent columns.

#### Cap PDF references to prevent ReportLab row-height crashes
**Source:** I found after testing  
**Files:** `src/session_logger.py`  
Limited the number of interview and narrative references displayed in each PDF table cell. The PDF now shows a few representative refs and summarizes the rest with text such as `... and 22 more`, preventing a single table row from becoming taller than a page.

#### Fix browser table header contrast
**Source:** You found  
**Files:** `src/ui.py`  
Set `color:white;` directly on browser table `<th>` elements so Gradio theme CSS does not render black text on dark headers.

#### Lock workflow with confirmation gates
**Source:** You found + I confirmed  
**Files:** `src/interview_bot.py`, `src/ui.py`  
Added explicit workflow phases and confirmation steps. During interview, only the interview tab is available. Ending the interview requires confirmation and moves the user to the report tab. Submitting the report requires confirmation and starts grading. After grading, all tabs are visible, but interview and report inputs are disabled.

#### Add preliminary progress support during grading
**Source:** You found + I confirmed  
**Files:** `src/grader.py`, `src/ui.py`  
Passed the Gradio progress object into grading functions and enabled Gradio queueing. Testing showed the progress bar appears, but it still does not advance reliably through grading, so full progress-bar behavior remains on the future list.

#### Preload LanguageTool during interview
**Source:** You found + I confirmed  
**Files:** `src/grader.py`, `src/ui.py`  
Started LanguageTool initialization in a background thread when a session starts. The grammar checker still falls back to waiting for `get_tool()` if the officer finishes the interview/report before initialization completes.

#### Add crime-type scenario constraints
**Source:** You found  
**Files:** `src/scenario_generator.py`  
Added crime-type-specific prompt constraints for all current crime types so scenarios remain focused on the selected primary crime type and do not drift into unrelated crimes unless the checklist requires it.

#### Make interviewee responses shorter and more reserved
**Source:** You found  
**Files:** `src/interview_bot.py`  
Added role-specific response style instructions so victim, witness, and caller answers are concise, natural, and mostly limited to what the officer explicitly asked.

#### Count facts provided by the interviewee during the interview
**Source:** You found  
**Files:** `src/interview_bot.py`, `src/grader.py`, `src/ui.py`, `src/session_logger.py`  
Added response-provided coverage tracking. If a victim/witness/caller volunteers information in response to a question, and the officer later includes that information in the report, the grader can count it as interview coverage. The system still does not give credit when the officer writes a fact that was neither asked about nor provided by the interviewee.

#### Simplify response-coverage JSON to prevent Groq JSON failures
**Source:** I found after testing  
**Files:** `src/interview_bot.py`  
Removed `provided_fact` from the LLM-required JSON shape in `check_response_coverage()`. The app now stores the full actual interviewee response as the evidence context instead of asking the LLM to return a quote field, preventing invalid JSON when the model tries to return multiple quoted strings.

### Priority 2 — Scenario quality and grading accuracy

#### Clean legal context file
**Source:** You found + I confirmed  
**Files:** `source_docs/legal_context/legal_context.txt`  
Rewrote the legal context in plain language. Removed statute numbers, federal code citations, state-specific references, case-law/court references, and language that encouraged automatic co-occurring crime generation. Preserved general procedural concepts such as protection orders, firearm restrictions, probable cause, victim safety, evidence collection, and documentation requirements.

#### Display role as caller in browser session info
**Source:** You found  
**Files:** `src/ui.py`  
Changed the visible browser session info panel so the officer sees `Role: caller`. This is a cosmetic UI-only change and does not modify internal scenario role behavior.

#### Reduce repeated names, places, and scenario patterns with prompt variety rules
**Source:** You found + I confirmed  
**Files:** `src/scenario_generator.py`  
Added prompt-only variety instructions requiring the LLM to internally generate multiple options before selecting fictional names, places, settings, and scenario setup details. This avoids hardcoded name/address lists and does not rely on prior-scenario memory. Testing showed this is only a partial improvement, so stronger scenario diversity remains on the future list.

#### Add demographic diversity constraints
**Source:** You found  
**Files:** `src/scenario_generator.py`  
Expanded scenario variety instructions so the LLM varies age ranges, genders, household types, relationship dynamics, living situations, reporting-party roles, and scenario setups rather than defaulting to a young female victim/reporting party.

#### Improve narrative coverage matching prompt
**Source:** I found after reviewing grading behavior  
**Files:** `src/grader.py`  
Adjusted the narrative coverage prompt so the same phrase can support multiple checklist elements, relationship facts can support both relationship identification and relationship history, threat statements can support threat/fear elements, and concrete negative facts can count when the checklist asks whether something occurred. Also instructed the model not to count headings, checklist-question text, or empty strings as evidence.

### Priority 3 — Reliability and polish

#### Append UUID suffix to session IDs
**Source:** I found  
**Files:** `src/session_logger.py`  
Kept the existing session ID format and appended an 8-character UUID suffix to prevent file collisions when multiple users or sessions save during the same second.

#### Filter LanguageTool false positives for generated scenario words
**Source:** I found  
**Files:** `src/grader.py`  
Built a scenario-derived ignore list for generated proper nouns and skipped LanguageTool spelling/dictionary errors for those generated names and places. Real grammar, punctuation, sentence-structure, and repetition issues remain in the grading report.

## Future Fixes

### Progress bar reliability during grading
**Source:** You found + I confirmed  
The progress bar appears but still tends to sit at one early value during long grading operations. Future work should make progress updates visible between expensive LLM calls and ensure stage-level progress reaches the browser reliably.

### Grading fairness and coverage matching review
**Source:** You found + I confirmed  
Review and improve cases where relevant officer questions or relevant narrative facts are not credited. The grader should better match broad but valid questions, volunteered interviewee facts, relationship facts, threats, emotional state, and other overlapping checklist elements without requiring exact wording.

### Concrete “did not occur” facts
**Source:** You found + I confirmed  
Allow checklist elements to be satisfied by concrete negative facts where appropriate, such as `No children were present`, `No medical attention was needed`, `No weapons were mentioned`, or `The victim did not report sexual violence`, instead of treating those as missing.

### Stronger scenario diversity system
**Source:** You found + I confirmed  
The prompt-only variety fix helped only partially. Future work should use a stronger approach to reduce repeated demographic and relationship patterns without hardcoding lists, such as LLM-generated candidate scenario seeds, stronger validation of demographic variety, or another non-hardcoded variation mechanism.

### PDF role display consistency
**Source:** You found + I confirmed  
The browser session info displays `Role: caller`, but PDF/session metadata can still show the internal generated role such as `victim` or `witness`. Future work should decide whether all user-facing outputs should display `caller` consistently while preserving internal role behavior for response style.

### API key cleanup and environment secret loading
**Source:** I found  
Move API key handling to environment variables or managed secrets and remove bundled key files from the project.

### Interview contradiction checker changes
**Source:** I found  
Revisit contradiction detection design. The intended behavior needs to focus on whether the officer contradicts themselves or the final report contradicts established scenario facts, without treating bot dialogue as authoritative fact.

### Generate DOBs when children are present
**Source:** You found  
When children are part of a scenario, generate age, DOB, relationship, location during incident, and safety disposition.

### Add address, phone, and DOB for witnesses
**Source:** You found  
Generate complete identifying/contact details for all witnesses when witnesses are included in the scenario.

### Generate medical-response details when medical attention occurs
**Source:** You found  
When medical attention occurs, generate hospital, ambulance/EMS agency, paramedic details, treatment/refusal, transport destination, and relevant timing.

### Structured entity/name/info bank
**Source:** You found + I confirmed  
Create structured internal records for victim, suspect, caller, witnesses, children, places, EMS/hospital details, and other recurring entities so generated facts remain consistent.

### Quote-existence verification
**Source:** I found  
Verify that LLM-provided evidence snippets or “exact quotes” actually exist in the source narrative/interview text before showing them in the UI or PDF.

### Full JSON schema validation/retry system
**Source:** I found  
Add robust schema validation, retry handling, and safe fallbacks for LLM JSON outputs across scenario generation, interview coverage, response coverage, narrative coverage, grading, and validation.

### Professor agreement KPI
**Source:** You found  
Have a criminology professor or qualified evaluator review the app and target at least 80% agreement/approval with the app’s grading and instructional usefulness.
