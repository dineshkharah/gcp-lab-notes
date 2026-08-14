# Machine Learning Operations (MLOps) with Agent Platform: Model Evaluation

A course rather than a lab. No console work, no Cloud Shell, no checkpoints. Two required quizzes and a survey. Stated at two and a half hours; the quizzes themselves are about ten minutes.

Both quizzes scored **100%**.

Course last updated about one month before this run.

## The two quizzes

| Quiz | Questions | Pass mark | Margin |
|---|---|---|---|
| MLOps: Introduction to Model Evaluation | 4 | 75% | 3 of 4 |
| MLOps: Model Evaluation for Generative AI | 7 | 80% | 6 of 7 |

The second one is the tighter of the two. 80% over 7 questions means one wrong answer passes and two does not.

## Quiz 1, Introduction to Model Evaluation

**1. Model performs excellently on training data and poorly on new data.** → **Overfitting.**

Generalization is the property the model *lacks*, not the name of the problem. Bias variance tradeoff is the underlying tension rather than the symptom. Data shift would mean the new data's distribution changed, not that the model memorised.

**2. Team struggling with inconsistent performance, version tracking and collaboration.** → **"MLOps with Vertex AI would provide a structured framework for managing the entire ML lifecycle, promoting collaboration, enabling version control, and improving model performance consistency."**

The other three options are all "it only does one thing" statements, and each explicitly names one of the team's stated problems as something MLOps would *not* solve. When a question lists three symptoms, the answer addresses all three.

**3. Wants the model to adapt to changes in real world data over time.** → **Continuous evaluation**, monitoring on new data after deployment and retraining as needed.

The hinge is "over time". Holdout validation, cross validation and leave one out cross validation all only ever see data you already had at training time.

**4. Which scenarios benefit from Vertex AI's model evaluation service? Select all that apply.** → **three of four**:

- large dataset, compare multiple model versions ✓
- detailed feedback **from users** about quality and relevance ✗
- concerned about overfitting, assess on unseen data ✓
- monitor deployed performance over time, detect concept drift ✓

The user feedback option is the distractor: the service produces metrics, not human opinions. Note that drift detection is strictly Vertex AI Model Monitoring rather than the evaluation service, but this course folds continuous evaluation into evaluation, and question 3 makes that framing explicit.

## Quiz 2, Model Evaluation for Generative AI

**1. Compare multiple model versions summarising news articles, want the most informative and coherent summaries.** → **Ranking evaluation** with human evaluators.

"Informative and coherent" are precisely the qualities ROUGE and BLEU cannot measure, since those score n gram overlap against a reference.

**2. Marketing copy that is both creative and relevant.** → **Combining automated metrics for diversity and relevance with human evaluation for creativity and brand alignment.**

The other three all contain "only" or "solely".

**3. Which LLM block component stores and retrieves past interactions?** → **Memory.**

Data sources feed retrieval, prompt templates shape the request, guardrails filter it.

**4. Responses are fluent and grammatical but factually incorrect.** → **Model complexity and decision-making.**

Hallucination, and the *evaluation* challenge it illustrates is that the output's surface form tells you nothing about whether the reasoning behind it was sound. The two "data" options belong to question 6's topic; data contamination is train and test leakage.

**5. Evaluating a model that generates creative stories. Select all that apply.** → **two of four**:

- grammar and syntax checker ✗
- **perplexity** ✓
- **diversity and originality** ✓
- BLEU against reference stories ✗

**Perplexity counts, and it is the one worth remembering**, because there is a real argument against it: low perplexity means predictable text, which is arguably the opposite of creative. The course teaches computation based metrics as part of generative evaluation, and that framing wins. BLEU is out for the opposite reason, that creative writing has no single correct answer and n gram overlap penalises originality.

**6. BLEU or ROUGE with a limited or biased reference dataset.** → **"The evaluation may underestimate the model's true capabilities because the reference data doesn't cover the full range of acceptable responses."**

A good answer phrased differently from the one reference scores badly. **The "overestimate" option is the trap**: a narrow reference set makes a model look worse, not better.

**7. Pick the best of three image classification models, understand real world performance, identify areas for improvement.** → **Pointwise evaluation**, absolute performance per model with strengths and weaknesses.

Worth reading against question 1, which also compares several models and wants **ranking** instead. The difference is the goal: question 1 only needs a winner on subjective quality, question 7 says "identify areas for potential improvement", which needs each model assessed on its own terms.

## The distractor patterns, which are the reusable part

Four shapes recur across both quizzes and are worth more than the answers themselves.

- **"Only" and "solely" mark a wrong answer.** Every question whose correct answer is "combine automated metrics with human evaluation" surrounds it with single method options.
- **An option that denies one of the question's own stated problems is wrong.** Quiz 1 question 2 lists three symptoms and three of its options each say MLOps would not fix one of them.
- **Underestimate versus overestimate is a real distinction, not a coin flip.** A narrow reference set penalises correct answers that differ from the reference, so it drags scores down.
- **Ranking, pointwise, pairwise and binary are chosen by the stated goal, not by how many models are involved.** Want a winner, rank. Want to know why each one is good or bad, pointwise. Both quizzes test that distinction and quiz 2 tests it twice.

## Related files

- `gsp341-create-ml-models-bigquery-ml-challenge.md` and `gsp327-bigquery-ml-fare-prediction-challenge.md` for evaluation metrics in practice, including `ML.EVALUATE` returning `mean_squared_error` rather than RMSE.
- `gsp374-bigquery-soccer-bqml.md` for the same in a guided form.
