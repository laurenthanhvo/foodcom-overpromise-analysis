# Recipe Overpromise Analytics: A Predictive Study of Online Recipes

In today’s world of food blogging and recipe sharing, authors often use enthusiastic marketing-style language—words like “delicious,” “amazing,” and “you will **LOVE** this”—to entice home cooks to try their creations. However, some recipes hyped in their descriptions fail to live up to expectations, receiving low ratings despite the promise of “deliciousness.”

In this project, we analyze a dataset of **83,782** recipes scraped from Food.com to answer the question:

> **Can we detect, based solely on a recipe’s publish-time attributes (ingredients, steps, nutrition, and description text), when a recipe author has overpromised and home cooks end up rating it poorly?**

Understanding this “mismatch” between marketing language and actual user satisfaction can help recipe platforms flag recipes likely to disappoint, improve recommendation engines, and guide authors toward more honest, data-driven descriptions.

---

## Introduction

Our merged dataset contains **83,782** rows. For this prediction problem, we focus on the following columns:

| Column Name           | Description                                                                         |
| --------------------- | ----------------------------------------------------------------------------------- |
| `description`         | Original text description written by the recipe author.                             |
| `avg_rating`          | Average user rating (1–5 stars); recipes without ratings are marked missing.        |
| `n_ingredients`       | Number of ingredients listed in the recipe.                                         |
| `n_steps`             | Number of preparation steps.                                                        |
| `minutes`             | Estimated total cook + prep time in minutes.                                        |
| `calories`            | Total calories per serving.                                                         |
| `protein_g`           | Grams of protein per serving.                                                       |
| `carbs_g`             | Grams of carbohydrates per serving.                                                 |
| `tag_list`            | List of categorical tags (e.g., “vegetarian”, “dessert”).                           |
| `desc_has_delicious`  | Boolean flag: whether the description contains the word “delicious.”                |
| `mismatch`            | Target label: 1 if `desc_has_delicious` is true but `avg_rating` < 4; otherwise 0.  |

---

## Data Cleaning & Exploratory Data Analysis

### 1. Data Cleaning

After merging our recipe metadata, nutrition facts, and ratings, we discovered:

- **Missing `avg_rating`** (3.1% of rows):  
  These recipes never received a rating, so we dropped them—our “mismatch” target depends on a true rating.  

- **Missing `description`** (0.08%):  
  We filled these with the empty string so our `.str.contains("delicious")` check still works.  

- **Outliers in `minutes` & `calories`**:  
  A few recipes claimed hundreds of thousands of minutes or calories—almost certainly typos.  
  We capped cook time at 1,440 min (24 h) and calories at 10,000 kcal.  

- **Type conversions**:  
  Parsed our submission timestamps into `datetime64`, cast numeric fields to floats,  
  and turned our binary flags (`desc_has_delicious`, `mismatch`) into integers.  

#### Cleaned DataFrame (first 5 rows)

| id     | name                                    | submitted   | n_ingredients | avg_rating | calories | desc_has_delicious | mismatch |
| ------ | --------------------------------------- | ----------- | ------------- | ---------- | -------- | ------------------ | -------- |
| 333281 | brownies in the world best ever         | 2008-10-15  | 9             | 4.0        | 138.4    | 0                  | 0        |
| 453467 | 1 in canada chocolate chip cookies      | 2011-04-23  | 11            | 5.0        | 595.1    | 0                  | 0        |
| 306168 | 412 broccoli casserole                  | 2008-05-07  | 9             | 5.0        | 194.8    | 0                  | 0        |
| 286009 | millionaire pound cake                  | 2008-02-11  | 7             | 5.0        | 878.3    | 0                  | 0        |
| 475785 | 2000 meatloaf                           | 2012-03-29  | 13            | 5.0        | 267.0    | 0                  | 0        |

---

### 2. Distribution of Number of Ingredients

<iframe
  src="assets/n_ingredients_dist.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

> *Nearly three‐quarters of recipes use between 5 and 15 ingredients (the box shows the 25th–75th percentile), but there’s a long tail of complex recipes up to 30+ ingredients.*

---

### 3. Average Rating vs. Number of Ingredients

<iframe
  src="assets/avg_rating_vs_ingredients.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

> *The fitted trendline is essentially flat, indicating that recipe length (ingredient count) by itself does not strongly predict whether home cooks rate the recipe higher or lower.*

---

### 4. Cook Time Groups – Mean Rating

