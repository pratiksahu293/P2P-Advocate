# P2P Advocate Demo

A multi-agent Langflow workflow that assists with prior-authorization insurance appeals. Given a case description, it triages the case, retrieves the applicable payer policy, builds a medical-necessity argument, simulates a payer's pushback with a synthesized rebuttal, and logs the outcome to a tracked Google Sheet.

## What it does

1. Takes a case description (patient details, diagnosis/treatment category, payer's denial reason, clinical findings) as input.
2. Triages the case for an initial win-probability and urgency estimate.
3. Retrieves matching payer policy criteria from a vector-store knowledge base ("Policy Corpus").
4. Builds an appeal argument grounded in that policy language.
5. Simulates a skeptical payer's objection, and synthesizes an honest rebuttal to it.
6. Reconciles the final win-probability/urgency against the deeper analysis.
7. Logs the case (win probability, urgency, argument, call brief, objection, rebuttal, outcome) to a Google Sheet via Composio.
8. Returns a clean two-part summary: **Case Summary** and **Call Prep Detail**.

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the full step-by-step breakdown of every agent and component in the flow.

## Requirements

- [Langflow](https://www.langflow.org/) to import and run the flow.
- A Google AI Studio API key (Gemini), set as the `GOOGLE_API_KEY` environment variable — **not included in this export**.
- A [Composio](https://composio.dev/) account with a connected Google Sheets account, used for logging case outcomes.
- A Google Sheet named "P2P Advocate Outcome Log" with a header row: `Case ID, Win Probability, Urgency, Argument Summary, Call Brief, Payer Objection, Rebuttal, Outcome, Date Logged`.

## Setup

1. Import `P2P Advocate.json` into Langflow.
2. Set your `GOOGLE_API_KEY` environment variable (or configure it as a Global Variable in Langflow).
3. Connect your Google account to Composio's Google Sheets integration, and update the spreadsheet ID in the Outcome_Capture_Agent's instructions to point at your own sheet.
4. Open the Playground and paste in a case description to run it.

## Files

- `P2P Advocate.json` — the exported Langflow flow.
- `ARCHITECTURE.md` — detailed explanation of every agent, component, and the reasoning behind key design decisions.

## Status

This is a demo/testing project — cases are currently run manually, one at a time, through the Langflow Playground.
