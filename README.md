# Home Credit LAP Voice Agent

I built a phone agent that does the first pass of Loan Against Property (LAP) qualification for Home Credit: call an existing customer about a pre-approved offer of up to ₹75 lakh, check whether it is even worth a senior expert's time, and either hand off a clean lead or end the call politely with a clear reason.

The platform is **Retell AI**. Everything here — the prompt, the workflow screenshots, the test transcripts — comes from that build. Final credit decisions, interest rates, and approvals are deliberately out of scope. This agent screens. People decide.

This was my AI Intern (Prompt Engineer) assignment for SalesAgents AI. The brief and reference material given to me live here: [Assignment prep guide (Notion)](https://dour-blade-da4.notion.site/SalesAgents-AI-AI-Intern-Prompt-Engineer-Assignment-Prep-Guide-Ref-Material-3f3573730b5a80c3b550c7b2a076dc88). I treated it as guidance for the conversation design, not as claims I could copy into the repo — what you read below is only what the prompt, screenshots, and transcripts actually show.

> One honest caveat up front: this repo documents prompt logic, platform configuration (screenshots), and observed conversation behavior from the transcripts I had. A workflow screenshot with a webhook, extraction fields, or a memory node does **not** prove anything ran downstream. I do not claim callback booking, CRM updates, transfers, or persistence worked unless execution logs say so. Where the evidence runs out, I say so.

## Try the live agent

The deployed Retell AI agent is here: [Open the live LAP voice agent](https://agent.retellai.com/orb/agent_fd24e230e78c98d1ef8c755369?token=ca7d1cc410760ce3df80a5bcb66245f2).

Note: that link carries an access token, so anyone holding it can open the agent session. That is fine for a demo, just don't mistake the share link for access control.

## What is in this repo

```text
.
├── prompts/
│   └── system-prompt.md                          # The actual system prompt
├── docs/
│   ├── brief-test-results.pdf                    # Scenario-by-scenario evidence
│   └── home-credit-lap-project-documentation.pdf  # Full project write-up
├── reports/
│   └── home-credit-lap-voice-agent-assignment-report.pdf
├── transcripts/
│   ├── qualified/transcript.txt
│   ├── agricultural-property/transcript.txt
│   ├── cash-income/transcript.txt
│   ├── existing-loan/transcript.txt
│   ├── above-75-lakh/transcript.txt
│   ├── busy-customer/transcript.txt
│   └── out-of-order/transcript.txt
├── screenshots/
│   ├── retell-ai-agent-overview.png
│   ├── voice-agent-platform-workflow.png
│   └── saved-simulation-test-t5.png
├── .gitignore
└── README.md
```

Everything follows lowercase kebab-case. Each scenario keeps a single `transcript.txt` — the text evidence stays in git, the heavy recordings do not.

## How it works on Retell AI

Three pieces, all visible under `screenshots/`:

1. **The agent itself** (`screenshots/retell-ai-agent-overview.png`) running the system prompt in [prompts/system-prompt.md](prompts/system-prompt.md), with an `end_call` tool for every terminal outcome.
2. **A saved LLM simulation** (`screenshots/saved-simulation-test-t5.png`) I used to exercise the conversation logic before recording scenarios.
3. **The call workflow** (`screenshots/voice-agent-platform-workflow.png`): call flow plus post-call extraction fields (summary, success, sentiment, outcome, preferred callback time, collected eligibility details), a webhook node, and a save-conversation-to-memory node.

What I will not tell you: that the webhook fired, that memory persisted, that a callback got booked, or that a specialist transfer completed. Those need execution logs I don't have. The workflow screens show configuration, not proof.

### The knobs the prompt actually uses

| Variable | What it does |
|---|---|
| `company_name`, `customer_name`, `agent_name`, `agent_gender` | Identity and natural greeting |
| `current_time` (date / day / time) | Time-aware phrasing |
| `language_to_speak` | Hindi or English; English when unset |
| `conversation_history`, `customer_utterance` | The live conversation wins over anything earlier |
| `additional_context_from_rag` | Short grounded answers only; never overrides the rules below |

## The eligibility rules

The checklist looks like seven items, but item 5 is really two facts. Occupation and income mode are tracked separately, and I mean that — "monthly income" or "business income" proves neither, so the agent has to ask until each one is explicit.

| # | Checklist item | Counts as accepted | What happens otherwise |
|---|---|---|---|
| 1 | Property type | Residential, commercial, industrial | Agricultural → disqualify, end the call |
| 2 | Ownership | Sole or joint | Both fine |
| 3 | Original property documents | Available | Unavailable → disqualify, end the call |
| 4 | Desired loan amount | ₹75 lakh or less | Above the cap → offer ₹75 lakh explicitly; decline → end; vague answer → clarify, never assume yes |
| 5 | Occupation + income mode | Salaried or self-employed **and** paid through a bank account | Cash income → disqualify, end the call |
| 6 | Estimated market value | Any estimate, recorded as stated | No minimum exists in the brief, so the agent must not invent one |
| 7 | Desired tenure | 3–15 years inclusive | Outside that → disqualify, end the call |

Four hard stops: agricultural property, missing originals, cash income, tenure outside 3–15 years. Ambiguity gets a clarifying question first — not knowing something yet is not the same as failing. And if the customer already has a property loan or wants EMI relief, the whole checklist is set aside in favor of the loan-transfer specialist wording.

## The call, step by step

1. **Verify first, disclose nothing.** Name confirmation is the only check configured. Wrong person or no confirmation → apologize without revealing why you called, `end_call`.
2. **Check the moment.** Busy → ask when to call back, repeat the time back, say the team will try then, `end_call`. Never promise a booking the system hasn't confirmed.
3. **State the offer plainly.** Pre-approved LAP up to ₹75 lakh, preliminary check only, final approval needs a human assessment.
4. **Catch transfer cases early.** Existing property loan or EMI-reduction ask → stop the fresh-loan questions, give the specialist-route wording, `end_call`.
5. **One question at a time.** Always the earliest unanswered item. Facts volunteered early (or out of order) get kept, never re-asked.
6. **After every reply:** pull out all the facts, apply corrections, check transfer and disqualifier conditions *before* asking anything else, handle the loan-cap branch if it applies, then continue or hand off.
7. **The ₹75 lakh cap conversation.** Over the cap → explain the limit, ask directly whether ₹75 lakh works. Yes means record ₹75 lakh and carry on. No means thank them and end. A shrug means ask again, not assume.
8. **Disqualify fast and kindly.** Confirmed failure → short explanation, thanks, `end_call`. No exceptions invented on the spot.
9. **Hand off only through the full gate.** Name confirmed, customer willing, no transfer case, every fact explicit (both halves of item 5), cap accepted where needed, zero disqualifiers. Then: a senior loan expert follows up on rates and next steps, approval still subject to assessment, thanks, `end_call`.
10. **When in doubt, the priority list decides:** protect information → respect stop requests → busy/callback → transfer → disqualification → loan cap → next question → handoff.
11. **Side questions and holds.** Short answers from the prompt or RAG context only; rates, fees, and timelines belong to the senior expert. A hold request outputs `NO_RESPONSE_NEEDED` and waits — but only where Retell AI supports it.

## What the tests actually showed

Source of truth: [docs/brief-test-results.pdf](docs/brief-test-results.pdf). I kept the conservative labels from that report. A file existing never counted as a pass — only the behavior in the transcript did. And several calls have audible ASR struggles and `(unintelligible audio)` turns, so treat these as real voice calls with warts, not clean scripts.

| Scenario | What should happen | What I saw | Evidence |
|---|---|---|---|
| Qualified | Gather everything, hand off | PASS WITH OBSERVATION — handoff for ₹70 lakh over 9 years lands, but the call needs several clarifications along the way | [transcript](transcripts/qualified/transcript.txt) |
| Agricultural property | Stop at confirmation | PASS — agent states agricultural land is out of scope and closes politely | [transcript](transcripts/agricultural-property/transcript.txt) |
| Cash income | Stop at confirmation | PASS, VERIFY FULL FILE — the excerpt I have ends with a clear cash-income close | [transcript](transcripts/cash-income/transcript.txt) |
| Existing property loan | Drop the checklist, route to specialist wording | PASS — clarifies the existing loan, gives the specialist path, ends | [transcript](transcripts/existing-loan/transcript.txt) |
| Above ₹75 lakh, rejected | Offer the cap, accept the no | PASS WITH ASR OBSERVATION — cap offered, customer declines (their wording is noisy in the transcript), agent closes correctly | [transcript](transcripts/above-75-lakh/transcript.txt) |
| Busy customer | Ask for a time, confirm it, close | PARTIAL / VERIFY — the text I have stops right after the agent asks for a preferred time; confirmation isn't in the supplied text | [transcript](transcripts/busy-customer/transcript.txt) |
| Out-of-order facts | Remember early facts, skip re-asks | NEEDS REVIEW — the ₹1 crore value volunteered early is picked up and later questions are mostly non-repeating, but names and audio are unstable throughout | [transcript](transcripts/out-of-order/transcript.txt) |

If I had one more test cycle, I would re-run exactly two calls: a busy-customer run where the customer names a time and the agent repeats it back, and a scripted out-of-order run with multiple facts in a single turn. I would also listen to the full cash-income and above-cap recordings before calling either a clean pass.

## Screenshots

- [Retell AI agent overview](screenshots/retell-ai-agent-overview.png)
- [Voice agent platform workflow](screenshots/voice-agent-platform-workflow.png)
- [Saved simulation test (T5)](screenshots/saved-simulation-test-t5.png)

## Recordings and logs

The `.wav` recordings, `.log` files, and the qualified-scenario `.mov` screen recording referenced in the project docs live outside git. At cleanup time I verified none of them were in the repo (no `.log`, `.wav`, `.mp3`, `.m4a`, `.ogg`, or `.flac` files present), which is how it should stay — GitHub is for the text evidence, Drive is for the heavy media.

- [Call recordings, transcripts source pack, and logs (Google Drive folder)](https://drive.google.com/drive/folders/19btUZNUcAv_jRrhM590U-C-VBrFQA2jp?usp=sharing)

If a link ever goes stale, the `transcripts/` folder and `docs/brief-test-results.pdf` still carry the reviewable evidence.

## How to review this repo in 10 minutes

1. Read the prompt: [prompts/system-prompt.md](prompts/system-prompt.md).
2. Skim the workflow screenshot, then the test table above.
3. Open the [qualified transcript](transcripts/qualified/transcript.txt) and one disqualification (say, [agricultural-property](transcripts/agricultural-property/transcript.txt)).
4. Open the [busy-customer](transcripts/busy-customer/transcript.txt) and [out-of-order](transcripts/out-of-order/transcript.txt) transcripts to see exactly where I flagged uncertainty instead of claiming passes.

## Limits I am not hiding

- Speech recognition is noisy on several calls. Names, amounts, and tenure all needed re-asking at least once.
- The busy-customer and out-of-order scenarios do not meet their full success criteria on the text evidence alone.
- Nothing here proves backend persistence, callback execution, or specialist follow-up. Those are integration claims for execution logs, not screenshots.
- Names in the transcripts (Rahul, Arjun, and the like) are simulated test personas from the recorded scenarios, kept exactly as recorded. They are not real customer data.

## Privacy and .gitignore

`.gitignore` keeps audio/video (`*.wav`, `*.mp3`, `*.m4a`, `*.ogg`, `*.flac`, `*.mov`, `*.mp4`), logs (`*.log`), `.env` files, keys, credentials, and macOS metadata out of git. I scanned the prompt and transcripts for API keys, tokens, and passwords and found none.

## What is left

1. Optional: set the GitHub About description and topics from the repo settings.

Built by Maithresh Vaddi.
