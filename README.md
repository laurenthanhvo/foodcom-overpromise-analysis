# Evaluating Consistency Between Recipe Descriptions and Home Cook Ratings

In today’s world of food blogging and recipe sharing, authors often use enthusiastic marketing-style language—words like “delicious,” “amazing,” and “you will **LOVE** this”—to entice home cooks to try their creations. However, some recipes hyped in their descriptions fail to live up to expectations, receiving low ratings despite the promise of “deliciousness.”

In this project, we analyze a dataset of **83,782** recipes scraped from Food.com to answer the question:

> **Can we detect, based solely on a recipe’s publish-time attributes (ingredients, steps, nutrition, and description text), when a recipe author has overpromised and home cooks end up rating it poorly?**

Understanding this “mismatch” between marketing language and actual user satisfaction can help recipe platforms flag recipes likely to disappoint, improve recommendation engines, and guide authors toward more honest, data-driven descriptions.

---

## Dataset Overview

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
