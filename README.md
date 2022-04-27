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
```
qiime demux summarize --i-data paired-end-demux.qza --o-visualization paired-end-demux.qzv
qiime tools view paired-end-demux.qzv
```
### 2. Importamos la salida DADA a QIIME2
#### 2.1. Importamos las secuencias
```
qiime tools import \
      --input-path rep-seqs.fna \
      --type 'FeatureData[Sequence]' \
      --output-path rep-seqs.qza
```
#### 2.2. Importamos la tabla de frecuencias
Qiime no entiende los ficheros tabulares por lo que primero hay que pasarlo a biom
```
echo -n "#OTU Table" | cat - seqtab-nochim.txt > biom-table.txt
biom convert -i biom-table.txt -o table.biom --table-type="OTU table" --to-hdf5
```
Ahora podemos importar la tabla de frecuencias
```
qiime tools import \
      --input-path table.biom \
      --type 'FeatureTable[Frequency]' \
      --input-format BIOMV210Format \
      --output-path table.qza
```
#### 2.3. Importamos los estadísticos
```
echo -n "#OTU Table" | cat - stats.txt > biom-stats.txt
biom convert -i biom-table.txt -o stats.biom --table-type="OTU table" --to-hdf5
qiime tools import \
      --input-path stats.biom \
      --type 'FeatureTable[Frequency]' \
      --input-format BIOMV210Format \
      --output-path stats.qza
```
#### 2.3. Generamos los archivos que puedan ser visualizados en quiime2 view
```
qiime feature-table summarize \
      --i-table table.qza \
      --o-visualization table.qzv \
      --m-sample-metadata-file metadata
      
qiime feature-table summarize \ #comprobar
      --i-table stats.qza \
      --o-visualization stats.qzv \
      --m-sample-metadata-file metadata

qiime feature-table tabulate-seqs \
      --i-data rep-seqs.qza \
      --o-visualization rep-seqs.qzv

```
#### 2.4. Filtrado de 
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
# CAMBIAR LOS PARÁMETROS
```
qiime feature-classifier extract-reads \
      --i-sequences 85_otus.qza \
      --p-f-primer CCTACGGGNGGCWGCAG \ 
      --p-r-primer GACTACHVGGGTATCTAATCC \
      --p-min-length 100 \
      --p-max-length 490 \
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
                                          --p-n-jobs 2 \
                                          --output-dir taxa

qiime taxa barplot --i-table table.qza \
                   --i-taxonomy taxa/classification.qza \
                   --m-metadata-file metadata \
                   --o-visualization taxa/taxa_barplot.qzv
```
También podemos realizarla agrupando por los replicados, para lo cual necesitaríamos el archivo metadata_sample
```
qiime feature-table group --i-table table_filt.qza \
                          --p-axis sample \
                          --p-mode sum \
                          --m-metadata-file metadata \
                          --m-metadata-column Day_Temp \
                          --o-grouped-table table_sample_mio.qza

qiime feature-table summarize \
      --i-table table_sample.qza \
      --o-visualization table_sample_mio.qzv \
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
                                  
qiime diversity beta-group-significance --i-distance-matrix diversity/weighted_unifrac_distance_matrix.qza \
                                        --m-metadata-file metadata \
                                        --m-metadata-column Day_Temp \
                                        --o-visualization diversity/weighted_unifrac_Day_Temp_significance.qzv \
                                        --p-method permanova \
                                        --p-pairwise
```
