# study-kit

A small, dependency-free TypeScript engine for study apps. You hand it a description of a subject — courses, revision cards, questions, terms and assessment dates — and it hands back the behaviour every study app needs:

- a **grounded tutor** that answers only from your course material and cites the section it used
- a **study lab** that builds guides, practice papers and matching games
- **answer marking** with keyword rubrics and typo tolerance
- **spaced review** and a mastery ladder
- **progress maths** for activity and assessment readiness
- **deadline maths** and a calendar export

It contains no subject content and no UI. Swap algebra for anatomy or law by passing a different bundle.

```ts
import { createSubjectKit } from "study-kit";

const kit = createSubjectKit(algebra);
kit.answer(1, "What does slope measure?");
```

## Install

From GitHub:

```bash
npm install github:Akira-Val-297/study-kit
# or
bun add github:Akira-Val-297/study-kit
```

From a clone or a local folder:

```bash
git clone https://github.com/Akira-Val-297/study-kit.git
npm install ./study-kit
```

From npm, once published (`npm publish` in this repo):

```bash
npm install study-kit
```

The package ships ESM with type declarations, has **no runtime dependencies**, and works in browsers, Node 18+, Bun, Deno and workers. There is no build step on your side.

## Quick start

```ts
import { createSubjectKit, type Subject } from "study-kit";

const subject: Subject = {
  name: "Algebra Refresher",
  courses: [
    {
      id: 1,
      code: "ALG-101",
      title: "Linear equations",
      subtitle: "Lines and their graphs",
      description: "Straight lines, slope, and where they cross the axes.",
      objectives: ["Read a slope off a graph", "Find the y-intercept"],
      exitCriteria: ["Explain what slope measures"],
      sections: [
        { title: "Slope", content: "The slope of a line measures how steep it is. It is rise over run." },
        { title: "Intercept", content: "The y-intercept is where the line crosses the vertical axis." },
      ],
    },
  ],
  cards: [{ id: "c1", course: 1, question: "Slope is rise over _____.", answer: "run" }],
  questions: [
    {
      id: "q1",
      course: 1,
      question: "What gives the slope of a line?",
      options: ["Rise over run", "Run over rise", "The intercept"],
      correctAnswer: 0,
      explanation: "Slope is the vertical change divided by the horizontal change.",
    },
  ],
  terms: [{ id: 1, startWeek: 1, startDate: "2026-09-07" }],
  milestones: [{ id: "m1", code: "MS-1", title: "Quiz 1", termId: 1, week: 4, weightPercent: 20 }],
};

const kit = createSubjectKit(subject);

kit.answer(1, "Explain slope like I'm 12");
// { intent: "simplify", title: "…", body: [...], bullets: [...],
//   citations: [{ courseId: 1, courseTitle: "Linear equations", section: "Slope" }],
//   followUps: [...] }

kit.quiz(1, 5);                 // PracticeQuestion[]
kit.studyGuide(1);              // StudyGuide
kit.practiceTest(1, null, 8);   // PracticeQuestion[]
kit.projectedTermScore({ m1: 72 }, 1);
// { recordedWeight: 20, totalWeight: 20, score: 72, passing: true, complete: true }

kit.nextMilestone(["m1"]);
// { milestone: {...}, date: Date, days: 9 }

kit.milestonesToIcs();          // "BEGIN:VCALENDAR…" string
```

Every function is also exported standalone (`answerTutor`, `quizQuestions`, `gradeShortAnswer`, …) when you would rather pass the subject in yourself.

## The bundle

All types are exported. Only `name` and `courses` are required.

| Type | Fields |
| --- | --- |
| `Subject` | `name`, `courses`, `cards?`, `questions?`, `terms?`, `milestones?` |
| `Course` | `id`, `code`, `title`, `subtitle`, `description`, `objectives[]`, `exitCriteria[]`, `sections[]` |
| `Section` | `title`, `content`, `type?: "text" \| "list"` — a `list` section is rendered as ordered points |
| `Card` | `id`, `course`, `question` (use `_____` for a cloze), `answer` |
| `ChoiceQuestion` | `id`, `course`, `question`, `options[]`, `correctAnswer` (index), `explanation` |
| `Term` | `id`, `startWeek`, `startDate` (`YYYY-MM-DD`) |
| `MilestoneSpec` | `id`, `code?`, `title`, `termId`, `week`, `weightPercent`, `qualifying?` (default 40), `coversCourses?` |
| `Material` | `title`, `summary[]`, `keywords[]`, `cards[]`, `questions[]` — text a learner pasted or uploaded |

`Material` is how "study my own notes" works: pass it to `studyGuide`, `practiceTest` or `matchPairs` and the output mixes your material with the course content, labelling which is which.

## API

### Tutor

| Function | Returns |
| --- | --- |
| `detectIntent(question)` | one of `explain`, `simplify`, `summarise`, `keypoints`, `compare`, `steps`, `example`, `exam`, `quiz`, `help` |
| `rankSections(subject, courseId, question)` | `SectionMatch[]` sorted by relevance |
| `searchAllSections(subject, question, limit)` | matches across **every** course, best first |
| `answerTutor(subject, courseId, question)` | `TutorReply` — `intent`, `title`, `body[]`, `bullets[]`, `citations[]`, `followUps[]`, `practice?` |