| Cook Time     | Mean Rating | # Recipes |
| ------------- | ----------- | --------- |
| ≤ 30 minutes  | 4.62        | 21,845    |
| > 30 minutes  | 4.64        | 61,937    |

> *Longer recipes (>30 min) have a slightly higher average rating (4.64 vs. 4.62), but the difference is very small—cook time alone is not a reliable overpromise signal.*

---

## Assessment of Missingness

### NMAR vs. MAR

We do **not** believe any column in our dataset is NMAR (“Not Missing At Random”).  The only missingness in our cleaned data is in the `avg_rating` column (3.11% of recipes never received a rating).  This is almost certainly because those recipes were unpopular or not viewed—an **observable** phenomenon tied to recipe visibility or share counts—rather than being missing due to an unobserved “true” rating.  If we could obtain page‐view or social‐share data for each recipe, we could explain away this missingness (making it MAR).

### Missingness Summary

- **Missing `avg_rating`**: 2,609 recipes (3.11%)  
- **Observed `avg_rating`**: 81,173 recipes  

### Permutation Tests

We performed two permutation tests (B = 5 000) on the test‐set to see whether recipes **with** missing ratings differ systematically on:

1. **Number of Ingredients**  
2. **“Contains the word ‘delicious’”** flag

Let  
- **X** = distribution of the feature among recipes with _observed_ ratings  
- **Y** = distribution of the feature among recipes with _missing_ ratings  
- **Δ** = mean(Y) – mean(X)

We then compared the observed Δ to its null distribution under random label‐permutations.

---

#### 3.1 Δ mean(`n_ingredients`) under H₀
<iframe
  src="assets/permtest_1.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

- **Observed Δ =** 0.254  
- **Permutation p-value (two-sided) =** 0.0006  

> **Interpretation:** Although the difference in average ingredient count between “missing” vs. “not missing” recipes is very small (≈0.25 ingredients), it is statistically significant.  Recipes that never received a rating tend to have marginally more ingredients—but the effect size is negligible for practical predictive use.

---

#### 3.2 Δ mean(`desc_has_delicious`) under H₀
<iframe
  src="assets/permtest_2.html"
  width="800"
  height="450"
  frameborder="0"
></iframe>

- **Observed Δ =** 0.001  
- **Permutation p-value (two-sided) =** 0.9246  

> **Interpretation:** There is no evidence that the “contains ‘delicious’” flag differs between rated vs. unrated recipes.  Missingness in `avg_rating` is not associated with our target‐related text feature.

---

#### Conclusion

- No column in our dataset is NMAR.  
- Missingness in `avg_rating` can be explained by recipe **popularity** (an observable factor), so it is MAR.  
- Because the only non‐zero—but tiny—difference was in `n_ingredients`, we feel comfortable dropping unrated recipes without introducing substantive bias in our “mismatch” labels.

---

## Hypothesis Testing

### Null & Alternative Hypotheses

**Null hypothesis (H₀):**  
The mean average rating for recipes whose descriptions _contain_ “delicious” is the same as for recipes whose descriptions _do not_ contain “delicious.”

**Alternative hypothesis (H₁):**  
Recipes whose descriptions contain “delicious” have a _different_ mean average rating than those that do not.

---

### Test Statistic & Significance Level

We use the difference in group means as our test statistic:

> **T** = mean( avg_rating | desc_has_delicious = 1 )  
> &nbsp;&nbsp;– mean( avg_rating | desc_has_delicious = 0 )

Because we make no strong parametric assumptions about the rating distributions, we estimate the null distribution via a two-sided permutation test (**B = 5000** shuffles of the `desc_has_delicious` labels). We choose a conventional significance level of **α = 0.05**.

---

### Results

- **Observed test statistic** (Tₒbₛ):  
  mean(ratings | “delicious”) – mean(ratings | not “delicious”) = **0.023**
- **Permutation p-value (two-sided):**  
  **p = 0.0028**

Since _p_ < α, we reject H₀ and conclude there is a statistically significant difference in mean ratings between the two groups. However, the effect size is very small (~0.02 stars), so while “delicious” in the description correlates with a slightly higher rating, it is not by itself a practically strong over-promise signal.

---

### Null Distribution of Δ mean(avg_rating)

<iframe
  src="assets/permtest_3.html"
  width="800" height="450"
  frameborder="0">
</iframe>

