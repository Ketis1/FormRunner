# FormRunner — LLM Generation Guide

This document tells you exactly how to generate a valid FormRunner import file when asked by a user. Read it fully before generating any JSON. Every rule here reflects how the app actually renders and validates forms — deviating will cause silent failures or broken UI.

---

## Your job

When a user asks you to create a form, you output a single `.json` file. Nothing else — no explanation wrapped around it, no markdown prose, just the raw JSON. The user drops it into `formrunner.html` and it works immediately.

---

## The envelope — always required

Every form file has this exact shape. Nothing lives outside `form`.

```json
{
  "form": {
    "id": "...",
    "title": "...",
    "description": "...",
    "version": "1.0",
    "settings": { ... },
    "sections": [ ... ]
  }
}
```

### `form.id`
- Lowercase letters, digits, underscores only. No spaces.
- Used as the filename prefix of exported results, e.g. `employee_survey_results_2024-11-15.json`.
- Make it short and descriptive: `"employee_survey"`, `"product_feedback"`, `"onboarding_checklist"`.

### `form.title`
- Shown as the large heading at the top of the form. Make it human-readable.

### `form.description`
- Optional. A subtitle or instruction paragraph shown below the title. Good place for: estimated completion time, anonymity notice, context.

### `form.version`
- Always set to `"1.0"` unless you have a reason to change it.

### `form.settings`
```json
{
  "allow_multiple_submissions": true,
  "show_progress_bar": true,
  "confirmation_message": "Thank you for completing the form."
}
```
- `show_progress_bar`: set `true` when the form has more than one section. Pointless on single-section forms but harmless.
- `confirmation_message`: shown on the thank-you screen after submission. Tailor it to the form's purpose.

---

## Sections

A form has one or more sections. Each section renders as its own card, one at a time. The user clicks Next/Previous to navigate between them.

```json
{
  "id": "section_demographics",
  "title": "About You",
  "description": "Optional intro text for this section.",
  "questions": [ ... ]
}
```

**When to use multiple sections:**
- The form is long (more than ~8 questions) — split by topic.
- There are clear conceptual groups (e.g. Personal Info / Work Experience / Open Feedback).
- A single-section form is perfectly valid for short forms.

**Section id rules:** lowercase, underscores, unique within the form. Example: `"section_personal"`, `"section_feedback"`.

---

## Questions — universal rules

Every question, regardless of type, must have these three fields:

| Field | Type | Rule |
|---|---|---|
| `id` | string | Unique within the entire form. Use `q_` prefix + short snake_case name. Example: `"q_full_name"`, `"q_satisfaction"`. Never reuse an id. |
| `type` | string | One of the 13 types listed below. Exact spelling, lowercase. |
| `label` | string | The question text the respondent reads. Full sentence or clear phrase. |

Optional on every question:
- `"required": true` — if omitted, defaults to `false`. Use it for questions where a blank answer would make the results meaningless.
- `"description": "..."` — helper text shown in smaller type below the label. Use it to clarify ambiguous questions, give examples, or add instructions.

**Id naming convention:** always `q_` followed by a meaningful snake_case word. Never use `q1`, `q2`, etc. — descriptive ids make exported results readable without needing to look up the form definition.

---

## The 13 question types

---

### 1. `short_text`
Single-line text input. Use for: names, job titles, short free-text answers, anything under ~100 characters.

```json
{
  "id": "q_full_name",
  "type": "short_text",
  "label": "What is your full name?",
  "required": true,
  "placeholder": "First and last name",
  "validation": {
    "min_length": 2,
    "max_length": 100
  }
}
```

- `placeholder`: greyed-out hint text inside the input. Optional but recommended.
- `validation.min_length` / `validation.max_length`: both optional integers.

---

### 2. `long_text`
Multi-line textarea. Use for: open-ended feedback, descriptions, comments, anything needing more than one sentence.

```json
{
  "id": "q_feedback",
  "type": "long_text",
  "label": "Please describe your experience in detail.",
  "required": false,
  "placeholder": "Write as much or as little as you like...",
  "rows": 5,
  "validation": {
    "max_length": 2000
  }
}
```

- `rows`: number of visible lines (default 4). Use 3 for short prompts, 6–8 for lengthy responses.

---

