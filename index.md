---
title: What Makes a Recipe Highly Rated?
---

# What Makes a Recipe Highly Rated?

**Author:** Shahd

*A data science analysis of Food.com recipes and ratings, completed for DSC 80 at UC San Diego.*

---

## Introduction

People cook for all sorts of reasons, but on a recipe-sharing site like Food.com the
clearest signal of whether a recipe "worked" is the rating it earns from the people who
made it. This project investigates a single guiding question:

> **Are quicker recipes rated more highly than longer recipes on Food.com?**

The motivation is practical. If you are deciding what to publish — or what to cook — it
would be useful to know whether the time a recipe demands is associated with how much
people end up enjoying it.

The analysis uses the **Recipes and Ratings** dataset, which combines two tables. The
`recipes` table contains **83,782 recipes** posted to Food.com since 2008, and the
`interactions` table contains roughly **731,000 user reviews and ratings** of those
recipes. After merging them and computing each recipe's average rating, the working
dataset has one row per recipe.

The columns most relevant to the question are:

| Column | Description |
| --- | --- |
| `minutes` | Preparation time in minutes (the basis for "quick" vs. "long") |
| `avg_rating` | Mean star rating a recipe received across all of its reviews |
| `n_steps` | Number of steps in the recipe |
| `n_ingredients` | Number of ingredients |
| `nutrition` | List of nutrition values (calories, fat, sugar, etc.), later parsed |
| `tags` | List of descriptive tags (e.g. `easy`, `desserts`) |

---

## Data Cleaning and Exploratory Data Analysis

**Cleaning.** I performed several cleaning steps so the data matched the way ratings
actually behave on the site:

1. **Treated a rating of 0 as missing.** Food.com records a "review with no star rating"
   as a `0`, which is not a real one-star-equivalent score. Leaving these in would
   artificially drag every average toward zero, so I replaced `0` with `NaN` before
   aggregating.
2. **Computed `avg_rating` per recipe** by grouping interactions on the recipe id and
   taking the mean of the (now cleaned) ratings, then merged this back onto the recipe
   table so every recipe has one average rating.
3. **Parsed the `nutrition` column** from a string-encoded list into seven numeric
   columns: `calories`, `total_fat_pdv`, `sugar_pdv`, `sodium_pdv`, `protein_pdv`,
   `saturated_fat_pdv`, and `carbohydrates_pdv`.
4. **Parsed `tags`, `steps`, and `ingredients`** from strings into real Python lists, and
   converted `submitted` to a datetime.
5. **Flagged implausible `minutes` values** (the maximum is over a million minutes) but
   kept them, restricting to a plausible range only inside the hypothesis test.

The head of the cleaned recipe table looks like this:

| name | minutes | n_steps | n_ingredients | avg_rating |
| --- | --- | --- | --- | --- |
| 1 brownies in the world ... | 40 | 10 | 9 | 4.00 |
| 1 in canada chocolate chip cookies | 45 | 12 | 11 | 5.00 |
| 412 broccoli casserole | 40 | 6 | 9 | 5.00 |

**Univariate analysis.** Preparation time is heavily right-skewed — most recipes take
between roughly 15 and 60 minutes, with a long tail of multi-hour (and clearly erroneous)
entries. Average ratings, by contrast, are extremely concentrated near five stars. This
"J-curve" — where users overwhelmingly leave high ratings and rarely bother with low ones
— is a well-documented feature of online review platforms, and it turns out to be the
single most important fact about this dataset.

<iframe src="assets/avg_rating_distribution.html" width="800" height="450" frameborder="0"></iframe>

**Bivariate analysis.** When I bin recipes by preparation time and look at the rating
distribution within each bin, the medians stay pinned at five across every time bucket.
Any relationship between time and rating lives in small shifts of the *mean*, not in the
bulk of the distribution.

<iframe src="assets/minutes_vs_rating.html" width="800" height="450" frameborder="0"></iframe>

**Interesting aggregates.** Grouping recipes into preparation-time buckets and computing
the mean rating per bucket shows the mean drifting slightly downward as recipes get
longer — but the differences only appear in the third decimal place, foreshadowing both
the hypothesis-test result and the modeling difficulty later on.

| Time bucket | Mean `avg_rating` |
| --- | --- |
| ≤ 15 min | 4.65 |
| 15–30 min | 4.64 |
| 30–60 min | 4.62 |
| > 60 min | 4.60 |

---

## Assessment of Missingness

**NMAR discussion.** I believe the missingness of `rating` is plausibly **NMAR** (Not
Missing At Random). A user who finishes a recipe and feels neutral or mildly disappointed
is less likely to come back and leave a star rating than someone who loved it — so whether
a rating is missing depends on the (unobserved) rating itself. To move this toward an MAR
explanation, I would want additional data such as whether the user left a written comment
without a score, or how long after saving the recipe they returned.

**Missingness dependency tests.** I tested whether the missingness of `rating` depends on
other columns using permutation tests on the difference in means between the
rating-missing and rating-present groups (500 shuffles each).

- **Depends on `n_steps`:** observed difference in mean `n_steps` of **1.34**, p ≈ **0.000**.
  Missingness *does* depend on recipe complexity — more elaborate recipes are more likely
  to go unrated, consistent with an abandonment story.
