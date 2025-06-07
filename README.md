# Evaluating Consistency Between Recipe Descriptions and Home Cook Ratings

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

<section id="hypothesis-testing" class="prose lg:prose-xl mx-auto my-12">
  <h2>Step 4: Hypothesis Testing</h2>

  <h3>Null &amp; Alternative Hypotheses</h3>
  <p><strong>Null hypothesis (H<sub>0</sub>):</strong>  
     The mean average rating for recipes whose descriptions <em>contain</em> “delicious” is the same as for recipes whose descriptions <em>do not</em> contain “delicious.”</p>
  <p><strong>Alternative hypothesis (H<sub>1</sub>):</strong>  
     Recipes whose descriptions contain “delicious” have a <em>different</em> mean average rating than those that do not.</p>

  <h3>Test Statistic &amp; Significance Level</h3>
  <p>We use the difference in group means as our test statistic:</p>
  <blockquote>
    T = mean(avg_rating | desc_has_delicious = 1) – mean(avg_rating | desc_has_delicious = 0)
  </blockquote>
  <p>
    Because we make no strong parametric assumptions about the rating distributions, we estimate the null distribution via a two-sided permutation test (B = 5 000 shuffles of the <code>desc_has_delicious</code> labels).  
    We choose a conventional significance level of <var>α</var> = 0.05.
  </p>

  <h3>Results</h3>
  <ul>
    <li><strong>Observed test statistic:</strong>  
        T<sub>obs</sub> = mean(ratings | “delicious”) – mean(ratings | not “delicious”) = 0.023</li>
    <li><strong>Permutation p-value (two-sided):</strong>  
        p = 0.0028</li>
  </ul>
  <p>
    Since p &lt; α, we reject H<sub>0</sub> and conclude there is a statistically significant difference in mean ratings between the two groups.  
    However, the effect size is very small (~0.02 stars), so while “delicious” in the description correlates with a slightly higher rating, it is not by itself a practically strong over-promise signal.
  </p>

  <h3>Null Distribution of Δ mean(avg_rating)</h3>
  <iframe
    src="assets/permtest_3.html"
    width="800"
    height="450"
    frameborder="0"
  ></iframe>
  <p>
    <em>
      The gray bars show the permutation-test null distribution of Δ mean(avg_rating), and the vertical red dashed line marks our observed Δ = 0.023.  
      Only ~0.28% of shuffled labelings produced a difference as extreme or more extreme, yielding p = 0.0028.
    </em>
  </p>
</section>

