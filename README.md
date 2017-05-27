# FusorSV
(c) 2016-2017 Timothy Becker  <br> <br>
A Data Fusion Method for Multi Source (VCF4.0+) Structural Variation Analysis. <br>
Takes as input a callset or group of VCF calls made by several SV callers for one sample and applies prior knowledge encoded in a fusion model to merge together the results that provide the best mergeing of the input files.  The resulting VCF file can be coordinate lifted and all samples can be clustered together based on reciprocal overlap.  The output includes for every FusorSV call the caller identifier and the specific call from that caller ids input VCF that contributed to the call in addition to an expectation estimate given the prior expectations on the agreement in that call.  Prior expectations are dervided by using an laternate training mode that requires a tru callset be given for each training observation that is used. <br> 

![Alt text](images/overview_method_panelv2.jpg?raw=true "SVE") <br>
*illustration courtesy of Jane Cha <br>

## Requirements (docker)
docker toolbox (or engine) version 1.13.0+

```bash
docker pull timothyjamesbecker/fusorsv
```

## Requirements/Options (non-docker)
python 2.7.10 <br> 
cython 0.23.4 <br> 
pysam 0.9.1 <br> 
numpy 1.10.0 <br> 
scipy 1.16.0    (will be factored out in the next release) <br> 
HTSeq 0.6.0     (will be factored out in the next release) <br> 
bx-python 0.5.0 (will be optional in the next release) <br> 
mygene 3.0.0    (will be optional in the next release) <br> 

### Building (automated pip will be available very soon)
#### (1) If using pip: install the requirements:

```
pip --upgrade pip
pip install 'cython>=0.23.4,<0.25.3'
pip install 'pysam>=0.9.0,<0.9.2'
pip install 'numpy>=1.10.0,<1.11.0'
pip install 'scipy>=1.16.0,<1.18.0'
pip install 'HTSeq>=0.6.0,<0.7.0'
pip install 'bx-python>=0.5.0,<0.7.3'
pip install 'mygene>=3.0.0'
```

#### (2) Clone the git repo or download a release:

```bash
git clone https://github.com/timothyjamesbecker/FusorSV.git
```

#### (3) Build the shared object fusion_utils.so and its bindings:

```bash
cd FusorSV
python fusion_utils.py build_ext --inplace
```

(4) Test the requirements and shared library:

```bash
FusorSV.py --test_libs
```

## Usage (omit the first line when using non-docker)

