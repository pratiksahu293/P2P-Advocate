# P2P Advocate Demo — Architecture

A Langflow visual AI workflow that assists with prior-authorization insurance appeals. Given a case description, it researches the applicable payer policy, builds an appeal argument, simulates a payer's pushback, drafts a rebuttal, and logs the outcome to a tracking spreadsheet.

## Flow overview

1. **Input.** A case description (patient details, diagnosis/treatment category, payer's denial reason, clinical findings) is pasted directly into the Playground chat.

2. **P2P Orchestrator** (top-level agent, `gemini-flash-lite-latest`) receives the case as its task. Its own instructions define the sequence of sub-agents it must call as tools, plus rules for how to assemble and format the final output. Everything downstream happens because the Orchestrator calls it, in order, carrying information forward from one step to the next.

3. **Triage Agent.** First sub-agent called. Produces an initial win-probability estimate and urgency rating from a first-pass read of the case, before any policy or evidence lookup.

4. **Evidence Retrieval Agent.** Searches the Policy Corpus (see below) to find the specific policy criteria relevant to the case, and reports whether the case's clinical facts satisfy them.

5. **Argument Builder Agent.** Constructs the appeal argument from what Evidence Retrieval found — citing specific policy language and clinical facts — and flags any gaps (e.g., missing quantitative data).

6. **Payer_Simulation_Agent.** Role-plays a skeptical payer-side reviewer and generates a realistic simulated objection to the argument.

7. **Rebuttal synthesis** (handled by the Orchestrator itself, not a separate agent). Synthesizes a rebuttal to the simulated objection using only the evidence/argument already produced, honestly conceding real gaps rather than overstating the case.

8. **Reconciliation** (also the Orchestrator). Reconciles Triage's initial win-probability/urgency estimate against what Evidence Retrieval and Argument Builder actually found, adjusting the final numbers if the deeper analysis changed the picture.

9. **Outcome_Capture_Agent.** Appends one row (case ID, win probability, urgency, argument summary, call brief summary, payer objection, rebuttal, outcome, date) to the "P2P Advocate Outcome Log" Google Sheet, via the Composio MCP tool. Only reports success if the tool's response confirms the write; otherwise reports the actual failure.

10. **Final output assembly** (Orchestrator). Assembles everything into two required sections: **Case Summary** (win probability, urgency, argument summary, call brief summary, payer objection, sheet-logging confirmation) and **Call Prep Detail** (full argument with citations, objection/rebuttal pair, explicitly flagged gaps).

11. **Clean Final Summary** (custom Python component). Strips any preamble before the first `**Case Summary**` header, so only the clean, final two-section output is shown.

12. **Chat Output.** Displays the cleaned response in the Playground.

## Key components

### Policy Corpus (Chroma Vector Store)

The knowledge base the Evidence Retrieval Agent searches to ground its claims in actual policy language, rather than the LLM guessing what a payer's coverage criteria might say.

Payer policy documents are loaded into a Chroma vector store, which breaks them into chunks and converts each chunk into an embedding (a numerical representation of its meaning). When Evidence Retrieval needs to check whether a case meets policy criteria, it converts the case's clinical facts into that same embedding space and pulls back the most semantically similar policy chunks — rather than doing a keyword search that could miss a match due to wording differences. This is retrieval-augmented generation (RAG): it's what lets Argument Builder cite specific policy language with confidence instead of fabricating plausible-sounding but made-up criteria.

### Composio (MCP server)

Composio is what lets agents actually *do* things in the outside world (Google Sheets, in this flow) instead of just producing text. The `composio` node in Langflow acts as an MCP (Model Context Protocol) server: a middleman exposing a catalog of external-app actions as callable tools to any Agent component wired into it.

Three meta-tools do the real work in this flow:

- `COMPOSIO_SEARCH_TOOLS` — looks up which specific action is needed (e.g., "find the Google Sheets append action") without hardcoding the exact tool name.
- `COMPOSIO_GET_TOOL_SCHEMAS` — fetches the parameters that action expects.
- `COMPOSIO_MULTI_EXECUTE_TOOL` — executes the call (e.g., `GOOGLESHEETS_SPREADSHEETS_VALUES_APPEND`), authenticated using the connected Google account.

Authentication is handled via Composio's "Connected Accounts" system — an OAuth token tied to the Google account, independent of whatever label the connection is given.

Two component settings matter for reliability:
- **Use Cached Server** — disabled, so each run gets a fresh MCP connection rather than potentially reusing stale session state.
- **Tool Execution Timeout** — raised to 60 seconds, to give slower API calls room to complete.

## Known issues resolved

- **Stale OAuth token**: caused persistent 404s on the Sheets append. Fixed by reconnecting the Google Sheets account in Composio's dashboard.
- **Spreadsheet ID typo**: a lowercase `l` vs. capital `I` in the hardcoded Sheet ID (visually near-identical) caused 404s pointing at a nonexistent spreadsheet, even with a valid token. Fixed by copying the ID directly from the sheet's URL bar rather than retyping it.
- **Output over-trimming**: an early version of Clean Final Summary used last-match / `rfind` logic that could truncate the response if matching text recurred later in the output. Fixed by anchoring on the first occurrence of the `**Case Summary**` header.
- **Rebuttal / Case ID fidelity**: the sheet log initially showed a paraphrased rebuttal and a generic placeholder Case ID ("CASE-001") instead of the real values. Fixed by tightening both the Orchestrator's and Outcome_Capture_Agent's instructions to require verbatim reuse of the original identifiers, with an explicit fallback ("Not provided") if genuinely absent.
- **False success reporting**: Outcome_Capture_Agent could claim a row was logged even when the write failed. Fixed by requiring it to only report success when the tool's response explicitly confirms a written range/row number.

## Not yet built

- Text-to-speech narration of the case summary (for hands-free listening), via Composio's ElevenLabs toolkit (`ELEVENLABS_TEXT_TO_SPEECH`). Considered NotebookLM's audio overview feature first, but that requires NotebookLM Enterprise (a licensed Google Cloud product), not the free consumer NotebookLM.
- Re-connecting the CSV → Loop → Parser batch pipeline for automated multi-case processing (deliberately deferred in favor of manual, one-case-at-a-time testing).
