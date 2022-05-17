# METAG_units_1_2. Reconstrucción del protocolo de QIIME2
Alumna: Sofía González Matatoros

Fecha: 20/05/2022

Link al repositorio de Github: https://github.com/Sofia-Gonzalez-Matatoros/METAG_units_1_2.git

### 1. Importamos las muestras
```
conda activate qiime2-2020.11

qiime tools import --type 'SampleData[PairedEndSequencesWithQuality]' \
                   --input-path samplemanifest \
                   --output-path paired-end-demux.qza \
                   --input-format PairedEndFastqManifestPhred33
```
#### 1.1. Visualizamos un resumen del proceso de importación

```
qiime demux summarize --i-data paired-end-demux.qza --o-visualization paired-end-demux.qzv
qiime tools view paired-end-demux.qzv
```
### 2. Determinamos los ASV

```
qiime dada2 denoise-paired \
  --i-demultiplexed-seqs paired-end-demux.qza \
  --p-trunc-len-f 240 \
  --p-trunc-len-r 155 \
  --p-trim-left-f 19 \
  --p-trim-left-r 20 \
  --p-n-threads 7 \
  --p-n-reads-learn 1921748 \
  --o-representative-sequences rep-seqs.qza \
  --o-table table.qza \
  --o-denoising-stats stats.qza 
```
#### 2.1. Generamos los archivos que puedan ser visualizados en quiime2 view

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
### 3. Determinación de las distancias filogenéticas mediante MAFFT y FastTree

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

```
qiime feature-classifier fit-classifier-naive-bayes \
      --i-reference-reads ref-seqs.qza \
      --i-reference-taxonomy ref-taxonomy.qza \
      --o-classifier classifier.qza
```
#### 4.4. Realizamos la asignación taxonómica

```
qiime feature-classifier classify-sklearn --i-reads rep-seqs.qza \
                                          --i-classifier classifier.qza \
                                          --p-n-jobs 7 \
                                          --output-dir taxa

qiime feature-table group --i-table table.qza \
                          --p-axis sample \
                          --p-mode sum \
                          --m-metadata-file metadata \
                          --m-metadata-column Day_Temp \
                          --o-grouped-table table_sample.qza

qiime feature-table summarize \
      --i-table table_sample.qza \
      --o-visualization table_sample.qzv \
      --m-sample-metadata-file metadata_sample

qiime taxa barplot --i-table table_sample.qza \
                   --i-taxonomy taxa/classification.qza \
                   --m-metadata-file metadata_sample \
                   --o-visualization taxa/taxa_sample_barplot.qzv
```
### 5. Estudio de la diversidad
#### 5.1. Carpeta diversity

```
qiime diversity alpha-rarefaction --i-table table.qza \
                                  --p-max-depth 288000 \
                                  --p-steps 100 \
                                  --i-phylogeny rooted-tree.qza \
                                  --m-metadata-file metadata \
                                  --o-visualization rarefaction_curves.qzv
                                  
qiime diversity core-metrics-phylogenetic --i-table table.qza \
                                          --i-phylogeny rooted-tree.qza \
                                          --p-sampling-depth 85000 \
                                          --m-metadata-file metadata \
                                          --p-n-jobs-or-threads 2 \
                                          --output-dir diversity

qiime diversity beta-group-significance --i-distance-matrix diversity/bray_curtis_distance_matrix.qza \
                                        --m-metadata-file metadata \
                                        --m-metadata-column Day_Temp \
                                        --o-visualization diversity/bray_curtis_day_temp_significance.qzv \
                                        --p-method permanova \
                                        --p-pairwise True

qiime diversity beta-group-significance --i-distance-matrix diversity/unweighted_unifrac_distance_matrix.qza \
                                        --m-metadata-file metadata \
                                        --m-metadata-column Day_Temp \
                                        --p-pairwise \
                                        --o-visualization diversity/weighted_unifrac_day_temp_significance.qzv 
                                        

```
#### 5.2. Carpeta diversity_sample

```
qiime diversity core-metrics-phylogenetic --i-table table_sample.qza \
                                          --i-phylogeny rooted-tree.qza \
                                          --p-sampling-depth 101046 \
                                          --m-metadata-file metadata \
                                          --p-n-jobs-or-threads 2 \
                                          --output-dir diversity_sample

```
### 6. Gráfico DESeq
Tras ejecutar el protocolo DESeq2 se obtuvo la siguiente gráfica

![Screenshot](https://github.com/Sofia-Gonzalez-Matatoros/METAG_units_1_2/blob/2881a4912cc424afd59776abfca876296fca3d3b/Captura%20de%20pantalla%20de%202022-05-13%2019-34-39.png)

Este protocolo realiza análisis de expresión diferencial de RNAs en las muestras de ratón, de forma que en el eje de ordenadas observamos la magnitud del cambio de expresión (log2FC), en el de abscisas aparecen las familias microbianas identificadas, y los colores de los puntos hacen referencia a los filos. 

A rasgos generales, se puede apreciar que la cantidad de RNA perteneciente a individuos del filo Firmicutes ha aumentado, especialmente los de la familia Lachnospiraceae. Mientras que este ha disminuido en el de Bacteroidetes, siendo S24-7 la familia más representativa. Sin embargo, también cabe destacar que concretamente en esta familia que hay un par de excepciones a este decrecimiento.

Una posible interpretación de esta gráfica es que el frío genera cambios en la composición de la microbiota, de manera que aumenta el número de individuos pertenecientes al filo Firmicutes y disminuyen los pertenecientes al filo Bacteroidetes, por ello, la expresión de RNA relativo a esos filos aumenta y disminuye respectivamente.