```bash
docker run -v ~/data:/data -it timothyjamesbecker/fusorsv\
/software/FusorSV/FusorSV.py\
-r /data/ref/human_g1k_v37_decoy.fa\
-i /data/meta_caller_RC1/\
-f /data/human_g1k_v37_decoy.P3.pickle.gz\
-o /data/meta_caller_RC1_result/\
-M 0.5\
-L DEFAULT
-P 4
```
-r or --ref_path is the reference fast used to make SV calls and needed to properly format the resulting merged VCF file so the VCF tools and IGV will be able to visualize and or operate correctly. <br>
-i or --in_dir is the input folder input folder of sample folders where each sample folder has the SV calls in VCF form as generated by the SVE docker image and workflow (https://github.com/timothyjamesbecker/SVE) as an example with samples names sampl1 and sample2:

```bash
ls /data/vcfs/*

/data/sample1:
sample1_S10.vcf sample1_S17.vcf sample1_S35.vcf sample1_S9.vcf sample1_S1.vcf sample1_S38.vcf
sample1_S11.vcf	sample1_S18.vcf	sample1_S4.vcf sample1_S0.vcf sample1_S14.vcf

/data/sample2:
sample2_S10.vcf sample2_S17.vcf sample2_S35.vcf sample2_S9.vcf sample2_S1.vcf sample2_S38.vcf
sample2_S11.vcf	sample2_S18.vcf	sample2_S4.vcf sample2_S0.vcf sample2_S14.vcf
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
-f or --apply_fusion_model_path is a gzip compressed and serialized Python2.7 object file that was derived from a training mode as shown in the overview. The true callsets and SV callsets from every SV caller in every observation in traing are chareterized currently as:

#### Fusion Model Details

![Alt text](images/method_panel.jpg?raw=true "Fusion") <br>
*illustration courtesy of Jane Cha <br>

#### (a) partitioning scheme
The partitioning scheme is a set of intructions that guide the selection of SVUs inside the obersvations.  Currently these partitions select by the type of SV call such as INS, SUB, DEL, DUP, INV, TRA and then my the magintude of the resulting change or SVU size.  These magitudes do not have to be linear in constrcution or the same for every type and can be specified by altering the FusorSV.py sections or by using the FusorSV/data/bin_map.json to specifiy the magitude boudries for every SV type.

#### (b) group expectation lower-bound
During training an estimate of the epxectation of every callset group combination is calculated to the true callset in addition to the distance of every caller to each other.  Callers that use very similair strategies tend to have a small distance, while callers that use very different stratgeies tend to have a much larger distance.  callers that have a small distance to the true call set are prefered in the merging process.

#### (c) optimal filter point (alpha)
An optimal filter point for every partition is calculated by using all possible expectations (when the size of input callers is small) or by Expectation Maximization (EM for larger caller input sizes) to construct a proposal callset that allows all calls to pass that are above the current iteration's position p. 

#### (d) left and right breakpoint differentials
For every call that has more than r % of reciprocal overlap with a call in the tru callset, the difference in bp distance is stored.  These breakpoint differentials can be sued to apply break point smoothing mechanisms to new data that prefers callers that have precise breakpoints over those that offent are far away from a true breakpoint.

#### (e) number of calls made
The number of calls made can often be very difference between a true call set used in training and the calls that are generated under different circumstance.  For diagnositic purposes and reporting, these values are kept and tabulated in the FusorSV/visual folder.

-M or --cluster_overlap is used to apply optional clustering of the resulting individual per sample FusorSV VCF file, where a decimal is given r = [0.0,1.0] that is used to cluster those calls that have >= r amount of reciprocal overlap. In the resulting clustered VCF file: all_samples_genotypes.vcf each individual call still reports the SV caller that contributed, but now they are agregated accross the sames by the SV caller id.

-L or liftover is used to take the final call and apply a liftover cirtera.  The DEFAULT option does a h19 to Hg38 liftover and will mark those calls that have a 1:1 mapping from hg19 to Hg38 in a new VCF file marked all_sampes_genotypes.mapped.vcf, while those calls that either have a 1:0 mapping meaning that the coordinate in hg19 is missing in Hg38, or when a coordinate in hg19 now has multiple coordinate regions in Hg38 as all_samples_genotypes.unmapped.vcf

-c or --chroms is a comma seperated list that can be used to constrict the sequences used for analtsis such as 1,2,3 ect

-p or --cpus is used to run more than one cpu or core during training.  This value is passed to 1/2 that number during the VCF generation phase allowing very large number of samples to be processed at the expense of a longer runtime.

## Testing
With each release the current fusion model file will be included with some documentation with regaurd to the SV callers supported.  This effort is in conjunction with the further development of the SVE repository (https://github.com/timothyjamesbecker/SVE).  To test your installation fully: 

#### (a) download the metacaller_RC1.tar.gz and decompress it:

```bash
wget URL/meta_caller_RC1.tar.gz && tar -xzf meta_caller_RC1.tar.gz
```

#### (b) download the fusion model human_g1k_v37_decoy.P3.pickle.gz (it does not have to be uncompressed)
```bash
wget URL/human_g1k_v37_decoy.P3.pickle.gz
```

#### (c) run your installation
Run the command form the usage section above and you can check your results to those already computed meta_caller_RC1_applied_results:

```bash
wget URL/meta_caller_RC1_applied_result.tar.gz
```

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