<em>
The gray bars show the permutation-test null distribution of Δ mean(avg_rating), and the vertical red dashed line marks our observed Δ = 0.023. Only ~0.28% of shuffled labelings produced a difference as extreme or more extreme, yielding p = 0.0028.
</em>

---

## Framing a Prediction Problem

### Prediction Type  
This is a **binary classification** problem: given only information known at publication, we predict whether a recipe “over-promises” in its description and then “under-delivers” on user ratings.

---

### Response Variable (_y_)  
We define: mismatch = 1 if desc_has_delicious == 1 AND avg_rating < 4.0, mismatch = 0 otherwise

We chose this target because our central question is  
> “Can we detect, at publish-time, which recipes hyped as ‘delicious’ home cooks end up rating poorly?”

---

### Features (_X_)  
We only use fields available the moment the recipe goes live:

- **Quantitative**:  
  `n_ingredients`, `n_steps`, `minutes`,  
  `calories`, `protein_g`, `carbs_g`, `num_tags`
- **Binary**:  
  `desc_has_delicious`

Any rows missing these features are dropped so the model never “peeks” at post-publication data (e.g. actual user reviews).

---

### Why Binary Classification?  
There are exactly two outcomes—“mismatch” versus “not mismatch”—so binary classification is the natural choice.

---

### Evaluation Metric  
We use the **F₁-score** rather than plain accuracy because the positive class  
(`mismatch = 1`) is very rare (~0.65% of all recipes).  
F₁ balances precision (avoiding false alarms) and recall (catching true mismatches).

---

### Time-of-Prediction Justification  
All features (`n_ingredients`, `minutes`, etc.) are known immediately upon publication.  
We deliberately exclude any later data—no post-publication edits, user comments, or subsequent ratings—so our model truly simulates “real-time” predictions.

---

### Train/Test Split & Class Balance

| Dataset   | # Records | Mismatch = 1 (%) |
|-----------|-----------|------------------|
| Training  | 64,938    | 0.65%            |
| Test      | 16,235    | 0.65%            |

We performed a stratified 80/20 split to preserve the roughly 1-in-150 rate of mismatches in both sets.

---

## Baseline Model

### Model Type  
Baseline binary classifier using **Logistic Regression**, implemented as a single `sklearn.Pipeline`:

1. **Scaler**: `StandardScaler()` to put all features on the same scale  
2. **Classifier**: `LogisticRegression(random_state=42, max_iter=2000, class_weight="balanced")`

---

### Features Used (8 total)

- **Quantitative (7)**  
  `n_ingredients` (number of ingredients)  
  `n_steps` (number of recipe steps)  
  `minutes` (total cook + prep time)  
  `calories`, `protein_g`, `carbs_g` (nutritional values)  
  `num_tags` (number of tags)  

- **Nominal/Binary (1)**  
  `desc_has_delicious` (1 if description contains “delicious”, else 0)  

All quantitative features were standardized; the single binary feature was used as-is (no additional encoding).

---

### Performance on Test Set

| Class             | Precision | Recall | F1-score | Support |
|-------------------|:---------:|:------:|:--------:|:-------:|
| **0 (no mismatch)** | 1.0000   | 0.9012 |  0.9481  | 16130   |
| **1 (mismatch)**    | 0.0618   | 1.0000 |  0.1165  |   105   |
| **Accuracy**        |          |        |  0.9019  | 16235   |
| **Macro avg**       | 0.5309   | 0.9506 |  0.5323  | 16235   |
| **Weighted avg**    | 0.9939   | 0.9019 |  0.9427  | 16235   |

