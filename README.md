[![Build Status](https://api.travis-ci.org/timothyjamesbecker/FusorSV.svg)](https://travis-ci.com/timothyjamesbecker/FusorSV) ![GitHub All Releases](https://img.shields.io/github/downloads/timothyjamesbecker/fusorsv/total.svg)  [![DOI](https://zenodo.org/badge/68958501.svg)](https://zenodo.org/badge/latestdoi/68958501) [![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)<br>
![Alt text](images/fusor-logo.jpg?raw=true "fusor")
# FusorSV
(c) 2016-2018 Timothy Becker, paper citation: https://doi.org/10.1186/s13059-018-1404-6 <br> <br>
A Data Fusion Method for Multi Source (VCF4.0+) Structural Variation Analysis. <br>
Takes as input a callset or group of VCF calls made by several SV callers for one sample and applies prior knowledge encoded in a fusion model to merge together the results that provide the best merging of the input files.  The resulting VCF file can be coordinate lifted and all samples can be clustered together based on reciprocal overlap.  The output includes for every FusorSV call the caller identifier and the specific call from that caller ids input VCF that contributed to the call in addition to an expectation estimate given the prior expectations on the agreement in that call.  Prior expectations are derived by using an alternate training mode that requires a true callset be given for each training observation that is used. <br> 

![Alt text](images/fusorSV_overview_v3_crop.jpg?raw=true "SVE") <br>
*logo and illustrations courtesy of Jane Cha <br>

## Requirements (docker)
docker toolbox (or engine) version 1.13.0+

```bash
docker pull timothyjamesbecker/fusorsv
```

## Requirements (non-docker)
2.7.6 < python < 2.7.12 <br> 
cython 0.23.4+ <br> 
0.9.0 < pysam < 0.9.2 <br> 
numpy 1.10.0+ <br>

### Options (non-docker)
bx-python 0.5.0 (optional for crossmap liftover) <br> 
mygene 3.0.0    (optional for gene annotations) <br> 

### Installation
#### (I) pip: install the requirements and then a fusorsv release:

```
pip --upgrade pip
pip install -Iv 'cython>=0.23.4'
pip install -Iv 'pysam>=0.9.0,<0.9.2'
pip install -Iv 'numpy>=1.10.0'

(Optional)
pip install -Iv 'bx-python>=0.5.0,<0.7.3'
pip install -Iv 'mygene>=3.0.0'

(Now FusorSV package)
pip install https://github.com/timothyjamesbecker/FusorSV/releases/download/0.1.2-beta/fusorsv-0.1.2.tar.gz
```

#### (II) Test the requirements and shared library:
```bash
FusorSV.py --test_libs
fusion_utils.so and bindings are functional!
```
A more indepth test can be done by downloading and decompressing the included release test data set:

```bash
wget https://github.com/timothyjamesbecker/FusorSV/releases/download/0.1.0-beta/meta_caller_RC1.tar.gz
tar -xzf meta_caller_RC1.tar.gz 
```
and then following the usage guide below.

## Usage (omit the first line when using non-docker)
```bash
docker run -v ~/data:/data -it timothyjamesbecker/fusorsv \
FusorSV.py \
-r /data/ref/human_g1k_v37_decoy.fa \
-m /data/human_g1k_v37_decoy.bed \
-i /data/meta_caller_RC1/ \
-f DEFAULT \
-o /data/meta_caller_RC1_result/ \
-M 0.5 \
-L DEFAULT \
-p 4
```
-r or --ref_path is the reference fasta file used to make SV calls needed to properly format the resulting merged VCF file so the VCF tools and IGV will be able to visualize and or operate correctly. Internal coordinate offsets are now auto generated if not provided. <br>
-m or --sv_mask is the BED3 format areas that will be excluded in analysis. This flag is now required for proper funcitonality as of version 0.1.1. A default svmask is included in the fusorsv data folder:human_g1k_v37_decoy_exlcude.bed<br>
-i or --in_dir is the input folder input folder of sample folders where each sample folder has the SV calls in VCF form as generated by the SVE docker image and workflow (https://github.com/timothyjamesbecker/SVE) as an example with samples names sample1 and sample2:

```bash
ls /data/vcfs/*

/data/vcfs/sample1:
sample1_S10.vcf sample1_S17.vcf sample1_S35.vcf sample1_S9.vcf sample1_S1.vcf sample1_S38.vcf
sample1_S11.vcf	sample1_S18.vcf	sample1_S4.vcf sample1_S0.vcf sample1_S14.vcf

/data/vcfs/sample2:
sample2_S10.vcf sample2_S17.vcf sample2_S35.vcf sample2_S9.vcf sample2_S1.vcf sample2_S38.vcf
sample2_S11.vcf	sample2_S18.vcf	sample2_S4.vcf sample2_S0.vcf sample2_S14.vcf

-i /data/vcfs/ would pass the folder of samples to FusorSV.py correctly in this case
``` 
-o or --out_dir is the path of the output directory that will be created and populated with the data partitions converted to internal Structural Variation Unit List (SVUL) form, the model that will be generated if in training mode udner models/*pickle.gz, the resulting merged VCF folder and performance metric files will be in the vcf and g1k folders (VCF is prefered and supports clustering and liftover):

```bash
ls -lh /data/meta_caller_RC1_result/
g1k
models
svul
tigra_ctg_map
vcf
visual
```
-f or --apply_fusion_model_path is a gzip compressed and serialized Python2.7 object file that was derived from a training mode as shown in the overview. The true callsets and SV callsets from every SV caller in every observation in training are characterized currently as:

#### Fusion Model Details

![Alt text](images/method_panel.jpg?raw=true "Fusion") <br>
*illustration courtesy of Jane Cha <br>

#### (a) partitioning scheme
The partitioning scheme is a set of instructions that guide the selection of SVUs inside the observations.  Currently these partitions select by the type of SV call such as INS, SUB, DEL, DUP, INV, TRA and then my the magnitude of the resulting change or SVU size.  These magnitudes do not have to be linear in construction or the same for every type and can be specified by altering the FusorSV.py sections or by using the FusorSV/data/bin_map.json to specify the magnitude boundries for every SV type.

#### (b) group expectation lower-bound
During training an estimate of the expectation of every callset group combination is calculated to the true callset in addition to the distance of every caller to each other.  Callers that use very similar strategies tend to have a small distance, while callers that use very different strategies tend to have a much larger distance.  callers that have a small distance to the true call set are preferred in the merging process.

#### (c) optimal filter point (alpha)
An optimal filter point for every partition is calculated by using all possible expectations (when the size of input callers is small) or by Expectation Maximization (EM for larger caller input sizes) to construct a proposal callset that allows all calls to pass that are above the current iteration's position p. 

#### (d) left and right breakpoint differentials
For every call that has more than r % of reciprocal overlap with a call in the true callset, the difference in bp distance is stored.  These breakpoint differentials can be sued to apply break point smoothing mechanisms to new data that prefers callers that have precise breakpoints over those that often are far away from a true breakpoint.

#### (e) number of calls made
The number of calls made can often be very difference between a true call set used in training and the calls that are generated under different circumstance.  For diagnostic purposes and reporting, these values are kept and tabulated in the FusorSV/visual folder.

-E or --stage_exclude_list is used to filter out calls made by a particuliar caller id.  For example if you have VCF files for caller id 4,9,10,11,17 but only want 4,9,17 you would use: -E 10,11 to exclude caller id 10 and 11 from training or application. The default exclude is 1,13,36.

-M or --cluster_overlap is used to apply optional clustering of the resulting individual per sample FusorSV VCF file, where a decimal is given r = [0.0,1.0] that is used to cluster those calls that have >= r amount of reciprocal overlap. In the resulting clustered VCF file: all_samples_genotypes.vcf each individual call still reports the SV caller that contributed, but now they are aggregated across the samples by the SV caller id.

-L or liftover is used to take the final call and apply a liftover criteria.  This uses the crossmap tool written by Liguo Wang http://crossmap.sourceforge.net with modifications to mapped versus unmapped criteria. DEFAULT option does a h19 to Hg38 liftover and will mark those calls that have a 1:1 mapping from hg19 to Hg38 in a new VCF file marked all_sampes_genotypes.mapped.vcf, while those calls that either have a 1:0 mapping meaning that the coordinate in hg19 is missing in Hg38, or when a coordinate in hg19 now has multiple coordinate regions in Hg38 as all_samples_genotypes.unmapped.vcf. Chain files can be obtained from http://hgdownload.cse.ucsc.edu/downloads.html 

-c or --chroms is a comma separated list that can be used to constrict the sequences used for analysis such as 1,2,3 ect

-p or --cpus is used to run more than one cpu or core during training.  This value is passed to 1/2 that number during the VCF generation phase allowing very large number of samples to be processed at the expense of a longer runtime.

### stage_map.json modifications: <br>

The syntax is stage_id (a unique integer) as the key and a SV Caller String as the value. The K:V pair "-1":"fusorSV" should be present. <br>

```bash
{
    "0":"True",
    "-1":"fusorSV",
    "1":"MetaSV",
    "4":"BreakDancer",
    "9":"cnMOPS",
    "10":"CNVnator",
    "11":"Delly",
    "13":"GATK",
    "14":"GenomeSTRiP",
    "17":"Hydra",
    "18":"Lumpy",
    "35":"BreakSeq",
    "36":"Pindel",
    "38":"Tigra"
}
```

## Large sample size Usage (omit the first line when using non-docker)
```bash
docker run -v ~/data:/data -it timothyjamesbecker/fusorsv \
FusorSV.py \
-r /data/ref/human_g1k_v37_decoy.fa \
-m /path_to_FusorSV/data/human_g1k_v37_decoy.bed \
-i /data/meta_caller_RC1_batch1/ \
-f DEFAULT \
-o /data/meta_caller_RC1_result/ \
--no_merge \
-p 4
```
which is then followed by a single merge step (omit the first line when using non-docker)
```bash
docker run -v ~/data:/data -it timothyjamesbecker/fusorsv \
FusorSV.py \
-r /data/ref/human_g1k_v37_decoy.fa \
-o /data/meta_caller_RC1_result/ \
--merge \
-M 0.5 \
-L DEFAULT
```


