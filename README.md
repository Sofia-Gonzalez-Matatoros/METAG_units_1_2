# METAG_units_1_2. Sofía González Matatoros

## Reconstrucción del protocolo de QIIME2
### 1. Importamos las muestras
```
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
#### 2.3. Generamos los archivos que puedan ser visualizados en quiime2 view
```
qiime feature-table summarize \
      --i-table table.qza \
      --o-visualization table.qzv \
      --m-sample-metadata-file metadata

qiime feature-table tabulate-seqs \
      --i-data rep-seqs.qza \
      --o-visualization rep-seqs.qzv
```
