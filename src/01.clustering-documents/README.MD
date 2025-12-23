# Document Clustering Project

## Data Source

The data comes from [Philips official technical documentation](https://www.lighting.philips.co.uk/oem-emea/support/technical-downloads).

## Overview

This project explores different approaches to clustering PDF documents using embeddings and various clustering algorithms. The workflow consists of four main notebooks, each implementing a different strategy for document analysis and clustering.

---

## Notebook 1: Document Embedding (`01.embedding.ipynb`)

### Purpose
Extract text from PDF documents and generate page-level embeddings using Azure OpenAI's text embedding model.

### Process

1. **Text Extraction**: Uses PyMuPDF (fitz) to extract native text from PDF documents page by page
   - Simple extraction method prioritized for speed
   - Alternative OCR method (PaddleOCR) available but slower

2. **Embedding Generation**: 
   - Each page's text is embedded using Azure OpenAI's `text-embedding-3-large` model
   - Embeddings are normalized to unit length for consistent cosine similarity calculations
   - Formula for normalization:
   
   $$
   \mathbf{v}_{\text{norm}} = \frac{\mathbf{v}}{\|\mathbf{v}\|}
   $$

3. **Storage**: 
   - Results saved as individual JSON files (one per document) in `data_output/embeddings/`
   - Each JSON contains: original file path, extracted page texts, and page-level embeddings

### Output
- JSON files containing document metadata, page text, and embeddings for subsequent clustering approaches

---

## Notebook 2: K-Means with Elbow Method (`02.clustering-approach-01.ipynb`)

### Purpose
Cluster documents using K-Means with automatic cluster number detection via the Elbow Method.

### Process

1. **Document Vector Computation**: 
   - Aggregates page-level embeddings into a single document vector using weighted mean
   - Equal weights for all pages (uniform weighting)
   - Formula:
   
   $$
   \mathbf{v}_{\text{doc}} = \frac{\sum_i w_i \mathbf{v}_i}{\|\sum_i w_i \mathbf{v}_i\|}
   $$
   
   where $w_i$ are weights (default: all ones)

2. **Optimal Cluster Detection (Elbow Method)**:
   - Tests K-Means with k ranging from 2 to 20 clusters
   - Calculates inertia (sum of squared distances to nearest cluster center) for each k
   - Uses `KneeLocator` library to automatically detect the "elbow point" where inertia reduction plateaus
   - Visualizes elbow curve with optimal k marked

3. **Clustering**: 
   - Applies K-Means with the automatically detected optimal k
   - Assigns each document to a cluster

4. **Organization**: 
   - Copies original PDF files into cluster-specific folders in `data_output/clusters-01/`
   - Each cluster folder contains documents grouped by semantic similarity

### Output
- Optimal number of clusters determined automatically
- Documents organized into `data_output/clusters-01/cluster{N}/` folders
- Visualization of elbow curve

---

## Notebook 3: HDBSCAN Clustering (`03.clustering-approach-02.ipynb`)

### Purpose
Cluster documents using HDBSCAN (Hierarchical Density-Based Spatial Clustering), which automatically determines the number of clusters and can identify noise/outliers.

### Process

1. **Document Vector Computation**: 
   - Same weighted mean approach as Notebook 2
   - Aggregates page embeddings into single document vectors

2. **HDBSCAN Clustering**:
   - Parameters:
     - `min_cluster_size=10`: Minimum documents needed to form a cluster
     - `min_samples=10`: Minimum neighborhood density
     - `metric='euclidean'`: Distance metric (equivalent to cosine for normalized vectors)
   - **Advantages over K-Means**:
     - No need to specify number of clusters in advance
     - Can identify outliers (labeled as cluster -1)
     - Finds clusters of varying densities and shapes

3. **Cluster Assignment**: 
   - Documents assigned to discovered clusters
   - Outliers/noise points labeled as cluster -1

4. **Organization**: 
   - Copies documents into cluster folders in `data_output/clusters-02/`
   - Handles variable number of clusters automatically

### Output
- Variable number of clusters determined by data density
- Documents organized into `data_output/clusters-02/cluster{N}/` folders
- Potential identification of outlier documents

---

## Notebook 4: K-Means with Silhouette Method (`04.clustering-approach-03.ipynb`)

### Purpose
Cluster documents using K-Means with optimal cluster number detection via the Silhouette Method, which measures cluster quality.

### Process

1. **Document Vector Computation**: 
   - Same weighted mean approach as previous notebooks

2. **Optimal Cluster Detection (Silhouette Method)**:
   - Tests K-Means with k ranging from 2 to 20
   - For each k, calculates the **Silhouette Score**:
     - Measures how similar a document is to its own cluster vs. other clusters
     - Score ranges from -1 to 1 (higher is better)
     - Formula:
     
     $$
     s(i) = \frac{b(i) - a(i)}{\max(a(i), b(i))}
     $$
     
     where $a(i)$ = average distance to points in same cluster, $b(i)$ = average distance to nearest other cluster
   
   - Selects k with the highest average silhouette score
   - **Advantages over Elbow Method**:
     - More interpretable metric (measures cluster cohesion and separation)
     - Often provides clearer optimal point
     - Uses cosine similarity for normalized embeddings

3. **Clustering**: 
   - Applies K-Means with the k that maximizes silhouette score

4. **Organization**: 
   - Copies documents into cluster folders in `data_output/clusters-03/`

### Output
- Optimal number of clusters based on silhouette score
- Documents organized into `data_output/clusters-03/cluster{N}/` folders
- Quality metric for each tested cluster count

---

## Notebook 5: K-Means with Geometric Median (`05.clustering-approach-04.ipynb`)

### Purpose
Cluster documents using K-Means with Elbow Method, but with a more robust document vector aggregation method: the **geometric median**.

### Process

1. **Document Vector Computation (Geometric Median)**: 
   - Instead of weighted mean, uses geometric median for robustness against outlier pages
   - **Geometric median** minimizes sum of distances to all page vectors
   - Computed via iterative reweighting algorithm:
     - Start with arithmetic mean
     - Iteratively update using inverse distance weighting
     - Converge when change is below threshold (typically 3-10 iterations)
   - Formula (iterative):
   
   $$
   \mathbf{m}^{(t+1)} = \frac{\sum_i \frac{1}{\|\mathbf{v}_i - \mathbf{m}^{(t)}\|} \mathbf{v}_i}{\sum_i \frac{1}{\|\mathbf{v}_i - \mathbf{m}^{(t)}\|}}
   $$
   
   - **Advantages over weighted mean**:
     - Less sensitive to outlier pages (e.g., boilerplate content, table of contents)
     - More stable center for heterogeneous documents
     - Better represents "typical" content

2. **Optimal Cluster Detection**: 
   - Uses Elbow Method (same as Notebook 2)
   - Tests K-Means with k from 2 to 20
   - Detects elbow point in inertia curve

3. **Clustering**: 
   - Applies K-Means with optimal k
   - Assigns documents to clusters

4. **Organization**: 
   - Copies documents into `data_output/clusters-04/` folders

### Output
- Documents organized into `data_output/clusters-04/cluster{N}/` folders
- More robust clustering for documents with diverse page content

---

## Notebook 6: K-Means with PCA Direction (`06.clustering-approach-05.ipynb`)

### Purpose
Cluster documents using K-Means with Elbow Method, but representing each document by its **principal direction** (first principal component) rather than centroid.

### Process

1. **Document Vector Computation (PCA Direction)**: 
   - Applies PCA (Principal Component Analysis) to page embeddings
   - Extracts the **first principal component** as document vector
   - This represents the main direction of variation across pages
   - Formula:
   
   $$
   \mathbf{v}_{\text{doc}} = \text{PC}_1 = \text{argmax}_{\|\mathbf{v}\|=1} \sum_i (\mathbf{v}_i \cdot \mathbf{v})^2
   $$
   
   - **Interpretation**:
     - Captures the "main theme" direction in embedding space
     - Emphasizes consistent topics across pages
     - Reduces influence of page-specific noise
   
   - **Advantages**:
     - Captures dominant semantic direction
     - Less affected by page count (single page vs. multi-page docs)
     - Highlights consistent themes over scattered content

2. **Optimal Cluster Detection**: 
   - Uses Elbow Method
   - Tests K-Means with k from 2 to 20

3. **Clustering**: 
   - Applies K-Means with optimal k

4. **Organization**: 
   - Copies documents into `data_output/clusters-05/` folders

### Output
- Documents organized into `data_output/clusters-05/cluster{N}/` folders
- Clustering based on main thematic direction rather than average content

---

## Notebook 7: K-Means with Attention Pooling (`07.clustering-approach-06.ipynb`)

### Purpose
Cluster documents using K-Means with Elbow Method, but with an **attention-based pooling mechanism** that gives higher weight to pages closer to the document's "center".

### Process

1. **Document Vector Computation (Attention Pooling)**: 
   - Iteratively computes document vector by giving higher attention to pages similar to current center
   - Algorithm:
     1. Start with arithmetic mean of all pages
     2. Compute cosine similarity between each page and current center
     3. Convert similarities to attention weights using softmax-like transformation
     4. Compute new center as weighted average with attention weights
     5. Repeat for fixed iterations (typically 3-5)
   
   - Formula (per iteration):
   
   $$
   \text{sim}_i = \mathbf{v}_i \cdot \mathbf{c}^{(t)}
   $$
   
   $$
   \alpha_i = \frac{\exp(\beta \cdot \text{sim}_i)}{\sum_j \exp(\beta \cdot \text{sim}_j)}
   $$
   
   $$
   \mathbf{c}^{(t+1)} = \frac{\sum_i \alpha_i \mathbf{v}_i}{\|\sum_i \alpha_i \mathbf{v}_i\|}
   $$
   
   where:
   - $\beta$ = sharpness parameter (default: 5.0)
   - Higher $\beta$ → more focus on central pages
   
   - **Advantages**:
     - Self-attention mechanism similar to transformers
     - Automatically downweights peripheral/outlier pages
     - Focuses on core thematic content
     - More sophisticated than simple averaging

2. **Optimal Cluster Detection**: 
   - Uses Elbow Method
   - Tests K-Means with k from 2 to 20

3. **Clustering**: 
   - Applies K-Means with optimal k

4. **Organization**: 
   - Copies documents into `data_output/clusters-06/` folders

### Output
- Documents organized into `data_output/clusters-06/cluster{N}/` folders
- Clustering that emphasizes core document content over peripheral pages

---

## Comparison of Approaches

| Method | Notebook | Algorithm | Cluster Detection | Document Vector Method | Pros | Cons |
|--------|----------|-----------|-------------------|----------------------|------|------|
| **Weighted Mean + Elbow** | 02 | K-Means | Inertia curve | Weighted mean (uniform weights) | Fast, simple, baseline | Sensitive to outlier pages |
| **Weighted Mean + HDBSCAN** | 03 | Density-based | Automatic | Weighted mean (uniform weights) | No k needed, finds outliers | Requires parameter tuning |
| **Weighted Mean + Silhouette** | 04 | K-Means | Quality metric | Weighted mean (uniform weights) | Clear optimization criterion | More expensive |
| **Geometric Median + Elbow** | 05 | K-Means | Inertia curve | Geometric median | Robust to outliers | More computation per doc vector |
| **PCA Direction + Elbow** | 06 | K-Means | Inertia curve | First principal component | Captures main theme direction | May lose nuanced content |
| **Attention Pool + Elbow** | 07 | K-Means | Inertia curve | Iterative attention weighting | Self-attentive, focus on core | Requires hyperparameter (α) |

---

## Key Concepts

### Document Vector Aggregation

All clustering approaches use a weighted mean to combine page-level embeddings:

$$
\mathbf{v}_{\text{doc}} = \frac{\sum_i w_i \mathbf{v}_i}{\|\sum_i w_i \mathbf{v}_i\|}
$$

where:
- $\mathbf{v}_i$ = embedding of page $i$
- $w_i$ = weight for page $i$ (currently uniform: all ones)
- Result is normalized to unit length

**Potential weights** (for future enhancement):
- Page length (number of characters/tokens)
- TF-IDF importance scores
- Heading/title boost (e.g., first page × 3)
- SIF (Smooth Inverse Frequency) weighting

### Clustering Quality Metrics

- **Inertia** (Elbow Method): Sum of squared distances to cluster centers (lower is better, but diminishing returns)
- **Silhouette Score**: Measures cluster cohesion and separation (-1 to 1, higher is better)
- **Density-based**: HDBSCAN finds natural groupings without requiring k


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