- **Depends on `n_ingredients`:** I additionally confirmed a column the missingness does
  *not* significantly depend on by testing a feature with no complexity link, which fails
  to reject at α = 0.05.

<iframe src="assets/missingness_n_steps_distributions.html" width="800" height="450" frameborder="0"></iframe>

---

## Hypothesis Testing

I tested whether quick recipes are rated more highly than long ones.

- **Null hypothesis:** Quick recipes (≤ 30 min) and longer recipes (> 30 min) have the
  same distribution of average ratings; any observed difference in mean `avg_rating` is
  due to chance.
- **Alternative hypothesis:** Quick recipes have a *higher* mean `avg_rating` than longer
  recipes.
- **Test statistic:** the signed difference in group means,
  T = mean(avg_rating | quick) − mean(avg_rating | long).
- **Method:** a one-sided permutation test with 2,000 shuffles at α = 0.05. I chose a
  permutation test over a t-test because `avg_rating` is heavily left-skewed and bounded
  at 5, which violates the normality assumptions a t-test leans on.

**Result.** Quick recipes averaged **4.6446** (n = 36,418) and long recipes **4.6092**
(n = 44,212), an observed difference of **+0.0355**. None of the 2,000 permutations
produced a difference this large, giving **p ≈ 0.000**.

<iframe src="assets/hypothesis_test_null.html" width="800" height="450" frameborder="0"></iframe>

**Conclusion.** I reject the null hypothesis at the 5% level: the data are consistent with
quick recipes receiving higher average ratings than longer recipes. This is not proof that
shorter recipes are *better* — a permutation test does not establish causation, and the
practical magnitude (about 0.035 stars) is small. Plausible mechanisms include selection
(people attempt quick recipes when they specifically want something fast, and rate from a
happier baseline) and forgiveness (raters may hold elaborate recipes to a higher standard).

---

## Framing a Prediction Problem

I framed a **regression** problem: predict a recipe's `avg_rating` from metadata available
**at the moment it is first posted** — `minutes`, `n_steps`, `n_ingredients`, `n_tags`,
the parsed nutrition columns, and two tag indicators. Every feature is known before any
user rates the recipe, so nothing leaks from the response. I evaluate with **RMSE**
(primary, in star units) alongside R², and drop recipes that were never rated, since they
have no target.

---

## Baseline Model

The baseline is a linear regression on two quantitative features, `n_steps` and
`n_ingredients`, with no transformations and an 80/20 train-test split.

- **Test RMSE:** ≈ **0.637**
- **Test R²:** ≈ **0.00**

This is statistically indistinguishable from naively predicting the global mean rating for
every recipe (naive RMSE ≈ 0.637). Given the J-curve seen in the EDA, this is unsurprising:
two raw count features carry almost no information about a rating that is nearly always
close to five. The baseline establishes the floor the final model must beat.

---

## Final Model

For the final model I engineered features the EDA suggested might matter and switched to a
more expressive estimator:

- **`minutes`**, log-transformed (`log1p`) to tame its extreme right skew, then scaled.
- The **seven parsed nutrition columns**, capturing what *kind* of recipe it is.
- **`n_tags`** plus binary indicators **`is_easy`** and **`is_dessert`**.

All features feed a `RandomForestRegressor` inside a single `Pipeline` with a
`ColumnTransformer`. I tuned `max_depth` and `min_samples_leaf` with `GridSearchCV`
(3-fold, scoring on RMSE). The best hyperparameters were `max_depth = 8` and
`min_samples_leaf = 50` — a shallow, well-regularized forest.

- **Final test RMSE:** ≈ **0.633** (vs. baseline 0.637)
- **Final test R²:** ≈ **0.01**

The improvement over the baseline is real but very small. The honest conclusion is that
recipe metadata is only weakly predictive of average rating: because almost every recipe
is rated near 5, there is very little rating variance for any feature set to explain. The
richer features help at the margin but cannot overcome the ceiling imposed by the J-curve.

---

## Fairness Analysis

Finally, I asked whether the model is equally accurate for **quick** recipes (≤ 30 min) and
**long** recipes (> 30 min) — the same grouping as the hypothesis test.

- **Group X:** quick recipes; **Group Y:** long recipes.
- **Evaluation metric:** RMSE within each group.
- **Null hypothesis:** the model is fair — RMSE is the same for both groups; any
  difference is due to chance.
- **Alternative hypothesis:** the model's RMSE differs between the groups.
- **Test statistic:** RMSE(quick) − RMSE(long), via a two-sided permutation test (2,000
  shuffles).

**Result.** RMSE was **0.600** on quick recipes and **0.659** on long recipes, an observed
difference of **−0.059** with **p ≈ 0.000**.

<iframe src="assets/fairness_permutation.html" width="800" height="450" frameborder="0"></iframe>

**Conclusion.** I reject the null hypothesis of equal performance: the model predicts quick
recipes' ratings meaningfully more accurately than long recipes'. A plausible explanation
is that longer, more elaborate recipes have more variable outcomes and more polarized
raters, making their average ratings intrinsically harder to predict from metadata alone.
This is a real limitation worth surfacing — a tool built on this model to recommend
"likely-to-be-loved" recipes would be systematically less reliable for the long-form
cooking population.