```text
Confusion Matrix:
[[14537  1593]
 [    0   105]]

- **Accuracy** is high (≈90.2%) because “no-mismatch” is the dominant class.

- **Positive-class** F1 (mismatch=1) is only 0.1165, despite perfect recall (1.0), due to very low precision (0.0618).

**Interpretation:**
Although this baseline almost never misses a true mismatch (recall = 100%), it generates many false positives, so its precision—and hence F1—is extremely low. In a real deployment, this would lead to too many false alarms. Therefore, the current model is not “good” enough and motivates more sophisticated modeling or feature engineering.

### Model Coefficients (sorted by magnitude)

| Feature                   | Coefficient |
|---------------------------|------------:|
| **desc_has_delicious**  |       4.9430 |
| **carbs_g**             |       0.2487 |
| **protein_g**           |       0.1002 |
| **minutes**             |       0.0523 |
| **n_steps**             |       0.0038 |
| **n_ingredients**       |       0.0004 |
| **num_tags**            |      −0.0317 |
| **calories**            |      −0.2702 |

The large positive weight on `desc_has_delicious` confirms it is the strongest single predictor, but the overall poor precision shows we need richer features or a more flexible model to reliably detect mismatches.

---

## Final Model

### New Feature Engineering

- **`cal_per_ing`** = `calories` / (`n_ingredients` + 1)  
  _Captures average caloric density per ingredient; very rich recipes may disappoint if over-hyped._

- **`time_per_step`** = `minutes` / (`n_steps` + 1)  
  _Measures average time per instruction step; extreme pacing can affect user satisfaction._

- **`prot_carb_ratio`** = `protein_g` / (`carbs_g` + 1)  
  _Encodes macronutrient balance; unbalanced recipes (too low protein or too high carbs) may under-deliver._

By grounding each feature in the data-generating process (ingredient complexity, effort distribution, and nutritional profile), we hypothesize these proxies help the model detect when a “delicious” description overpromises.

---

### Modeling Algorithm & Hyperparameter Tuning

We switched from a linear model to a **Random Forest Classifier** to capture nonlinear interactions among both raw and derived features.

**Pipeline**  
```python
from sklearn.pipeline import Pipeline
from sklearn.ensemble import RandomForestClassifier

pipeline = Pipeline([
  ("clf", RandomForestClassifier(random_state=42))
])

**Hyperparameter grid** (GridSearchCV, 5-fold CV, scoring="f1"):
```python
param_grid = {
  "clf__n_estimators": [100, 200],
  "clf__max_depth":    [None, 10, 20],
  "clf__class_weight": ["balanced"]
}

**Best parameters** found via cross‐validation:
```python
{'clf__class_weight': 'balanced',
 'clf__max_depth': 20,
 'clf__n_estimators': 100}

---

### Final Model vs. Baseline
| Model                      | Precision | Recall | F₁-score |
| -------------------------- | --------: | -----: | -------: |
| Logistic Regression (base) |    0.0618 |  1.000 |   0.1165 |
| Random Forest (final)      |    0.0734 |  0.495 |   0.1279 |

- **Precision ↑** from 0.0618 → 0.0734 (fewer false positives)

- **Recall ↓** from 1.000 → 0.495 (trade‐off accepted to reduce alarms)

- **F₁-score ↑** from 0.1165 → 0.1279

---

### Confusion Matrix (Final Model)
[[15474   656]
 [   53    52]]

**Interpretation:** The Random Forest trades off some recall for a substantial precision gain, yielding an overall higher F₁-score. By combining publish-time and engineered features in a flexible tree-based model, we make measurable progress toward flagging mismatches without overwhelming users with false alarms.

## Fairness Analysis

**Group X:** “simple” recipes (`n_ingredients ≤ 5`)  
**Group Y:** “complex” recipes (`n_ingredients > 5`)

### Evaluation Metric
We assess fairness by comparing the **precision** for the positive class (`mismatch = 1`) in each group.

### Null & Alternative Hypotheses
- **H₀:** precision₁(simple) ≈ precision₁(complex) (any difference is due to chance)  
- **H₁:** precision₁(simple) ≠ precision₁(complex)

### Test Statistic & Significance Level
We define our test statistic as  
> **Δ precision** = precision(simple) − precision(complex)

To estimate its null distribution, we perform a **two-sided permutation test** with **B = 5000** random shuffles of the `mask_simple` indicator. We use a conventional significance level of **α = 0.05**.

---

### Results

- **Observed precision (simple):** 0.0638  
- **Observed precision (complex):** 0.0741  
- **Observed Δ precision:** 0.0638 − 0.0741 = **−0.0103**  
- **Permutation p-value (two-sided):** **0.6826**

Since _p_ > α, we **fail to reject H₀**. There is no statistically significant difference in mismatch detection precision between simple and complex recipes.

---

### Null Distribution of Δ Precision

<iframe
  src="assets/permtest_4.html"
  width="800" height="450"
  frameborder="0">
</iframe>

<em>
The histogram shows the permutation-test null distribution of Δ precision under random group assignments, and the vertical red dashed line marks our observed Δ = −0.0103. With p = 0.6826, the observed gap is consistent with chance.
</em>

###Conclusion
Our model’s ability to flag “over-promised” recipes does not differ meaningfully between simple and complex recipes, at least as measured by precision. This suggests that, along the axis of recipe complexity, our mismatch detector behaves fairly, without unduly favoring one group over the other.
