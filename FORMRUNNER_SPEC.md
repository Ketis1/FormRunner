# FormRunner — Local Google Forms Alternative
## Specification v1.0

---

## 1. System Requirements

### Runtime
- **No installation required** — single HTML file, runs in any modern browser (Chrome 90+, Firefox 88+, Edge 90+, Safari 14+)
- **No server needed** — open `formrunner.html` directly via `file://` protocol or serve with any static HTTP server
- **Optional static server** (for XLSX export): `npx serve .` or `python -m http.server`

### File export dependencies
- **JSON export**: built-in, zero dependencies
- **XLSX export**: loads SheetJS from CDN (`cdn.jsdelivr.net`) — requires internet on first use, or bundle manually

### Browser permissions
- `File` API for importing form definition
- `Blob` + `URL.createObjectURL` for downloading results
- Local storage (optional) for saving draft responses

---

## 2. Form Definition Format

**Chosen format: JSON**

Rationale: JSON is the best fit because:
- Native to JavaScript (zero parsing overhead)
- Strict schema validation is straightforward
- Supports nested structures (options lists, validation rules)
- Human-readable enough for manual editing
- Widely supported by all editors and tools

Alternatives considered:
- YAML: friendlier to write by hand, but requires a parser library
- TOML: good for config, poor for nested arrays of questions
- Markdown: great for display, unsuitable for structured metadata

---

## 3. Form Definition Schema

### Top-level structure

```json
{
  "form": {
    "id": "string (required) — unique identifier, used as filename prefix",
    "title": "string (required) — displayed at top of form",
    "description": "string (optional) — subtitle or instructions",
    "version": "string (optional, default '1.0')",
    "settings": { ... },
    "sections": [ ... ]
  }
}
```

### settings object

```json
{
  "settings": {
    "allow_multiple_submissions": false,
    "show_progress_bar": true,
    "shuffle_sections": false,
    "confirmation_message": "Thank you for completing the form."
  }
}
```

### sections array

Forms are organized into one or more sections. A single-section form is fine.

```json
{
  "sections": [
    {
      "id": "string (required)",
      "title": "string (optional)",
      "description": "string (optional)",
      "shuffle_questions": false,
      "questions": [ ... ]
    }
  ]
}
```

### questions array — common fields

Every question shares these base fields:

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | ✅ | Unique within form. Used as key in results output. |
| `type` | string | ✅ | Question type (see types below) |
| `label` | string | ✅ | The question text shown to respondent |
| `description` | string | ❌ | Helper text shown below the label |
| `required` | boolean | ❌ | Default: `false` |
| `placeholder` | string | ❌ | Input placeholder (text/textarea types only) |

---

## 4. Question Types

### `short_text` — Single-line open answer
```json
{
  "id": "q1",
  "type": "short_text",
  "label": "What is your full name?",
  "required": true,
  "placeholder": "First and last name",
  "validation": {
    "min_length": 2,
    "max_length": 100,
    "pattern": null
  }
}
```

### `long_text` — Multi-line open answer
```json
{
  "id": "q2",
  "type": "long_text",
  "label": "Describe your experience in detail.",
  "required": false,
  "placeholder": "Write here...",
  "rows": 5,
  "validation": {
    "max_length": 2000
  }
}
```

### `single_choice` — Pick one (radio buttons / dropdown)
```json
{
  "id": "q3",
  "type": "single_choice",
  "label": "What is your employment status?",
  "required": true,
  "display": "radio",
  "shuffle_options": false,
  "options": [
    { "value": "employed", "label": "Employed full-time" },
    { "value": "part_time", "label": "Employed part-time" },
    { "value": "self_employed", "label": "Self-employed" },
    { "value": "unemployed", "label": "Not currently employed" },
    { "value": "student", "label": "Student" }
  ],
  "other_option": false
}
```

`display` options: `"radio"` (default), `"dropdown"`