Answers are always built from `sections[].content`; nothing is invented. If the open course has nothing relevant, the tutor searches the other courses, answers from the best match and cites that other course instead of guessing. A `quiz` question returns practice questions in `reply.practice`.

### Practice and marking

| Function | Returns |
| --- | --- |
| `quizQuestions(subject, courseId, count)` | MCQs plus cloze blanks built from cards |
| `writtenQuestions(subject, courseId, count)` | short-answer prompts with keyword rubrics |
| `gradeShortAnswer(question, response)` | `{ score, matched[], missed[], feedback }` |
| `mentions(answer, keyword)` | keyword presence with one-edit tolerance, so `intercept` matches `intercepts` and `slop` matches `slope` |

Marking is coverage of the question's `keywords`, capped at 30 when the response is under five words, with feedback naming what is missing.

### Memory

| Function | Returns |
| --- | --- |
| `masteryLevel(correct, attempted)` | `{ level, index }` — `Unfamiliar` → `Learning` → `Familiar` → `Mastered` |
| `scheduleReview(rating, previous?, now?)` | `{ intervalDays, ease, dueAt }` from a small SM-2 style scheduler; `rating` is `again`, `good` or `easy` |

Mastered needs 80% accuracy over at least five attempts, so one lucky answer does not clear a concept.

### Study lab

| Function | Returns |
| --- | --- |
| `buildStudyGuide(subject, courseId, material?)` | `{ title, overview, sections[], keyTerms[], examTips[], source }` |
| `buildPracticeTest(subject, courseId, material?, count)` | interleaved course and material questions, de-duplicated, timed-test ready |
| `buildMatchPairs(subject, courseId, material?, count)` | `{ id, term, definition }[]` for a matching game |

### Progress

| Function | Returns |
| --- | --- |
| `activityPercent({ lessonDone, labsDone, labsTotal, projectsDone, projectsTotal })` | 0–100, weighting lesson 40 / labs 30 / projects 30 |
| `readinessPercent({ milestones, conceptPercent? })` | 0–100 or `null` — weighted average of recorded milestone scores, falling back to concept accuracy |
| `projectedTermScore(subject, scores, termId)` | `{ recordedWeight, totalWeight, score, passing, complete }` |

Absent components are renormalised rather than counted as zero, so a course with no labs is not penalised for having none. `activityPercent` ("what you have done") and `readinessPercent` ("how you are scoring") are deliberately separate: work done never inflates a mark.

### Deadlines

| Function | Returns |
| --- | --- |
| `milestoneDate(subject, milestone)` | `Date`, from the term's `startDate` plus `week − startWeek` weeks |
| `daysUntil(date, now?)` | whole days, rounded at local midnight |
| `nextMilestone(subject, cleared?, now?)` | the nearest uncleared milestone, `{ milestone, date, days }`, or `null` |
| `milestonesToIcs(subject, now?)` | a complete `.ics` string, one all-day event per milestone with weight and pass mark in the description |

### Bound kit

`createSubjectKit(subject)` returns every above, already bound:

```ts
kit.answer / kit.detectIntent / kit.search
kit.quiz / kit.written / kit.grade
kit.mastery / kit.studyGuide / kit.practiceTest / kit.matchPairs
kit.activityPercent / kit.readinessPercent / kit.projectedTermScore
kit.nextMilestone / kit.milestoneDate / kit.milestonesToIcs
kit.subject
```

## Wiring it into an app

The kit is pure: no storage, no network, no globals. Persistence is yours.

**React** — bind once per subject, keep learner state outside:

```tsx
const kit = useMemo(() => createSubjectKit(subject), []);
const reply = useMemo(() => kit.answer(courseId, question), [kit, courseId, question]);
```

**Storage** — the shapes are plain JSON, so `localStorage`, IndexedDB, SQLite or an API all work without adapters:

```ts
const scores = JSON.parse(localStorage.getItem("scores") ?? "{}");
const projection = kit.projectedTermScore(scores, termId);
```

**Server** — it runs anywhere, so the same rules can mark submissions on the server:

```ts
app.post("/api/mark", (req, res) => {
  const { questionId, response } = req.body;
  const question = kit.written(courseId, 50).find((q) => q.id === questionId);
  res.json(kit.grade(question, response));
});
```

## Design notes

- **Grounded, not generative.** No model is called. Answers are assembled from the text you supplied, which keeps them cheap, offline and citable.
- **Content lives in data.** Adding a subject means adding an object, never editing the engine.
- **Deterministic.** Same input, same output — safe to snapshot, diff and test.
- **Small surface.** One file, no dependencies, roughly 1,000 lines including its types.

## Development

```bash
npm install
npm test        # vitest
npm run build   # dist/index.js + dist/index.d.ts
npm run example # prints a full walkthrough for the algebra subject
```

The test suite uses a throwaway two-course subject rather than any real curriculum, so it fails if subject-specific assumptions creep into the engine.

## Contributing

Issues and PRs are welcome at <https://github.com/Akira-Val-297/study-kit>. Please keep the engine content-free: if a change needs to know anything about a particular subject, it belongs in the bundle, not in `src/index.ts`.

## Licence

MIT — see [LICENSE](./LICENSE).