### 3. `short_text` vs `long_text` — decision rule
If the expected answer fits on one line → `short_text`. If you expect sentences or paragraphs → `long_text`.

---

### 4. `single_choice`
Pick exactly one option. Renders as radio buttons by default, or a dropdown.

```json
{
  "id": "q_department",
  "type": "single_choice",
  "label": "Which department do you work in?",
  "required": true,
  "display": "radio",
  "options": [
    { "value": "engineering", "label": "Engineering" },
    { "value": "design", "label": "Design" },
    { "value": "sales", "label": "Sales" },
    { "value": "hr", "label": "Human Resources" },
    { "value": "other", "label": "Other" }
  ]
}
```

- `display`: `"radio"` (default, all options visible) or `"dropdown"` (collapsed select menu).
- Use `"radio"` when there are ≤6 options. Use `"dropdown"` for 7+ options or long option labels.
- `options[].value`: machine key stored in results. Lowercase, underscores, no spaces. Must be unique within the options array.
- `options[].label`: human-readable text shown to the respondent.
- `"other_option": true` — adds a free-text "Other" field. Omit or set `false` if not needed.

---

### 5. `multiple_choice`
Pick one or more options. Renders as checkboxes.

```json
{
  "id": "q_interests",
  "type": "multiple_choice",
  "label": "Which topics interest you? (Select all that apply)",
  "required": false,
  "options": [
    { "value": "leadership", "label": "Leadership" },
    { "value": "technical", "label": "Technical skills" },
    { "value": "communication", "label": "Communication" },
    { "value": "strategy", "label": "Strategy" }
  ],
  "other_option": true
}
```

- Same `options` structure as `single_choice`.
- `"other_option": true` adds a checkbox labelled "Other" with a free-text field that activates when checked. The value saved in results is `"other:whatever they typed"`.
- `min_selections` / `max_selections`: optional integers to enforce selection count.
- Results are stored as an array: `["leadership", "communication", "other:Mentoring"]`.

---

### 6. `dropdown`
A plain dropdown select. Functionally identical to `single_choice` with `display: "dropdown"`. Use this type directly when you want a dropdown and don't need the radio fallback.

```json
{
  "id": "q_country",
  "type": "dropdown",
  "label": "Select your country",
  "required": true,
  "options": [
    { "value": "pl", "label": "Poland" },
    { "value": "de", "label": "Germany" },
    { "value": "fr", "label": "France" },
    { "value": "gb", "label": "United Kingdom" }
  ]
}
```

---

### 7. `rating`
Star or number rating. Best for satisfaction, quality, or experience questions with a clear low-to-high scale.

```json
{
  "id": "q_satisfaction",
  "type": "rating",
  "label": "How satisfied are you with the service?",
  "required": true,
  "min": 1,
  "max": 5,
  "style": "stars",
  "low_label": "Very dissatisfied",
  "high_label": "Very satisfied"
}
```

- `style`: `"stars"` (clickable star icons, best for 1–5 or 1–10) or `"numbers"` (numbered buttons).
- `min` / `max`: integers. Typical ranges: 1–5 for satisfaction, 1–10 for importance.
- `low_label` / `high_label`: anchor labels shown below the rating. Optional but strongly recommended — they disambiguate which end is good.
- Result stored as a number: `4`.

---

### 8. `scale`
A linear numeric scale rendered as a row of clickable buttons. Ideal for NPS (0–10), agreement scales (1–7), or any continuous range.

```json
{
  "id": "q_nps",
  "type": "scale",
  "label": "How likely are you to recommend us to a friend or colleague?",
  "required": true,
  "min": 0,
  "max": 10,
  "step": 1,
  "low_label": "Not at all likely",
  "high_label": "Extremely likely"
}
```

- `min` / `max` / `step`: all integers. `step` defaults to 1.
- Difference from `rating`: `scale` is for larger ranges (0–10, 1–7) where stars would be visually awkward. `rating` is for small ranges (1–5) where stars communicate quality.
- Result stored as a number: `8`.

---

### 9. `matrix`
A grid where rows are sub-questions and columns are answer options. Each row gets one (or multiple) selections. Best for rating several items on the same scale, or evaluating multiple aspects of one thing.

