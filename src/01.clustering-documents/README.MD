# Document Clustering and Classification

## Data Source

THE DATA COMES FROM [Philips official technical documentation](https://www.lighting.philips.co.uk/oem-emea/support/technical-downloads).

## Methodology

### 1. Document Embedding

Compute a weighted mean of document chunk embeddings:

$$
\mathbf{v}_{\text{doc}} = \frac{\sum_i w_i \mathbf{v}_i}{\|\sum_i w_i \mathbf{v}_i\|}
$$

**Weights** $w_i$: chunk length, TF-IDF, or heading boost (e.g., title × 3).

**Optional SIF**: Use IDF weights and remove the first PCA component to reduce common noise.

### 2. Cluster Documents by Topic

- Run k-means or HDBSCAN on $\mathbf{v}_{\text{doc}}$
- Use UMAP for visualization if needed

### 3. Auto-label Clusters (no LLM)

**Step 1:** Prepare a small set of label phrases (e.g., "LED bulb specs", "installation guide", "warranty", "marketing brochure")

**Step 2:** Embed each label once → $\boldsymbol{\ell}_j$

**Step 3:** For each cluster centroid $\mathbf{c}$, pick label with max cosine similarity:

$$
\arg\max_j \cos(\mathbf{c}, \boldsymbol{\ell}_j)
$$

**Step 4:** Extract top n-grams from docs in the cluster; pick the highest-cosine n-grams to refine the label

### 4. Document-level Classification (Direct)

- For each doc vector $\mathbf{v}_{\text{doc}}$, compute cosine similarity to label embeddings $\boldsymbol{\ell}_j$ and assign the best match
- Use a confidence threshold (e.g., margin between top-1 and top-2 > 0.05)


## Instructions

```
python3 -m venv .venv

source .venv/bin/activate

pip install -r requrements.txt 

```


Have .env file with the following keys

```
AZURE_OPENAI_API_KEY=
AZURE_OPENAI_ENDPOINT=
AZURE_OPENAI_API_VERSION=2024-08-01-preview
```