### `multiple_choice` — Pick many (checkboxes)
```json
{
  "id": "q4",
  "type": "multiple_choice",
  "label": "Which programming languages do you use? (Select all that apply)",
  "required": false,
  "shuffle_options": false,
  "min_selections": 1,
  "max_selections": null,
  "options": [
    { "value": "python", "label": "Python" },
    { "value": "javascript", "label": "JavaScript" },
    { "value": "typescript", "label": "TypeScript" },
    { "value": "rust", "label": "Rust" },
    { "value": "go", "label": "Go" }
  ],
  "other_option": true
}
```

### `rating` — Numeric star/dot rating
```json
{
  "id": "q5",
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

`style` options: `"stars"`, `"numbers"`

### `scale` — Linear scale (like NPS)
```json
{
  "id": "q6",
  "type": "scale",
  "label": "How likely are you to recommend us to a friend?",
  "required": true,
  "min": 0,
  "max": 10,
  "step": 1,
  "low_label": "Not at all likely",
  "high_label": "Extremely likely"
}
```

### `date` — Date picker
```json
{
  "id": "q7",
  "type": "date",
  "label": "What is your date of birth?",
  "required": false,
  "min_date": "1900-01-01",
  "max_date": "today"
}
```

### `time` — Time picker
```json
{
  "id": "q8",
  "type": "time",
  "label": "What time do you prefer for the meeting?",
  "required": false,
  "format": "24h"
}
```

### `dropdown` — Select from list (alias for single_choice with display: dropdown)
```json
{
  "id": "q9",
  "type": "dropdown",
  "label": "Select your country",
  "required": true,
  "options": [
    { "value": "pl", "label": "Poland" },
    { "value": "de", "label": "Germany" },
    { "value": "fr", "label": "France" }
  ]
}
```

### `matrix` — Grid of radio/checkbox rows
```json
{
  "id": "q10",
  "type": "matrix",
  "label": "Rate the following aspects:",
  "required": false,
  "multiple_per_row": false,
  "rows": [
    { "id": "price", "label": "Price" },
    { "id": "quality", "label": "Quality" },
    { "id": "support", "label": "Support" }
  ],
  "columns": [
    { "value": "1", "label": "Poor" },
    { "value": "2", "label": "Fair" },
    { "value": "3", "label": "Good" },
    { "value": "4", "label": "Excellent" }
  ]
}
```

### `number` — Numeric input
```json
{
  "id": "q11",
  "type": "number",
  "label": "How many employees does your company have?",
  "required": false,
  "min": 1,
  "max": 1000000,
  "step": 1,
  "placeholder": "e.g. 50"
}
```

### `email` — Email address input (with validation)
```json
{
  "id": "q12",
  "type": "email",
  "label": "Your email address",
  "required": true,
  "placeholder": "you@example.com"
}
```

### `divider` — Visual section break (not a question, no response collected)
```json
{
  "id": "div1",
  "type": "divider",
  "label": "Personal Information",
  "description": "Please fill in your details below."
}
```

---

## 5. Results Output Format

### JSON output

```json
{
  "meta": {
    "form_id": "employee_survey_2024",
    "form_title": "Employee Satisfaction Survey",
    "form_version": "1.0",
    "exported_at": "2024-11-15T14:32:00.000Z",
    "total_responses": 1
  },
  "responses": [
    {
      "_id": "resp_1731678720000",
      "_submitted_at": "2024-11-15T14:32:00.000Z",
      "q1": "Jane Smith",
      "q2": "The onboarding was smooth but could use better documentation.",
      "q3": "employed",
      "q4": ["python", "typescript", "other:Kotlin"],
      "q5": 4,
      "q6": 8,
      "q7": "1990-04-22",
      "q8": "09:30",
      "q10": {
        "price": "3",
        "quality": "4",
        "support": "2"
      },
      "q11": 250,
      "q12": "jane.smith@company.com"
    }
  ]
}
```

### XLSX output

One sheet named `Responses`. Row 1 = headers (question labels). Row 2+ = one row per response. Matrix questions expand to one column per sub-row, labeled `Question Label — Sub-row Label`.

---

## 6. Full Example Form File

Save as `my_form.json` and import into FormRunner:

```json
{
  "form": {
    "id": "employee_survey_2024",
    "title": "Employee Satisfaction Survey",
    "description": "This survey takes approximately 5 minutes. All responses are anonymous.",
    "version": "1.0",
    "settings": {
      "allow_multiple_submissions": true,
      "show_progress_bar": true,
      "confirmation_message": "Thank you for your feedback! Your response has been recorded."
    },
    "sections": [
      {
        "id": "section_personal",
        "title": "About You",
        "questions": [
          {
            "id": "q_name",
            "type": "short_text",
            "label": "Your full name",
            "required": true,
            "placeholder": "First and last name"
          },
          {
            "id": "q_dept",
            "type": "single_choice",
            "label": "Which department are you in?",
            "required": true,
            "display": "dropdown",
            "options": [
              { "value": "eng", "label": "Engineering" },
              { "value": "design", "label": "Design" },
              { "value": "sales", "label": "Sales" },
              { "value": "hr", "label": "Human Resources" },
              { "value": "other", "label": "Other" }
            ]
          }
        ]
      },
      {
        "id": "section_satisfaction",
        "title": "Your Experience",
        "questions": [
          {
            "id": "q_satisfaction",
            "type": "rating",
            "label": "Overall, how satisfied are you with your role?",
            "required": true,
            "min": 1,
            "max": 5,
            "style": "stars",
            "low_label": "Very dissatisfied",
            "high_label": "Very satisfied"
          },
          {
            "id": "q_recommend",
            "type": "scale",
            "label": "How likely are you to recommend this company as a great place to work?",
            "required": true,
            "min": 0,
            "max": 10,
            "low_label": "Not likely",
            "high_label": "Highly likely"
          },
          {
            "id": "q_aspects",
            "type": "matrix",
            "label": "Rate the following aspects of your work environment:",
            "required": false,
            "multiple_per_row": false,
            "rows": [
              { "id": "compensation", "label": "Compensation & benefits" },
              { "id": "worklife", "label": "Work-life balance" },
              { "id": "growth", "label": "Career growth opportunities" },
              { "id": "culture", "label": "Company culture" }
            ],
            "columns": [
              { "value": "1", "label": "Poor" },
              { "value": "2", "label": "Fair" },
              { "value": "3", "label": "Good" },
              { "value": "4", "label": "Excellent" }
            ]
          }
        ]
      },
      {
        "id": "section_feedback",
        "title": "Open Feedback",
        "questions": [
          {
            "id": "q_best",
            "type": "long_text",
            "label": "What do you like most about working here?",
            "required": false,
            "rows": 4
          },
          {
            "id": "q_improve",
            "type": "long_text",
            "label": "What is one thing you would improve?",
            "required": false,
            "rows": 4
          },
          {
            "id": "q_skills",
            "type": "multiple_choice",
            "label": "Which skills would you like more training in?",
            "required": false,
            "options": [
              { "value": "leadership", "label": "Leadership" },
              { "value": "technical", "label": "Technical skills" },
              { "value": "communication", "label": "Communication" },
              { "value": "project_mgmt", "label": "Project management" },
              { "value": "data", "label": "Data & analytics" }
            ],
            "other_option": true
          }
        ]
      }
    ]
  }
}
```

---

## 7. File Naming Conventions

| File | Purpose |
|---|---|
| `formrunner.html` | The app (single file, open in browser) |
| `my_form.json` | Form definition (you create this) |
| `{form_id}_results_{timestamp}.json` | Exported results (JSON) |
| `{form_id}_results_{timestamp}.xlsx` | Exported results (XLSX) |

---

*FormRunner Spec v1.0 — designed to run entirely in the browser with zero dependencies for core functionality.*