```json
{
  "id": "q_work_aspects",
  "type": "matrix",
  "label": "Rate the following aspects of your work environment:",
  "required": false,
  "multiple_per_row": false,
  "rows": [
    { "id": "compensation", "label": "Compensation & benefits" },
    { "id": "worklife",     "label": "Work-life balance" },
    { "id": "growth",       "label": "Career growth opportunities" },
    { "id": "culture",      "label": "Company culture" }
  ],
  "columns": [
    { "value": "1", "label": "Poor" },
    { "value": "2", "label": "Fair" },
    { "value": "3", "label": "Good" },
    { "value": "4", "label": "Excellent" }
  ]
}
```

- `rows[].id`: snake_case key, unique within the matrix. Used as sub-key in results.
- `multiple_per_row`: `false` = one radio per row (most common). `true` = checkboxes per row.
- Keep columns to 4–6 items maximum — more than that breaks the table layout on small screens.
- Result stored as an object: `{ "compensation": "3", "worklife": "4", "growth": "2", "culture": "3" }`.
- In XLSX export, each row becomes its own column: `"Rate the following... — Work-life balance"`.

---

### 10. `number`
Numeric input with optional min/max/step constraints.

```json
{
  "id": "q_employee_count",
  "type": "number",
  "label": "How many people are in your team?",
  "required": false,
  "min": 1,
  "max": 10000,
  "step": 1,
  "placeholder": "e.g. 12"
}
```

- Use for counts, ages, quantities, percentages, or any number where freeform text would be wrong.
- `step` can be a decimal for non-integer values: `0.1` for prices, `0.5` for half-steps.

---

### 11. `email`
Email input with built-in format validation.

```json
{
  "id": "q_email",
  "type": "email",
  "label": "Your email address",
  "required": false,
  "placeholder": "you@example.com"
}
```

- The browser validates email format automatically. Don't use `short_text` for email fields.

---

### 12. `date`
Date picker.

```json
{
  "id": "q_start_date",
  "type": "date",
  "label": "When did you start working here?",
  "required": false,
  "min_date": "2000-01-01",
  "max_date": "today"
}
```

- `min_date` / `max_date`: ISO format `"YYYY-MM-DD"`, or the special string `"today"` which resolves to the current date at render time.
- Result stored as `"YYYY-MM-DD"` string: `"2019-03-15"`.

---

### 13. `time`
Time picker.

```json
{
  "id": "q_preferred_time",
  "type": "time",
  "label": "What time of day do you prefer for meetings?",
  "required": false
}
```

- Result stored as `"HH:MM"` string: `"14:30"`.

---

### 14. `divider`
Not a question — a visual break with a heading and optional description. No response is collected. Use it to separate major topic groups within a single section without creating a new section.

```json
{
  "id": "div_background",
  "type": "divider",
  "label": "Professional Background",
  "description": "Tell us about your career so far."
}
```

- `id` must still be unique across the form (use `div_` prefix).
- `label` is required (becomes the heading). `description` is optional.

---

## Question type reference card

| Type | Use when | Result type |
|---|---|---|
| `short_text` | Short free text, names, titles | `string` |
| `long_text` | Multi-sentence open answers | `string` |
| `single_choice` | Pick one from a list (radio or dropdown) | `string` (the value) |
| `multiple_choice` | Pick any number from a list | `string[]` |
| `dropdown` | Pick one from a long list | `string` (the value) |
| `rating` | 1–5 stars or 1–10 numeric quality | `number` |
| `scale` | 0–10 NPS, 1–7 agreement | `number` |
| `matrix` | Rate multiple items on same scale | `object` (row ids → values) |
| `number` | Counts, ages, quantities | `number` |
| `email` | Email address | `string` |
| `date` | Calendar date | `string` `YYYY-MM-DD` |
| `time` | Time of day | `string` `HH:MM` |
| `divider` | Visual heading, no answer | *(nothing)* |

---

## Common mistakes to avoid

**Duplicate ids.** Every `id` — across all sections, all questions, all dividers — must be unique within the form. If you have `q_name` in section 1, you cannot have `q_name` in section 2.

**Non-snake_case ids.** Ids must be `lowercase_with_underscores`. No spaces, no hyphens, no camelCase. Wrong: `"Full Name"`, `"q-name"`, `"qName"`. Right: `"q_full_name"`.

