# Explainer for the Decisions API

This proposal is an early design sketch by the Google Chrome Built-in AI Team to describe the problem below and solicit feedback on the proposed solution. It has not been approved to ship in Chrome.

## Proponents

- Google Chrome Built-in AI Team

## Participate

- https://github.com/explainers-by-googlers/decisions-api/issues

## Introduction

Web applications and agentic workflows increasingly need to make fast, structured decisions over unstructured input and page state: *Which search filters match this natural-language query? Which UI action or tool fulfills the user's goal? Would this draft trigger a moderation review or miss key details before submission? What does this page section or form field represent?* Today, developers must choose between brittle heuristics, cloud APIs with privacy and latency trade-offs, text generation models with high resource requirements, or custom model integrations that increase user and developer burden.

This living document explores a potential Web API (`window.DecisionModel`) for fast, type-safe, on-device machine learning assistance with decision-making, evaluation, classification, and ranking. Recent machine learning advances have shown that scoring predefined options directly in a single pass unlocks major gains in speed, reliability, and efficiency:

> *"System One models skip text generation. Given state and a bounded question with predefined answer types, they predict probabilities in a single forward pass. This produces fast, low-cost predictions with scores that express the model’s confidence."*
> — [System One: fast judgments and deliberate checks](https://system-one-explainer.netlify.app/)

This shows promising ways to aid users and applications in ways currently not possible with monolithic generative reasoning, task-specific client-side and server-side models, classic heuristics, and other means of making structured predictions:
- **Parallel, Single-Pass Execution:** Evaluates an input against multiple independent questions and option sets in a single pass rather than generating text word-by-word.
- **Guaranteed Type Safety:** Scores the caller's predefined options directly, so it cannot hallucinate invalid labels or syntax errors.
- **Calibrated Probabilities & Confidence:** Returns calibrated probabilities, expected scores on ordered scales, and a confidence measure so applications can act when confident or ask the user when uncertain.
- **Small On-Device Footprint:** Compact models (X00M parameters, X00MB quantized) can achieve high accuracy on easy-to-medium structured decision tasks in tens or hundreds of milliseconds across consumer CPUs, GPUs, and NPUs. Larger models can provide additional capabilities to support more complex problems as needed.

## Goals

- **Responsive, low-friction user experiences:** Help end-users find relevant actions, navigate complex interfaces, and get immediate feedback as they type or interact, without waiting on network round-trips or multi-second text generation.
- **Privacy-preserving local assistance:** Keep sensitive user drafts, browsing context, and personal queries on the user's device during evaluation, classification, and routing.
- **Broad device accessibility:** Deliver reliable on-device ML decisions on everyday consumer hardware without draining battery or memory.
- **Predictable, trustworthy application behavior:** Provide calibrated probabilities and confidence scores over fixed choices, enabling applications to selectively act upon or defer decisions.

## Non-goals

- **Open-ended text generation or chat:** Generating prose, summaries, or conversational replies is out of scope and served by generative APIs like the [Prompt API](https://github.com/webmachinelearning/prompt-api) (`LanguageModel`) and [Summarizer API](https://github.com/webmachinelearning/writing-assistance-apis#summarizer-api) (`Summarizer`).
- **Complex analytical reasoning:** Providing chain-of-thought reasoning or other means for solving complex analytical problems is out of scope and better suited to larger generative models.
- **Replacing application policy:** Applications remain responsible for weighing error costs, deciding whether to act or defer action, and respecting user autonomy.
- **Executing arbitrary custom model weights:** Running custom neural network graphs directly in the browser is addressed by lower-level APIs like WebNN and WebGPU.

## User research

Early developer prototypes and community explorations indicate strong demand for low-latency, local semantic decisions that avoid the resource overhead of full text generation and the privacy trade-offs of cloud calls.

- TODO: Document additional user research findings as usability studies and experiments progress.

## Use cases

Keeping decisions local, fast, and probabilistic helps solve several problems end-users face on the web today:

### Use case 1: Natural Language to Structured Filters & Semantic Search

Users searching trip booking or hotel reservation sites, product catalogs, or local-first apps often express multi-part goals in everyday language (e.g., *"dog-friendly hotel downtown that won't break the bank"* or *"under $50 waterproof jacket"*). Rather than forcing users to manually toggle dozens of checkboxes and dropdowns or sending private queries to a remote server, the application can convert natural-language input directly into its predefined filter controls (binary toggles, categorical options, and ordinal price or rating tiers) and re-rank candidate items on the user's device.

### Use case 2: Instant Intent Routing & Command Discovery

Users interacting with support portals, settings pages, or application command palettes often struggle to locate the right workflow or tool unless they know the exact terminology the site expects. Sites can offer a means to connect user goals in everyday language (e.g., *"let my coworkers view this file"*) to the right action or resource, while keeping their private input on-device.

### Use case 3: Real-Time Writing Feedback & Pre-Submission Checks

Users filling out forms, filing bug reports, or posting in community forums frequently discover only after submitting (or waiting on a remote check) that their draft is missing key details, violates community guidelines, or accidentally contains personal contact information (PII). Users benefit from instant, private, on-keystroke feedback while they are still writing.

### Use case 4: Adaptive Interfaces, Autofill & Accessibility

Users navigating information-dense pages, complex web forms, or large collections of tabs and products can experience cognitive overload. Evaluating page content and DOM context locally lets web apps and browser extensions (such as password managers or form-filling assistants) classify non-standard form fields for accurate autofill, group related items, highlight likely next steps for keyboard or switch users, or surface reading aids without sending browsing activity or page structure to a remote server.

### Common Decision Tasks

These user scenarios rely on three question types (`boolean`, `choice`, and `score`) for binary verification, categorical routing, and ordinal scoring:

| Task Pattern | Question Type | Description | Web Examples |
| :--- | :--- | :--- | :--- |
| **Verification & Detection** | `boolean` | Check whether input satisfies binary policy rules, adequacy criteria, or risk checks (`"true"` / `"false"`). | Verifying a draft or AI response meets guidelines; flagging toxicity, spam, or accidental PII; checking if a policy condition is met. |
| **Classification & Routing** | `choice` | Assign content to categorical options, route user intent to workflows or tools, or extract discrete fields. | Converting search text into predefined trip or catalog filters; routing a request to `billing` or `returns`; classifying form fields for password or address autofill. |
| **Scoring & Ranking** | `score` | Rate degree or intensity along an ordered rubric to evaluate or rank items (`options` is optional and defaults to `"1"`–`"5"`). | Scoring customer tone (`calm` to `angry`) or incident severity (`cosmetic` to `critical`); rating response helpfulness (`1`–`5`); ranking candidate results. |

## Potential Solution

We are exploring a three-step workflow on `window.DecisionModel`:
1. **Define a schema** with context and one or more questions (`boolean`, `choice`, or `score`, with required `options` for `choice` and optional `options` for `score` defaulting to `"1"`–`"5"`), check readiness with `DecisionModel.availability(schema)`, and create a session via `DecisionModel.create(schema)`.
2. **Pass the input** (such as text or page state) to `model.decide(input)`.
3. **Receive a structured result** keyed by question `id`, containing the winning `label`, option `probabilities` (keyed by option `label`), `confidence`, and `expectedScore` (populated on `score` decisions, weighted by parsed numeric option `label`s when all labels in the question are finite numbers, or `1..N` ordinal indices otherwise).

### How this solution would solve the use cases

#### Use case 1: Natural Language to Structured Filters & Semantic Search

A hotel reservation or trip booking site can convert a user's natural-language search into a set of predefined UI filters (`pet_friendly`, `property_type`, and `price_tier`) in a single local pass—applying high-confidence filters automatically and offering suggested filter chips when the query is ambiguous:

```js
// 1. Define the filter schema for a hotel search UI
const schema = {
  context: "Hotel reservation search filter converter.",
  expectedInputs: [{ type: "text", languages: ["en"] }],
  questions: [
    {
      id: "pet_friendly",
      type: "boolean",
      prompt: "Does the traveler require pet-friendly accommodations?"
    },
    {
      id: "property_type",
      type: "choice",
      prompt: "Which property type best matches the traveler's request?",
      options: [
        { label: "hotel", description: "Standard hotel, resort, or boutique inn" },
        { label: "rental", description: "Entire apartment, house, or cabin rental" },
        { label: "hostel", description: "Shared dormitory or budget hostel" },
        { label: "any", description: "No specific property type mentioned" }
      ]
    },
    {
      id: "price_tier",
      type: "score",
      prompt: "Rate the target price tier from 1 (budget) to 4 (luxury).",
      options: [
        { label: "1", description: "Budget or inexpensive stay" },
        { label: "2", description: "Moderate or mid-range price" },
        { label: "3", description: "Upscale or premium property" },
        { label: "4", description: "Luxury or five-star resort" }
      ]
    }
  ]
};

const status = await DecisionModel.availability(schema);

if (status === "available" || status === "downloadable") {
  const model = await DecisionModel.create(schema);

  // 2. Evaluate the user's natural-language query in a single local pass
  const query = document.querySelector("#search-input").value;
  // e.g., "dog-friendly hotel downtown that won't break the bank"
  const result = await model.decide(query);

  // 3. Inspect the result object (keyed by question id)
  // {
  //   pet_friendly: {
  //     id: "pet_friendly", label: "true", confidence: 0.98,
  //     probabilities: { true: 0.98, false: 0.02 }
  //   },
  //   property_type: {
  //     id: "property_type", label: "hotel", confidence: 0.94,
  //     probabilities: {
  //       hotel: 0.94,
  //       rental: 0.03,
  //       hostel: 0.01,
  //       any: 0.02
  //     }
  //   },
  //   price_tier: {
  //     id: "price_tier", label: "1", expectedScore: 1.24, confidence: 0.88,
  //     probabilities: { "1": 0.88, "2": 0.04, "3": 0.04, "4": 0.04 }
  //   }
  // }

  // Apply confident filter selections; show suggested filter chips when ambiguous
  if (result.pet_friendly.confidence > 0.85) {
    setPetFilter(result.pet_friendly.label === "true");
  }
  if (result.property_type.confidence > 0.85) {
    setPropertyFilter(result.property_type.label, result.price_tier.expectedScore);
  } else {
    showSuggestedFilterChips(result.property_type.probabilities);
  }

  model.destroy();
}
```

#### Use case 2: Instant Intent Routing & Command Discovery

An application command palette can match a user's everyday phrasing to available commands without requiring exact keyword matches:

```js
const actionMatcher = await DecisionModel.create({
  context: "Document editor command palette",
  questions: [
    {
      id: "command",
      type: "choice",
      prompt: "Which command best fulfills the user's goal?",
      options: [
        { label: "export_pdf", description: "Download or save the document as a PDF" },
        { label: "share_link", description: "Invite collaborators or copy a sharing link" },
        { label: "archive_doc", description: "Move the document to trash or archive" }
      ]
    }
  ]
});

const { command } = await actionMatcher.decide("let my coworkers view this file");
// command -> { id: "command", label: "share_link", confidence: 0.93, probabilities: { ... } }
```

#### Use case 3: Real-Time Writing Feedback & Pre-Submission Checks

A bug tracker or forum form can check a user's draft locally as they type, nudging them if reproduction steps are missing or if sensitive credentials were pasted accidentally:

```js
const draftChecker = await DecisionModel.create({
  context: "Bug report submission form pre-check",
  questions: [
    {
      id: "has_repro_steps",
      type: "boolean",
      prompt: "Does the draft include steps to reproduce the issue?"
    },
    {
      id: "contains_pii",
      type: "boolean",
      prompt: "Does the draft contain personal contact info, API keys, or passwords?"
    }
  ]
});

const { has_repro_steps, contains_pii } = await draftChecker.decide(draftText);
if (contains_pii.label === "true" && contains_pii.confidence > 0.8) {
  showWarning("Please remove personal info or credentials before posting.");
} else if (has_repro_steps.label === "false" && has_repro_steps.confidence > 0.8) {
  showHint("Adding reproduction steps will help resolve this issue faster.");
}
```

#### Use case 4: Adaptive Interfaces, Autofill & Accessibility

A form assistant or password manager extension can evaluate the text surrounding a non-standard input field locally to determine what credential or data it expects, without sending page DOM structure to a remote server:

```js
const fieldClassifier = await DecisionModel.create({
  context: "Form assistant classifying a non-standard input field from surrounding DOM labels",
  questions: [
    {
      id: "field_purpose",
      type: "choice",
      prompt: "What type of user credential or data does this input field expect?",
      options: [
        { label: "username", description: "Account email, login ID, or handle" },
        { label: "current_password", description: "Existing password for signing in" },
        { label: "new_password", description: "New or updated password being created" },
        { label: "one_time_code", description: "Two-factor verification or SMS code" },
        { label: "other", description: "Unrelated input field" }
      ]
    }
  ]
});

const { field_purpose } = await fieldClassifier.decide(
  'Label: "Enter the 6-digit security code sent to your phone" | Placeholder: "000000"'
);
if (field_purpose.label !== "other" && field_purpose.confidence > 0.85) {
  offerAutofillSuggestion(inputElement, field_purpose.label);
}
```

## Detailed design discussion

### Core Requirements & Option Behavior

To make on-device decision models dependable for the web, the design targets several key properties:
- **Strict Schema Conformance:** Outputs always match one of the developer's supplied `options` with valid numeric scores, avoiding fragile string parsing or out-of-schema values.
- **Calibrated Confidence & Policy Separation:** Raw model scores are often overconfident. Providing calibrated `probabilities` and a normalized `confidence` signal lets application code decide when to act automatically versus when to ask the user.
- **Parallel Multi-Question Evaluation:** Grouping multiple questions into one schema lets the runtime encode the input once and answer all questions in parallel, saving latency and battery.
- **Predictable Option Behavior:**
  - *Practical Option Limits:* Single-pass evaluation works best with a bounded number of choices per question (e.g., 2–16 options in compact models); larger catalogs can be filtered in stages.
  - *Graceful Fallbacks & Ambiguity:* Schemas can include catch-all options (like `"other"` or `"none"`). When options overlap or inputs are unclear, scores reflect that uncertainty with lower `confidence`.
  - *Order Independence:* The order in which options are listed should not bias their scores.
- **Clear Input & Context Limits:** On-device models have finite context windows. Similar to the [Prompt API](https://github.com/webmachinelearning/prompt-api), developers need ways to check how much capacity a schema and input use, and receive clear errors rather than silent truncation when limits are exceeded.
- **Built-in AI Platform Alignment:** Session creation and inference should follow established Built-in AI conventions, including `AbortSignal` (`signal`) support for cancellation, `monitor` callbacks for download progress, explicit cleanup via `destroy()`, requiring transient user activation when `create()` initiates a model download, and `Permissions-Policy` integration (the DevTrial prototype reuses the `"language-model"` policy feature).

### Naming (`Decision API` / `window.DecisionModel`)

We welcome feedback on the name *Decision API* and its `window.DecisionModel` entrypoint. Potential alternatives include *Prediction*, *Evaluation*, *Assessor*, *Classification*, as well as whether to incorporate qualifying adjectives such as *Structured* or *Contextual* (e.g., *Structured Prediction API*, *Contextual Evaluation API*) and JS entrypoints.

### Result Shape & Uncertainty Controls

Does returning an object keyed by question `id` best serve developers, or should we also consider array or wrapper shapes? Should optional controls for trading compute effort for higher confidence (e.g., multi-step sampling hints or deeper uncertainty metrics) be exposed for advanced use cases?

### Batching Inputs (`decideBatch`)

Evaluating multiple questions on a single input happens in one pass. For scoring or ranking many inputs against the same schema (such as 50 feed items or open tabs), should `DecisionModel` offer a dedicated `decideBatch(inputs)` method?

### Input & Language Configuration

Using `expectedInputs: [{ type: "text", languages: ["en"] }]` in `availability()` and `create()` aligns with the Prompt API, verifies language coverage upfront, and prepares for future image or audio inputs. Would a simpler shorthand, like `expectedLanguages: ["en"]`, be preferable for common text-only tasks?

### Worker Support

How should `DecisionModel` support background workers (`DedicatedWorker`, `SharedWorker`, `ServiceWorker`) as proposals like [Permissions Policy for Workers](https://github.com/explainers-by-googlers/workers-permissions-policy) progress?

## Considered alternatives

### Brittle Client-Side Heuristics

Keyword rules and regular expressions are fast and local, but break on nuance, phrasing variations, typos, negation, and multilingual input.

### Server-Side AI APIs

Cloud AI endpoints are capable and require zero client download size, but introduce hundreds of milliseconds of network round-trip latency (blocking real-time on-keystroke UI), trigger privacy trade-offs by transmitting user drafts or page state off-device, and incur recurring or high-frequency server costs.

### Generative Prompt API (`LanguageModel`)

Using an on-device generative language model to output structured JSON or category labels is flexible and well-suited for open-ended writing, chat, summarization, and freeform information extraction. However, autoregressively generating tokens via client-side LLMs is orders of magnitude (e.g., 10–50x) heavier and slower than necessary when only a discrete choice or score is needed, consumes significant RAM and battery, and is not as directly amenable to producing calibrated option probabilities.

### Developer-Supplied Models (WebGPU / WebAssembly / WebNN)

Developers can run models today via libraries like Transformers.js and LiteRT.js. However, for most web products, curating training data, evaluating models, integrating runtimes, and bundling multi-megabyte model weights is a prohibitive operational hurdle. Users also face duplicated network and storage costs (potentially mitigated by [Cross-Origin Storage](https://github.com/WICG/cross-origin-storage)), and unmanaged resource contention across origins without browser-managed hardware scheduling.

### Fixed-Taxonomy API

Supporting a fixed set of standard taxonomies or questions keeps the API surface tiny, but is too narrow to support app-specific routing, custom verification rules, or domain-specific UI decisions.

## Security and Privacy Considerations

- **Privacy & No New Cross-Origin Data:** Inference runs locally on the user's device. The API is stateless: inputs are not sent over the network, saved to disk, or used to train models. Because `decide(input)` only evaluates data already available to the calling origin (with no access to cross-origin browsing history or profile state), it does not expose new cross-origin information to the site. Initiating a model download via `create()` requires transient user activation.
- **Security & User Agency:** User `input` is kept separate from developer `questions` and `options` so untrusted text cannot inject fake choices or alter the schema. Timing and numeric precision should be bounded to mitigate device fingerprinting. Model confidence is not execution authorization: state-changing or high-consequence actions should require explicit user confirmation regardless of confidence score.
- **Accessibility (a11y):** Sites can use fast local decisions to make interfaces easier to navigate. For example, by matching everyday phrasing or voice commands to page actions, highlighting likely next steps for keyboard users, or offering simpler summaries for dense text.
- **Internationalization (i18n):** Browsers can verify language support upfront via `expectedInputs` in `availability()` and `create()`, selecting a multilingual model or reporting `"unavailable"` rather than guessing on unsupported languages.
- **Device & Ecosystem Reach:** Because decision models are much smaller and faster than generative language models, they can run comfortably on a much wider range of consumer devices without straining memory or battery.
- TODO: Complete the [W3C Security and Privacy Self-Review Questionnaire](https://www.w3.org/TR/security-privacy-questionnaire/) and review interactions with [Chromium's Web Platform Security Guidelines](https://chromium.googlesource.com/chromium/src/+/master/docs/security/web-platform-security-guidelines.md).

## Stakeholder Feedback / Opposition

- **Web Developers & Prototype Authors:** Positive interest demonstrated through independent web libraries, extensions, and interactive demos exploring single-pass on-device decision models:
  - **[Open-Jev (`nico-martin/open-jev`)](https://github.com/nico-martin/open-jev):** Browser TypeScript library for single-pass typed decisions over text, running quantized ONNX decision models (`kev` and `open-jev`) locally via [Transformers.js](https://huggingface.co/docs/transformers.js/en/index) with WebGPU and WebAssembly.
  - **[System One Interactive Explainer & Jev × WebMCP](https://system-one-explainer.netlify.app/) ([GitHub](https://github.com/sdras/system-one-explainer)):** Interactive guide exploring categorical and ordinal decisions, confidence thresholds, and a Jev × WebMCP case study (with an [extension](https://chromewebstore.google.com/detail/gglnhcbhjfbmcpgccnmolbhejloflgkb) and [test site](https://shopping-webmcp-demo.netlify.app/)) for page tool routing with policy checks.
  - **[Web AI Studio](https://web-ai.studio/) & [WebAI Extension](https://web-ai.studio/extension) ([Chrome Web Store](https://chromewebstore.google.com/detail/webai-extension/lmjgpcigjcffnphimblhcoccjfefamcp), [GitHub](https://github.com/etiennenoel/web-ai.studio)):** Interactive [playground](https://web-ai.studio/playgrounds/decisions) for testing schemas and probability distributions, plus a [browser extension](https://github.com/etiennenoel/web-ai.studio/tree/master/extension) with a client-side [polyfill](https://github.com/etiennenoel/web-ai.studio/tree/master/extension/projects/content-script/src/polyfill) and [Laya runner](https://github.com/etiennenoel/web-ai.studio/tree/master/extension/projects/laya) executing [`litert-community/laya-LiteRT`](https://huggingface.co/litert-community/laya-LiteRT) models locally.
- **Browser Implementors & Standards Groups:**
  - TODO: Gather and link feedback from WebKit, Mozilla, W3C TAG, and Web Machine Learning Community Group discussions.

## References & acknowledgements

### External References & Inspirations

- **System One Framing & Workflow Design:**
  - [Introducing System One Models & Jev (TypeSafe AI)](https://typesafe.ai/blog/introducing-system-one-models-and-jev)
  - [System One: Fast Judgments and Deliberate Checks](https://system-one-explainer.netlify.app/)
- **Open Decision Models, Checkpoints & Runtimes:**
  - [Laya: Multilingual Non-Autoregressive System 1 Decision Engine (`NandhaKishorM/laya`)](https://github.com/NandhaKishorM/laya)
  - [laya-LiteRT: Laya Decision Encoders for LiteRT (`litert-community/laya-LiteRT`)](https://huggingface.co/litert-community/laya-LiteRT)
  - [Kev: Small Jev-like Decision Models (`jaredpalmer/kev`)](https://github.com/jaredpalmer/kev)
  - [Open-Jev: Browser-Focused TypeScript Library for Typed Decisions (`nico-martin/open-jev`)](https://github.com/nico-martin/open-jev)
  - [Web AI Studio](https://web-ai.studio/) & [WebAI Extension (`etiennenoel/web-ai.studio`)](https://github.com/etiennenoel/web-ai.studio)

### Acknowledgements

- TODO: Add acknowledgements as community members and reviewers contribute feedback.
