# METAG_units_1_2. Sofía González Matatoros
## Fecha: 20/05/2022
## Reconstrucción del protocolo de QIIME2 
### 1. Importamos las muestras
```
conda activate qiime2-2020.11

qiime tools import --type 'SampleData[PairedEndSequencesWithQuality]' \
                   --input-path samplemanifest \
                   --output-path paired-end-demux.qza \
                   --input-format PairedEndFastqManifestPhred33
```
#### 1.1. Visualizamos un resumen del proceso de importación
> Hecho
```
qiime demux summarize --i-data paired-end-demux.qza --o-visualization paired-end-demux.qzv
qiime tools view paired-end-demux.qzv
```
### 2. Determinamos los ASV
> Hecho
```
qiime dada2 denoise-paired \
  --i-demultiplexed-seqs paired-end-demux.qza \
  -- trunc-len-f 240 \
  -- trunc-len-r 155 \
  -- trim-left-f 19 \
  -- trim-left-r 20 \
  -- n-threads 7 \
  -- n-reads-learn 1921748 \

```
#### 2.1. Generamos los archivos que puedan ser visualizados en quiime2 view
> Hecho
```
qiime feature-table summarize \
      --i-table table.qza \
      --o-visualization table.qzv \
      --m-sample-metadata-file metadata
      
qiime metadata tabulate \
  --m-input-file stats.qza \
  --o-visualization stats1.qzv

qiime feature-table tabulate-seqs \
      --i-data rep-seqs.qza \
      --o-visualization rep-seqs.qzv

```
#### 2.2. Filtrado de ASV de baja frecuencia

> comprobar 

```
qiime feature-table filter-features --i-table table.qza \
                                    --p-min-frequency 79 \
                                    --p-min-samples 1 \
                                    --o-filtered-table table.qza

qiime feature-table filter-seqs --i-data rep-seqs.qza \
                                --i-table table.qza \
                                --o-filtered-data rep-seqs.qza
```
### 3. Determinación de las distancias filogenéticas mediante MAFFT y FastTree
> Hecho
```
qiime phylogeny align-to-tree-mafft-fasttree \
                --i-sequences rep-seqs.qza \
                --o-alignment aligned-rep-seqs.qza \
                --o-masked-alignment masked-aligned-rep-seqs.qza \
                --o-tree unrooted-tree.qza \
                --o-rooted-tree rooted-tree.qza
```
### 4. Asignación taxonómica
#### 4.1. Importamos los ficheros necesarios para la asignación taxonómica
```
qiime tools import \
      --type 'FeatureData[Sequence]' \
      --input-path 85_otus.fasta \
      --output-path 85_otus.qza

qiime tools import \
     --type 'FeatureData[Taxonomy]' \
     --input-format HeaderlessTSVTaxonomyFormat \
     --input-path 85_otu_taxonomy.txt \
     --output-path ref-taxonomy.qza
```
#### 4.2. Extraemos las lecturas que pueden ser amplificadas por nuestros primers 
> Hecho
```
qiime feature-classifier extract-reads \
      --i-sequences 85_otus.qza \
      --p-f-primer GTGYCAGCMGCCGCGGTAA \ 
      --p-r-primer GGACTACHVGGGTWTCTAAT \
      --p-min-length 100 \
      --p-max-length 400 \
      --o-reads ref-seqs.qza
```
#### 4.3. Creamos el clasificador
> Hecho
```
qiime feature-classifier fit-classifier-naive-bayes \
      --i-reference-reads ref-seqs.qza \
      --i-reference-taxonomy ref-taxonomy.qza \
      --o-classifier classifier.qza
```
#### 4.4. Realizamos la asignación taxonómica
> hecho
```
qiime feature-classifier classify-sklearn --i-reads rep-seqs.qza \
                                          --i-classifier classifier.qza \
                                          --p-n-jobs 7 \
                                          --output-dir taxa

qiime taxa barplot --i-table table.qza \
                   --i-taxonomy taxa/classification.qza \
                   --m-metadata-file metadata \
                   --o-visualization taxa/taxa_barplot.qzv
```
También podemos realizarla agrupando por los replicados, para lo cual necesitaríamos el archivo metadata_sample
> Esto no sé cómo comprobarlo
```
qiime feature-table group --i-table table.qza \
                          --p-axis sample \
                          --p-mode sum \
                          --m-metadata-file metadata \
                          --m-metadata-column Day_Temp \
                          --o-grouped-table table-sample-mio.qza

qiime feature-table summarize \
      --i-table table_sample.qza \
      --o-visualization table-sample-mio.qzv \
      --m-sample-metadata-file metadata_sample

qiime taxa barplot --i-table table_sample.qza \
                   --i-taxonomy taxa/classification.qza \
                   --m-metadata-file metadata_sample \
                   --o-visualization taxa/taxa_sample_barplot.qzv
```
### 5. Estudio de la diversidad
```
qiime diversity alpha-rarefaction --i-table table.qza \
                                  --p-max-depth 123000 \
                                  --p-steps 20 \
                                  --i-phylogeny rooted-tree.qza \
                                  --m-metadata-file metadata \
                                  --o-visualization rarefaction_curves.qzv
                                  
qiime diversity core-metrics-phylogenetic --i-table table_filt.qza \
                                          --i-phylogeny rooted-tree.qza \
                                          --p-sampling-depth 27000 \
                                          --m-metadata-file metadata \
                                          --p-n-jobs-or-threads 2 \
                                          --output-dir diversity

qiime diversity beta-group-significance --i-distance-matrix diversity/weighted_unifrac_distance_matrix.qza \
                                        --m-metadata-file metadata \
                                        --m-metadata-column Day_Temp \
                                        --o-visualization diversity/weighted_unifrac_Day_Temp_significance.qzv \
                                        --p-method permanova \
                                        --p-pairwise
```