**Missing `options` on choice types.** `single_choice`, `multiple_choice`, and `dropdown` all require a non-empty `options` array. Forgetting it will render a blank card.

**Missing `rows` or `columns` on matrix.** Both are required. A matrix with no rows or no columns is invalid.

**Using `short_text` for email.** Always use `type: "email"` for email fields. It adds format validation.

**Too many scale buttons.** A `scale` with `min: 0` and `max: 100` would render 101 buttons and break the layout. For large numeric ranges, use `type: "number"` instead.

**Empty `label`.** Every question (including dividers) must have a non-empty `label`. It's what the respondent reads.

**Options values with spaces.** Option `value` fields must be machine-safe: `"full_time"` not `"full time"`. Labels can be anything: `"label": "Full time"`.

---

## Choosing the right structure

**When to use one section:** the form is short (≤8 questions) or all questions belong to the same topic. Simple is better.

**When to split into sections:** the form covers distinct topics (demographics → experience → feedback), or has more than 8–10 questions. Each section should have a clear, distinct theme so the section title makes sense to the respondent.

**When to use a `divider` instead of a new section:** you want a visual heading within a section but the questions are closely related and don't warrant a full page break.

**How to pick between `rating` and `scale`:**
- `rating` with `style: "stars"` → satisfaction, quality, experience (1–5 range, star metaphor fits)
- `rating` with `style: "numbers"` → importance or priority (1–5 or 1–10, no star metaphor needed)
- `scale` → NPS (0–10), agreement (1–7), any range where the number itself communicates meaning

**How to pick between `single_choice` and `dropdown`:**
- ≤6 short options → `single_choice` with `display: "radio"` (all visible, faster to answer)
- 7+ options, or options with long labels → `dropdown` or `single_choice` with `display: "dropdown"`

---

## A complete minimal example

```json
{
  "form": {
    "id": "event_feedback",
    "title": "Event Feedback Form",
    "description": "Share your thoughts on today's event. Takes about 2 minutes.",
    "version": "1.0",
    "settings": {
      "allow_multiple_submissions": true,
      "show_progress_bar": false,
      "confirmation_message": "Thanks for your feedback — it really helps us improve!"
    },
    "sections": [
      {
        "id": "section_main",
        "title": "",
        "questions": [
          {
            "id": "q_overall",
            "type": "rating",
            "label": "How would you rate the event overall?",
            "required": true,
            "min": 1,
            "max": 5,
            "style": "stars",
            "low_label": "Poor",
            "high_label": "Excellent"
          },
          {
            "id": "q_highlight",
            "type": "short_text",
            "label": "What was the highlight of the event for you?",
            "required": false,
            "placeholder": "e.g. the keynote, the networking session..."
          },
          {
            "id": "q_improve",
            "type": "long_text",
            "label": "What could we do better next time?",
            "required": false,
            "rows": 4
          },
          {
            "id": "q_attend_again",
            "type": "single_choice",
            "label": "Would you attend this event again?",
            "required": true,
            "display": "radio",
            "options": [
              { "value": "definitely", "label": "Definitely yes" },
              { "value": "probably",   "label": "Probably yes" },
              { "value": "unsure",     "label": "Not sure" },
              { "value": "probably_not", "label": "Probably not" },
              { "value": "no",         "label": "No" }
            ]
          },
          {
            "id": "q_email",
            "type": "email",
            "label": "Your email (optional — only if you'd like us to follow up)",
            "required": false,
            "placeholder": "you@example.com"
          }
        ]
      }
    ]
  }
}
```

---

## Prompt template for users

If a user wants to generate a form, they should give you this information:

- **Purpose:** what the form is for and who will fill it in
- **Topics:** what areas or themes the form should cover
- **Length:** rough number of questions or how long it should take
- **Question types:** any preferences (e.g. "mostly multiple choice", "include a rating")
- **Required fields:** which questions must be answered
- **Sections:** whether to group questions into named sections

If any of this is missing, make reasonable assumptions based on the form's purpose and proceed. Do not ask for clarification on every missing detail — a well-structured draft is more useful than an incomplete one waiting for approval.

---

*FormRunner LLM Guide v1.0 — keep this file alongside your formrunner.html*
