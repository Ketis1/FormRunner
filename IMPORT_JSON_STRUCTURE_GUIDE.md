# FormRunner Import JSON Structure Guide (for LLMs)

This guide describes the JSON format that `formrunner.html` accepts when importing a form.
It combines the intended schema from `FORMRUNNER_SPEC.md` with the actual parser/runtime behavior in `formrunner.html`.

## 1) Minimum valid import JSON

Your JSON **must** include:

- top-level key: `form`
- `form.title` (non-empty string)
- `form.sections` (non-empty array)

Minimal valid object:

```json
{
  "form": {
    "title": "My Form",
    "sections": [
      {
        "questions": [
          {
            "id": "q1",
            "type": "short_text",
            "label": "Your answer"
          }
        ]
      }
    ]
  }
}
```

## 2) Canonical top-level shape

```json
{
  "form": {
    "id": "string (recommended)",
    "title": "string (required by parser)",
    "description": "string (optional)",
    "version": "string (optional)",
    "settings": {
      "allow_multiple_submissions": "boolean (optional, currently not enforced)",
      "show_progress_bar": "boolean (optional, default true)",
      "shuffle_sections": "boolean (optional, currently not used)",
      "confirmation_message": "string (optional)"
    },
    "sections": [
      {
        "id": "string (recommended)",
        "title": "string (optional)",
        "description": "string (optional)",
        "shuffle_questions": "boolean (optional, currently not used)",
        "questions": []
      }
    ]
  }
}
```

## 3) Question base fields

For normal questions (all except `divider`), always provide:

- `id` (string, unique across form)
- `type` (supported type string)
- `label` (string shown to user)

Optional common fields:

- `description` (string)
- `required` (boolean, default false)
- `placeholder` (string, used by text-like inputs)

## 4) Supported `type` values and required per-type fields

### 4.1 `short_text`
- Optional: `placeholder`, `validation.max_length`

### 4.2 `email`
- Optional: `placeholder`, `validation.max_length`

### 4.3 `long_text`
- Optional: `placeholder`, `rows`, `validation.max_length`

### 4.4 `number`
- Optional: `min`, `max`, `step`, `placeholder`

### 4.5 `date`
- Optional: `min_date`, `max_date`
- `max_date: "today"` is supported.

### 4.6 `time`
- No special required fields.

### 4.7 `single_choice`
- Required: `options` (array of `{ "value": "...", "label": "..." }`)
- Optional: `display` (`"radio"` default, `"dropdown"` supported)

### 4.8 `dropdown`
- Required: `options` (same structure as `single_choice`)

### 4.9 `multiple_choice`
- Required: `options` (array of `{ "value": "...", "label": "..." }`)
- Optional: `other_option` (boolean)

### 4.10 `rating`
- Optional: `min` (default 1), `max` (default 5), `style` (`"stars"` default, `"numbers"`), `low_label`, `high_label`

### 4.11 `scale`
- Optional: `min` (default 0), `max` (default 10), `step` (default 1), `low_label`, `high_label`

### 4.12 `matrix`
- Required:
  - `rows` array of `{ "id": "...", "label": "..." }`
  - `columns` array of `{ "value": "...", "label": "..." }`
- Optional: `multiple_per_row` (boolean; false = single-select per row, true = multi-select per row)

### 4.13 `divider`
- Visual separator, not answerable.
- Recommended: `id`, `label`, optional `description`.

## 5) Response data shape by question type (important for LLM planning)

When users submit responses, values are stored under each question `id` as:

- `short_text`, `email`, `long_text`, `date`, `time`, `single_choice`, `dropdown` -> string
- `number`, `rating`, `scale` -> number
- `multiple_choice` -> string array (if `other_option`, custom value becomes `"other:<text>"`)
- `matrix` -> object keyed by row id
  - single-per-row mode: `rowId -> "columnValue"`
  - multiple-per-row mode: `rowId -> ["columnValue1", "..."]`

## 6) LLM generation rules (strict)

1. Output valid JSON only (no comments, no trailing commas).
2. Always include `form.title` and at least one section with at least one question.
3. For every non-divider question, include `id`, `type`, and `label`.
4. Keep all `id` values unique (sections and questions).
5. For choice-based questions (`single_choice`, `dropdown`, `multiple_choice`, `matrix`), always include non-empty option/row/column arrays.
6. Keep `options[].value` stable, machine-friendly tokens (e.g., `very_satisfied`), and `options[].label` human-readable text.
7. Use supported `type` values only.
8. Prefer explicit booleans for `required`.

## 7) Recommended starter template for LLM output

```json
{
  "form": {
    "id": "customer_feedback_001",
    "title": "Customer Feedback Form",
    "description": "Please answer all required questions.",
    "version": "1.0",
    "settings": {
      "show_progress_bar": true,
      "confirmation_message": "Thank you. Your response has been recorded."
    },
    "sections": [
      {
        "id": "section_1",
        "title": "Basics",
        "questions": [
          {
            "id": "q_name",
            "type": "short_text",
            "label": "Your name",
            "required": true
          },
          {
            "id": "q_experience",
            "type": "rating",
            "label": "Rate your experience",
            "required": true,
            "min": 1,
            "max": 5,
            "style": "stars"
          }
        ]
      }
    ]
  }
}
```

## 8) Practical compatibility notes

- Import parser currently validates only a few top-level requirements (`form`, `form.title`, non-empty `form.sections`), so malformed internals may still parse but render poorly.
- Prefer the stricter structure in this guide to ensure reliable rendering and export behavior.
