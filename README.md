# Quantifying Discrepancies Between Health-Related Metadata and Nutritional Content in Online Recipe Datasets
### To what extent do semantic health labels (e.g., “low fat”, “gluten free”, “healthy”) embedded in recipe names and tags correspond to quantified nutritional values, and how do these discrepancies influence user-assigned ratings?

by Sunny Sun (chs019@ucsd.edu)

## Introduction

I like food and I like cooking, and I’m also really into staying on a healthy diet—so I’m pretty familiar with nutrition data. That’s why I’ve always wondered if recipes labeled as “healthy” actually live up to their name. Do these labels reflect real nutritional value, or are they just used to make recipes sound better? In this project, I explore that question using the Recipes and Ratings dataset from Food.com.

My main research question is: **Are recipes labeled as “healthy” actually healthier based on nutrition facts, and does that label influence user ratings?** To answer this, I created a Healthy Index using normalized nutrition values, compared labeled vs. unlabeled recipes, and predicted user ratings based on both nutrition and label metadata.

The dataset contains about 230,000 recipes and includes metadata such as prep time, ingredient count, nutrition info (in %DV), and user-submitted ratings. Key columns include:
- `tags`: list of descriptive labels (e.g. “gluten free”, “healthy”)
- `nutrition`: [calories, total fat, sugar, sodium, protein, saturated fat, carbs] in %DV
- `name`: recipe title
- `average_rating`: average user rating (0–5)

---

## Data Cleaning and Exploratory Data Analysis

To prepare the data, I:
- Merged recipe metadata with rating data and calculated `average_rating` per recipe
- Parsed the `nutrition` column into 7 separate numeric columns
- Normalized nutrition fields using MinMaxScaler
- Created binary indicators for labels like `"healthy"`, `"low fat"`, and `"gluten free"` by checking both `tags` and `name`
- Defined a custom `healthy_index` based on nutrition guidelines (rewarding protein, penalizing sugar, sodium, fat, etc.)

Here is a comparison of the Healthy Index between recipes labeled "healthy" and those without the label:

<iframe
  src="assets/healthy_index_by_label.html"
  width="800"
  height="600"
  frameborder="0"
></iframe>

As shown above, "healthy"-labeled recipes generally have higher Healthy Index values. However, the distributions still overlap, which raises the question of how consistent or meaningful these labels are. We'll explore this further in the hypothesis testing section.
