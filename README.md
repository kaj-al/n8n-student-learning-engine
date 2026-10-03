# Study Resources Personalization

This project is an n8n workflow that personalizes a student's 3-day study plan based on their assessment results. It reads student performance data from Google Sheets, identifies weak topics, uses a Groq-powered LLM to generate a tailored study schedule, matches each topic to a valid learning resource, stores the recommendations, and emails the student the final plan. Useful for coaching centres and educational platforms.

## Overview

The workflow is designed to:

- Read student records from Google Sheets
- Calculate average score and identify weak topics
- Check whether the student has enough quiz attempts for a recommendation
- Generate a 3-day learning plan using an LLM
- Validate the output structure and retry when necessary
- Map generated topic recommendations to available learning resources
- Save the final recommendations to a Google Sheet
- Send the final plan to the student via Gmail

## Workflow behavior

The workflow in `Study.json` runs on a schedule trigger and follows this flow:

1. Fetch student rows from the `Students` sheet.
2. Loop through each student record.
3. Calculate:
   - average quiz score
   - total quiz attempts
   - weak topics below a 60 threshold
4. If enough data is available, send the student's data to a Groq LLM.
5. The model is instructed to return structured JSON with three entries:
   - `day1`
   - `day2`
   - `day3`
6. Each day entry must include:
   - `topic`
   - `resource_id`
   - `reason`
7. The workflow validates the response and retries up to a limit if the output is invalid.
8. If the plan is valid, the workflow retrieves matching content from the learning content sheet using `resource_id` values.
9. It builds a final plan with direct resource links.
10. It writes the final recommendation to the `Recommendations` sheet.
11. It emails the student with the three-day plan.

## Supported resource IDs

The LLM is explicitly constrained to use these resource IDs only:

- `RES001` — algebra
- `RES002` — geometry
- `RES003` — trigonometry

This prevents invalid or fabricated resource recommendations.

## Files

- `Study.json` — the exported n8n workflow
- `README.md` — project documentation

## Required tools and services

To run this workflow, you need:

- n8n instance
- Google Sheets connection with access to the student and content sheets
- Gmail OAuth connection
- Groq API credentials
- A configured LLM model such as `openai/gpt-oss-120b`

## Google Sheets structure

### Students sheet
The workflow expects each row to include fields such as:

- `student_id`
- `email`
- `quiz_scores`
- `topic_wise_scores`

The `quiz_scores` field should contain a JSON array of numeric scores, and `topic_wise_scores` should contain a JSON object keyed by subject/topic names.

### Content library sheet
The learning content sheet should contain records with at least:

- `resource_id`
- `topic`
- `link`

The workflow uses `resource_id` to join the generated recommendations with the actual learning materials.

### Recommendations sheet
The workflow appends rows with columns such as:

- `student_id`
- `date`
- `day1_topic`
- `day1_link`
- `day2_topic`
- `day2_link`
- `day3_topic`
- `day3_link`
- `email`

## Example logic

The workflow calculates weak topics with logic similar to:

```javascript
const topicScores = JSON.parse(data.topic_wise_scores || '{}');
const weakTopics = Object.entries(topicScores)
  .filter(([topic, score]) => score < 60)
  .map(([topic]) => topic);
```

It also calculates average score using:

```javascript
const scores = JSON.parse(student.quiz_scores || '[]');
const avgScore = scores.length ? scores.reduce((a,b)=>a+b,0)/scores.length : 0;
```

## How the study plan is generated

The LLM prompt includes the student's weak topics and average score, for example:

```text
Weak topics: algebra, geometry. Average score: 52.
```

The output is required to be valid JSON shaped like:

```json
{
  "day1": {
    "topic": "",
    "resource_id": "",
    "reason": ""
  },
  "day2": {
    "topic": "",
    "resource_id": "",
    "reason": ""
  },
  "day3": {
    "topic": "",
    "resource_id": "",
    "reason": ""
  }
}
```

## Fallback behavior

If the student has insufficient data, or if the LLM returns invalid data, the workflow uses a diagnostic fallback plan:

- Day 1: Diagnostic Quiz - Core Concepts
- Day 2: Diagnostic Quiz - Applied
- Day 3: Diagnostic Quiz - Mixed Review

This ensures the system still produces a usable recommendation flow.

## Setup steps

1. Import `Study.json` into n8n.
2. Add your Google Sheets credentials.
3. Connect the `Students` and `ContentLibrary` sheets to the workflow.
4. Set up the Groq model credentials and ensure the model is available.
5. Add Gmail credentials for sending personalized emails.
6. Adjust the schedule trigger to run when needed.
7. Ensure your Sheets contain the exact columns expected by the workflow.


