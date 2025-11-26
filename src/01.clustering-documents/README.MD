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

## Comparison of Approaches

| Method | Notebook | Algorithm | Cluster Detection | Pros | Cons |
|--------|----------|-----------|-------------------|------|------|
| **Elbow Method** | 02 | K-Means | Inertia curve | Fast, simple, visual | Elbow point can be ambiguous |
| **HDBSCAN** | 03 | Density-based | Automatic | No k needed, finds outliers, variable shapes | Requires density parameters tuning |
| **Silhouette** | 04 | K-Means | Quality metric | Clear optimization criterion | Computationally more expensive |

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