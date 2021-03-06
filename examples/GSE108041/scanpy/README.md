# scanpy workflow for `GSE108041` data set

Here, we analyze the `GSE108041` data set ("Extreme heterogeneity of influenza virus infection in single cells", Russell *et al.*, eLIFE (2018), [DOI](https://doi.org/10.7554/eLife.32303), [GEO submission](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE108041)) using [scanpy](https://scanpy.readthedocs.io), following the [basics workflow described on the scanpy website](https://scanpy-tutorials.readthedocs.io/en/latest/pbmc3k.html) which includes similar steps as those performed in Seurat.
Then, import the [AnnData](https://anndata.readthedocs.io/en/stable) object produced by scanpy into Seurat, and from there export it to Cerebro.

## Preparation

Before starting, we clone the Cerebro repository (or manually download it) because it contains the raw data of our example data set.
One (optional) step of our analysis will require us to provide some gene sets in a `GMT` file.
We manually download the `c2.all.v7.0.symbols.gmt` file from [MSigDB](http://software.broadinstitute.org/gsea/downloads.jsp#msigdb) and put it in our current working directory.
Then, we pull the Docker image from the Docker Hub, convert it to Singularity, and start an R session inside.

```sh
git clone https://github.com/romanhaa/Cerebro
cd Cerebro/examples/GSE108041
# download GMT file (if you want) and place it inside this folder
singularity build <path_to>/cerebro_v1.1.simg docker://romanhaa/cerebro:v1.1
singularity exec --bind ./:/data <path_to>/cerebro_v1.1.simg bash
cd /data
python3
```

## Preparation

First, we load the required Python libraries.

```python
import numpy as np
import pandas as pd
import scanpy as sc
```

## Load data

For each of the four samples we load the transcript count matrix (`.h5` format), make feature names unique (some gene IDs share the same gene name), and then merge the transcript counts together.

```python
adata_uninfected = sc.read_10x_h5('raw_data/GSM2888370_Uninfected.h5')
adata_uninfected.var_names_make_unique()

adata_infected_6h = sc.read_10x_h5('raw_data/GSM2888371_6h.h5')
adata_infected_6h.var_names_make_unique()

adata_infected_8h = sc.read_10x_h5('raw_data/GSM2888372_8h.h5')
adata_infected_8h.var_names_make_unique()

adata_infected_10h = sc.read_10x_h5('raw_data/GSM2888374_10h.h5')
adata_infected_10h.var_names_make_unique()

adatas = [adata_uninfected, adata_infected_6h, adata_infected_8h, adata_infected_10h]
adata = adatas[0].concatenate(adatas[1:])
```

Then, we add the sample info to the meta data.

```python
adata.obs['sample'] = 'none'
adata.obs['sample'][adata.obs['batch'].str.startswith('0')] = 'uninfected'
adata.obs['sample'][adata.obs['batch'].str.startswith('1')] = 'infected_6h'
adata.obs['sample'][adata.obs['batch'].str.startswith('2')] = 'infected_8h'
adata.obs['sample'][adata.obs['batch'].str.startswith('3')] = 'infected_10h'
del(adata.obs['batch'])
adata.obs['sample'] = adata.obs['sample'].astype('category')
adata.obs['sample'].cat.reorder_categories(
  ['uninfected','infected_6h','infected_8h','infected_10h'],
  inplace = True
)
```

## Analysis in scanpy

Now, we...

* remove cells with fewer than `100` transcripts or `50` expressed genes,
* calculate the number of transcripts per cell, and
* remove genes expressed in fewer than `10` cells.

```python
sc.pp.filter_cells(adata, min_counts = 101)
sc.pp.filter_cells(adata, min_genes = 51)
sc.pp.filter_genes(adata, min_cells = 10)

adata.obs['n_counts'] = adata.X.sum(axis = 1).A1
```

Next, we export the raw transcript counts after cell and gene filtering so we can load it again later.

```python
np.savetxt('scanpy/raw_counts.tsv', adata.X.todense().astype(int), fmt = '%i', delimiter = '\t')
np.savetxt('scanpy/raw_counts_genes.tsv', adata.var.index, fmt = '%s', delimiter = '\t')
np.savetxt('scanpy/raw_counts_cells.tsv', adata.obs.index, fmt = '%s', delimiter = '\t')
```

What follows is the standard pre-processing procedure, including the following steps...

* normalizing transcript counts per cell,
* bringing transcript counts to log-scale,
* putting transcript counts in raw data slot,
* determine the cell cycle phase (genes taken from Seurat),
* identifying highly variable genes and limiting the expression matrix to those genes,
* regressing out the transcript count per cell,
* scaling gene expression values,
* performing PCA analysis,
* calculating neighbors and clusters of cells, and
* generating dimensional reductions (tSNE + UMAP).

```python
sc.pp.normalize_per_cell(adata)
sc.pp.log1p(adata)
adata.raw = adata
sc.tl.score_genes_cell_cycle(
  adata,
  s_genes = ["MCM5","PCNA","TYMS","FEN1","MCM2","MCM4","RRM1","UNG","GINS2","MCM6","CDCA7","DTL","PRIM1","UHRF1","MLF1IP","HELLS","RFC2","RPA2","NASP","RAD51AP1","GMNN","WDR76","SLBP","CCNE2","UBR7","POLD3","MSH2","ATAD2","RAD51","RRM2","CDC45","CDC6","EXO1","TIPIN","DSCC1","BLM","CASP8AP2","USP1","CLSPN","POLA1","CHAF1B","BRIP1","E2F8"],
  g2m_genes = ["HMGB2","CDK1","NUSAP1","UBE2C","BIRC5","TPX2","TOP2A","NDC80","CKS2","NUF2","CKS1B","MKI67","TMPO","CENPF","TACC3","FAM64A","SMC4","CCNB2","CKAP2L","CKAP2","AURKB","BUB1","KIF11","ANP32E","TUBB4B","GTSE1","KIF20B","HJURP","CDCA3","HN1","CDC20","TTK","CDC25C","KIF2C","RANGAP1","NCAPD2","DLGAP5","CDCA2","CDCA8","ECT2","KIF23","HMMR","AURKA","PSRC1","ANLN","LBR","CKAP5","CENPE","CTCF","NEK2","G2E3","GAS2L3","CBX5","CENPA"]
)
sc.pp.highly_variable_genes(adata)
adata = adata[:, adata.var['highly_variable']]
sc.pp.regress_out(adata, ['n_counts'])
sc.pp.scale(adata)
sc.tl.pca(adata)
sc.pp.neighbors(adata)
sc.tl.louvain(adata, resolution = 0.5)
sc.tl.tsne(adata, perplexity = 30, random_state = 100)
sc.tl.umap(adata, random_state = 100)
```

Now, we write our data to a `.h5ad` file.

```python
adata.write('scanpy/adata.h5ad')
```

It's good practice to keep track of package version we used.

```python
sc.logging.print_versions()
# scanpy==1.4.4.post1
# anndata==0.6.22.post1
# umap==0.3.10
# numpy==1.17.2
# scipy==1.3.1
# pandas==0.25.1
# scikit-learn==0.21.3
# statsmodels==0.10.1
# python-igraph==0.7.1
# louvain==0.6.1
```

## Import to Seurat

Next,...

* we hop into R,
* set some parameters,
* load packages, and
* import the `.h5ad` file we just wrote to disk using the `ReadH5AD()` function from the Seurat package.

```r
options(width = 100)
set.seed(1234567)

library('dplyr')
library('Seurat')
library('monocle')
library('cerebroApp')

seurat <- ReadH5AD('scanpy/adata.h5ad')
```

Since factorized meta data isn't imported correctly, we need to fix that a little here.

```r
seurat@meta.data$sample <- factor(seurat@meta.data$sample, levels = c(0,1,2,3))
levels(seurat@meta.data$sample) <- c('uninfected','infected_6h','infected_8h','infected_10h')

seurat@meta.data$louvain <- factor(seurat@meta.data$louvain, levels = sort(unique(seurat@meta.data$louvain)))

seurat@meta.data$phase <- factor(seurat@meta.data$phase, levels = c(0,1,2))
levels(seurat@meta.data$phase) <- c('G1','G2M','S')
```

## Optional (but recommended) steps

We could already export this object and visualize the contained data in Cerebro.
However, data exploration in Cerebro would greatly benefit from additional data generated by the functions of cerebroApp.
What follows is a set of (mostly) optional steps.

### Add raw counts

First,...

* we load the raw transcript counts that we exported earlier,
* make sure the gene names match, and
* attach it to the Seurat object.

```r
raw_counts <- readr::read_tsv('scanpy/raw_counts.tsv', col_names = FALSE) %>%
  as.matrix() %>%
  t() %>%
  as('sparseMatrix')
colnames(raw_counts) <- readr::read_tsv('scanpy/raw_counts_cells.tsv', col_names = FALSE) %>% dplyr::pull(X1)
rownames(raw_counts) <- readr::read_tsv('scanpy/raw_counts_genes.tsv', col_names = FALSE) %>% dplyr::pull(X1)

identical(rownames(raw_counts), rownames(seurat@assays$RNA@data))
# TRUE

seurat@assays$RNA@counts <- raw_counts
```

### Add cluster tree

Let's also...

* factorize the cluster column in the meta data,
* assign the cluster labels as the active identity for each cell,
* build a cluster tree that represents the similarity between clusters, and
* create a dedicated `cluster` column in the meta data.

```r
Idents(seurat) <- 'louvain'
seurat <- BuildClusterTree(
  seurat,
  dims = 1:30,
  reorder = TRUE,
  reorder.numeric = TRUE
)
seurat[['cluster']] <- factor(
  as.character(seurat@meta.data$tree.ident),
  levels = sort(unique(seurat@meta.data$tree.ident))
)
seurat@meta.data$louvain <- NULL
seurat@meta.data$tree.ident <- NULL
```

### Add 3D projections

We also add 3D dimensional reductions made with tSNE and UMAP.

```r
seurat <- RunTSNE(
  seurat,
  reduction.name = 'tSNE_3D',
  reduction.key = 'tSNE3D_',
  dims = 1:30,
  dim.embed = 3,
  perplexity = 30,
  seed.use = 100
)

seurat <- RunUMAP(
  seurat,
  reduction.name = 'UMAP_3D',
  reduction.key = 'UMAP3D_',
  dims = 1:30,
  n.components = 3,
  seed.use = 100
)
```

### Add meta data

In order to later be able to understand how we did the analysis, we add some meta data to the `misc` slot of the Seurat object.

```r
seurat@misc$experiment <- list(
  experiment_name = 'GSE108041',
  organism = 'hg',
  date_of_analysis = Sys.Date()
)

seurat@misc$parameters <- list(
  gene_nomenclature = 'gene_name',
  discard_genes_expressed_in_fewer_cells_than = 10,
  keep_mitochondrial_genes = TRUE,
  variables_to_regress_out = 'nUMI',
  number_PCs = 30,
  tSNE_perplexity = 30,
  cluster_resolution = 0.5
)

seurat@misc$parameters$filtering <- list(
  UMI_min = 100,
  UMI_max = Inf,
  genes_min = 50,
  genes_max = Inf
)

seurat@misc$gene_lists$G2M_phase_genes <- cc.genes$g2m.genes
seurat@misc$gene_lists$S_phase_genes <- cc.genes$s.genes

seurat@misc$technical_info <- list(
  'R' = capture.output(devtools::session_info())
)
```

### Run cerebroApp functions

Using the functions provided by cerebroApp, we check the percentage of mitochondrial and ribosomal genes and, for every sample and cluster, we...

* get the 100 most expressed genes,
* identify marker genes (with the `FindAllMarkers` of Seurat),
* get enriched pathways in marker lists (using Enrichr),
* and perform gene set enrichment analysis (using GSVA).

```r
seurat <- cerebroApp::addPercentMtRibo(
  seurat,
  organism = 'hg',
  gene_nomenclature = 'name'
)

seurat <- cerebroApp::getMostExpressedGenes(
  seurat,
  column_sample = 'sample',
  column_cluster = 'cluster'
)

seurat <- cerebroApp::getMarkerGenes(
  seurat,
  organism = 'hg',
  column_sample = 'sample',
  column_cluster = 'cluster'
)

seurat <- cerebroApp::getEnrichedPathways(
  seurat,
  column_sample = 'sample',
  column_cluster = 'cluster',
  adj_p_cutoff = 0.01,
  max_terms = 100
)

seurat <- cerebroApp::performGeneSetEnrichmentAnalysis(
  seurat,
  GMT_file = 'c2.all.v7.0.symbols.gmt',
  column_sample = 'sample',
  column_cluster = 'cluster',
  thresh_p_val = 0.05,
  thresh_q_val = 0.1,
  verbose = FALSE
)
```

### Perform trajectory analysis

#### All cells

Next, we perform trajectory analysis of all cells with Monocle using the previously identified highly variable genes.
We extract the trajectory from the generated Monocle object with the `extractMonocleTrajectory()` function of cerebroApp and attach it to our Seurat object.

```r
monocle_all_cells <- newCellDataSet(
  seurat@assays$RNA@data,
  phenoData = new('AnnotatedDataFrame', data = seurat@meta.data),
  featureData = new('AnnotatedDataFrame', data = data.frame(
    gene_short_name = rownames(seurat@assays$RNA@counts),
    row.names = rownames(seurat@assays$RNA@counts))
  )
)

monocle_all_cells <- estimateSizeFactors(monocle_all_cells)
monocle_all_cells <- estimateDispersions(monocle_all_cells)
monocle_all_cells <- setOrderingFilter(monocle_all_cells, seurat@assays$RNA@var.features)
monocle_all_cells <- reduceDimension(monocle_all_cells, max_components = 2, method = 'DDRTree')
monocle_all_cells <- orderCells(monocle_all_cells)

seurat <- cerebroApp::extractMonocleTrajectory(monocle_all_cells, seurat, 'all_cells')
```

#### Only cells in G1 phase

Then, we do the same procedure again, however this time only with a subset of cells (those which are in G1 phase of the cell cycle).

```r
G1_cells <- which(seurat@meta.data$phase == 'G1')

monocle_subset_of_cells <- newCellDataSet(
  seurat@assays$RNA@data[,G1_cells],
  phenoData = new('AnnotatedDataFrame', data = seurat@meta.data[G1_cells,]),
  featureData = new('AnnotatedDataFrame', data = data.frame(
    gene_short_name = rownames(seurat@assays$RNA@counts),
    row.names = rownames(seurat@assays$RNA@counts))
  )
)

monocle_subset_of_cells <- estimateSizeFactors(monocle_subset_of_cells)
monocle_subset_of_cells <- estimateDispersions(monocle_subset_of_cells)
monocle_subset_of_cells <- setOrderingFilter(monocle_subset_of_cells, seurat@assays$RNA@var.features)
monocle_subset_of_cells <- reduceDimension(monocle_subset_of_cells, max_components = 2, method = 'DDRTree')
monocle_subset_of_cells <- orderCells(monocle_subset_of_cells)

seurat <- cerebroApp::extractMonocleTrajectory(monocle_subset_of_cells, seurat, 'subset_of_cells')
```

## Export to Cerebro

Finally, we use the `exportFromSeurat()` function of cerebroApp to export our Seurat object to a `.crb` file which can be loaded into Cerebro.

```r
cerebroApp::exportFromSeurat(
  seurat,
  experiment_name = 'GSE108041',
  file = paste0('scanpy/cerebro_GSE108041_', Sys.Date(), '.crb'),
  organism = 'hg',
  column_nUMI = 'nCount_RNA',
  column_nGene = 'nFeatures_RNA',
  column_cell_cycle_seurat = 'phase'
)
```

## Save Seurat object

Very last step: Save the Seurat object.

```r
saveRDS(seurat, 'scanpy/seurat.rds')
```
