# GWAS of GRLS data

## 1. Setup
## 1.1. Install conda
We are using [conda](https://conda.io/projects/conda/en/stable/index.html) to install softaware needed to run this tutorial. Here are the steps we used to install conda on a **64-bit** computer with a **Linux** system using **Miniconda**. For other operating systems, you can find detailed instructions [here](https://conda.io/projects/conda/en/stable/user-guide/install/index.html)

```
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```

-  Follow the prompts on the installer screens and accept the defaults.
-  Restart the terminal

## 1.2. Create new environment and install softaware packages

```
conda create -n grGWAS
conda activate grGWAS
conda install -c bioconda plink
conda install -c bioconda plink2
conda install -c bioconda bcftools
conda install -c bioconda gcta
conda install -c conda-forge r-base=3.6.3
```
<br>

## 2. Input files
## 2.1. Genotyping data
1.  Affymetrix (thermofisher) Axiom Canine HD Array sets A and B were used to genotype the GRLS dogs. In this tutorial, we have the GRLS genotyping data of each array in a binary PLINK file format. You may use the [PLINK documentation](https://www.cog-genomics.org/plink/1.9/input#bed) to read more about this file format. The [genotyping analysis notes]() has detailed information on the bioinformatic pipeline used for genotyping and all notes that should be considered before any further analysis.
    - The files of array A have the prefix "output/setA/export_plink/AxiomGT1.bin"
    - The files of array B have the prefix "output/setB/export_plink/AxiomGT1.bin".
    ---
    **_Note:_** These PLINK files have the gender as predicted by the Axiom genotyping analysis tools. Check the [genotyping analysis notes]() for details
    
    ---

2.  A text file that maps between sample IDs in the genotyping files, biological sample IDs, and the public IDs used in phenotype data files. Moreover, the file has gender information of the dogs as reported by their owners: `map_id_sex.tab`
   
## 2.2. Phenotype data
Morris Animal Foundation [Data Commons](https://datacommons.morrisanimalfoundation.org/) provides open access to most of the data collected by the Golden Retriever Lifetime Study. An overview description of the study data can be found [here](https://datacommons.morrisanimalfoundation.org/node/221). To download data tables, you need to [register](https://datacommons.morrisanimalfoundation.org/user/login?destination=/node/1) at the Data Commons.

In this tutorial, we are using the [Conditions - Neoplasia](https://datacommons.morrisanimalfoundation.org/artisanal_dataset/71) dataset as an example.

<br>


## 3. QC and pre-processing of genotyping data
## 3.1. replicate SNPs  
Both arrays has a number of replicate SNPs which are usefull for QC but also require special attention for proper merging of the files of both arrays. We are using a QC merging mode of PLINK to identify mismatching nonmissing calls between the 2 arrays

```
plink --bfile output/setB/export_plink/AxiomGT1.bin --chr-set 38 no-xy --allow-no-sex --allow-extra-chr \
      --bmerge output/setA/export_plink/AxiomGT1.bin --merge-mode 7 \
      --output-chr 'chrM' --out AxiomGT1.mismatching7
```

Here is the output of this command


> 3339 samples loaded from output/setB/export_plink/AxiomGT1.bin.fam. <br> 
3355 samples to be merged from output/setA/export_plink/AxiomGT1.bin.fam. <br> 
Of these, 17 are new, while 3338 are present in the base dataset. <br> 
547232 markers loaded from output/setB/export_plink/AxiomGT1.bin.bim. <br> 
397685 markers to be merged from output/setA/export_plink/AxiomGT1.bin.bim. <br> 
Of these, 368215 are new, while 29470 are present in the base dataset. <br> 
Warning: Variants 'Affx-206088448' and 'Affx-205939940' have the same position. <br> 
Warning: Variants 'Affx-205859745' and 'Affx-205344060' have the same position. <br> 
Warning: Variants 'Affx-206706550' and 'Affx-205359377' have the same position. <br> 
1460 more same-position warnings: see log file. <br> 
Performing 1-pass diff (mode 7), writing results to AxiomGT1.mismatching7.diff <br> 
98370860 overlapping calls, 97897061 nonmissing in both filesets. <br> 
97593330 concordant, for a concordance rate of 0.996897. <br> 


The concordance rate of nonmissing genotypes is pretty good (99.7%). However, we still have to handle two issues: 

1.  **Mismatching genotypes:** Let us have a closer look on the mismatching genotypes to see if they are uniformly distributed among the samples or not
    ```
    tail -n+2 AxiomGT1.mismatching7.diff | awk '{print $2,$3}' | sort | uniq -c | sort -k1,1nr > AxiomGT1.mismatching7.diff.samples
    ```

    It seems that two samples show exceptional higher rate of mismatching (`GRLS S007258` and `GRLS S019740` have 11676 and 11440 mistmatches respectively. The latter sample is the only sample that was predicted to be female on Array A and male on array B according to the genotyping analysis notes). This likely indicates that these samples had something wrong. According to the the genotyping analysis notes, swapping the 2 samples on array B did not fix the issue. Therefore, we will exclude both samples from further analysis during the merging step. 

2.  **Variants having the same position:** The mismatching analysis reveals 1463 markrs that has the same position on bith arrays but with different SNP IDs. Futher digging in the array annotation showed that most of these markers are idententical with minor differences in the flanking sequences. To avoid genotyping errors, we will exclude these SNPs from array B.

    ```
    grep "Warning: Variants .* have the same position" AxiomGT1.mismatching7.log | awk -F"'" 'BEGIN{OFS="\n";}{print $2,$4}' > same_pos.arrB.lst
    plink --bfile output/setB/export_plink/AxiomGT1.bin --chr-set 38 no-xy --allow-no-sex --allow-extra-chr \
      --exclude same_pos.arrB.lst \
      --make-bed --output-chr 'chrM' --out output/setB/export_plink/AxiomGT1.bin_noSamePos
    ```
## 3.2. Merging of Array sets A and B Genotyping data
Now, we will merge the genotyping data of both arrays. For SNPs shared between the arrays, the genotypes of array A will overwrite the nonmissing calls in array B. Also, we will **exclude the 2 samples with higher rates of non-concordance**

```
echo "GRLS S007258|GRLS S019740" | tr '|' '\n' > swap_samples.lst
plink --bfile output/setB/export_plink/AxiomGT1.bin_noSamePos --chr-set 38 no-xy --allow-no-sex --allow-extra-chr \
      --bmerge output/setA/export_plink/AxiomGT1.bin --merge-mode 3 \
      --remove swap_samples.lst \
      --make-bed --output-chr 'chrM' --out AxiomGT1v2.comp_merge
```

The output of the merging command gives some useful stastics

> ... <br>
Warning: 1355 het. haploid genotypes present (see AxiomGT1v2.comp_merge.hh ); <br>
Warning: Nonmissing nonmale Y chromosome genotype(s) present; <br>
... <br>
Total genotyping rate in remaining samples is 0.994631. <br>
913984 variants and 3354 samples pass filters and QC. <br>
...

Looking at `AxiomGT1v2.comp_merge.hh` shows that all the 1355 heterozygous haploid genotypes belong to sample `GRLS S005865`. According to the genotyping analysis notes, this sample is reported male in metadata as well as by the genotyping algorithm on Array B but the algorithm failed to predict its gender on array A.


## 3.3. Identification and removal of duplicate samples
Plink2 has an efficient function to calculate the KING-robust knickship estimator and filter duplicate samples as well. Duplicate samples have kinship coefficients ~0.5, first-degree relations (parent-child, full siblings) correspond to ~0.25, second-degree relations correspond to ~0.125, etc. Here, we are using the conventional cufoff ~0.354 (the geometric mean of 0.5 and 0.25) to identify and filter duplicate samples

```
plink2 --bfile AxiomGT1v2.comp_merge --chr-set 38 no-xy --allow-no-sex --allow-extra-chr \
       --king-cutoff 0.354 \
       --out AxiomGT1v2.comp_merge.nodup
```

Now, let us compare the files selected to be removed by Plink2 knickship versus the list of samples planned to run in duplicates. This code will identify all the samples planned to run in duplicates then find those remaining after exlcusion of the samples selected by the Plink2 knickship filter. We are expecting one replicate to remain from each group of replicate samples 
```
awk 'BEGIN{FS=OFS="\t"}FNR==NR{a[$3]+=1;next}/^Family_ID/{print}{if(a[$3]>1)print}' map_id_sex.tab map_id_sex.tab > dup_samples.tab
awk 'BEGIN{FS=OFS="\t"}FNR==NR{a[$1 FS $2]=1;next}{if(!a[$1 FS $2])print $0}' AxiomGT1v2.comp_merge.nodup.king.cutoff.out.id dup_samples.tab > dup_samples_remain.tab
tail -n+2 dup_samples_remain.tab | cut -f3 | sort | uniq -c | sort -k1,1nr
```
Hear is the list of the remaining duplicates and their counts
> 2 &nbsp;&nbsp;&nbsp; 094-027376 <br>
  1 &nbsp;&nbsp;&nbsp; 094-002188 <br>
  1 &nbsp;&nbsp;&nbsp; 094-002396 <br>
  1 &nbsp;&nbsp;&nbsp; 094-002995 <br>
  . <br>
  . <br>


It seems that 2 duplicates of sample `094-027376` are still remaining! Let's try to calculate their KING knickship to see how similar they are:
```
grep 094-027376 dup_samples.tab > failed_deDup.lst
plink2 --bfile AxiomGT1v2.comp_merge --chr-set 38 no-xy --allow-no-sex --allow-extra-chr \
       --keep failed_deDup.lst --make-king-table \
       --out AxiomGT1v2.comp_merge.failed_deDup
cat AxiomGT1v2.comp_merge.failed_deDup.kin0
```
Here is the output. 
> #FID1 &nbsp;&nbsp;&nbsp; ID1 &nbsp;&nbsp;&nbsp; FID2 &nbsp;&nbsp;&nbsp; ID2 &nbsp;&nbsp;&nbsp; NSNP &nbsp;&nbsp;&nbsp; HETHET &nbsp;&nbsp;&nbsp; IBS0 &nbsp;&nbsp;&nbsp; KINSHIP <br>
GRLS &nbsp;&nbsp;&nbsp; S027376_2 &nbsp;&nbsp;&nbsp; GRLS &nbsp;&nbsp;&nbsp; S027376_1 &nbsp;&nbsp;&nbsp; 879956 &nbsp;&nbsp;&nbsp; 0.0645396 &nbsp;&nbsp;&nbsp; 0.0414475 &nbsp;&nbsp;&nbsp; -0.0559478 <br>

It is obvious that these two samples are unlrelated. Therefore, we will exclude both of them with the duplicate samples selected by the Plink2 knickship filter. These are 120 samples in total so we will end up having 3234 samples in our output PLINK file
```
plink2 --bfile AxiomGT1v2.comp_merge --chr-set 38 no-xy --allow-no-sex --allow-extra-chr \
       --remove <(cat AxiomGT1v2.comp_merge.nodup.king.cutoff.out.id failed_deDup.lst) \
       --make-bed --output-chr 'chrM' --out AxiomGT1v2.comp_merge.deDup
```


## 3.4. Check for gender accuracy & remove samples with wrong gender identities 
According to the [genotyping analysis notes](), there are two samples `GRLS S019740` and `GRLS S005865` that had discordant computed gender on the two arrays. The former was removed already because of the high rate of mismatching genotypes between the 2 arrays (see **3.1. replicate SNPs**). The latter showed high rate heterozygous haploid genotypes despite being reported as male in metadata (see **3.2. Merging of Array sets**). We will discard this sample from further analysis.

Moreover, there are additional 9 samples with concordant gender on both arrays but different from the metadata. We can reproduce this information by known gender in metadata versus the computed gender based on genotyping data
```
echo "Family_ID Individual_ID computed_sex metadata_sex" | tr ' ' '\t' > gender_conflict.lst
awk 'BEGIN{FS=OFS="\t"}FNR==NR{a[$1 FS $2]=$5;next}{if(a[$1 FS $2] && a[$1 FS $2]!=$5)print $1,$2,$5,a[$1 FS $2]}' map_id_sex.tab AxiomGT1v2.comp_merge.deDup.fam >> gender_conflict.lst
```

Wrong gender could be a mistake in the metadata or an indication for sample swap. Therefore, it is safer to exclude these samples from our genotyping data
```
echo "GRLS S005865" | tr ' ' '\t' >> gender_conflict.lst
plink2 --bfile AxiomGT1v2.comp_merge.deDup --chr-set 38 no-xy --allow-no-sex --allow-extra-chr \
       --remove gender_conflict.lst \
       --make-bed --output-chr 'chrM' --out AxiomGT1v2.comp_merge.deDup.sexConfirm
```
The output of our last PLINK command indeicate that our final file has 3224 samples (1615 females, 1609 males; 3224 founders)


## 3.5. Update sample IDs in the genotyping files to match the phenotyping files
The data tables at Morris Animal Foundation [Data Commons](https://datacommons.morrisanimalfoundation.org/) are using grls_ids (e.g. 094-000019) or public_ids (e.g. grlsH764T844). In this step, we will upadate the sample IDs in our genotyping files to match the grls_ids using the `map_id_sex.tab` file that maps between different types of IDs
```
tail -n+2 map_id_sex.tab | awk 'BEGIN{FS=OFS="\t"}{print $1,$2,$1,$3}' > grls_id.update.lst 
plink2 --bfile AxiomGT1v2.comp_merge.deDup.sexConfirm --chr-set 38 no-xy --allow-no-sex --allow-extra-chr \
       --update-ids grls_id.update.lst \
       --make-bed --output-chr 'chrM' --out AxiomGT1v2.comp_merge.deDup.sexConfirm.grls_ids
```

We can also do the same to upadate the sample IDs in our genotyping files to match the public_ids
```
tail -n+2 map_id_sex.tab | awk 'BEGIN{FS=OFS="\t"}{print $1,$2,$1,$4}' > public_ids.update.lst 
plink2 --bfile AxiomGT1v2.comp_merge.deDup.sexConfirm --chr-set 38 no-xy --allow-no-sex --allow-extra-chr \
       --update-ids public_ids.update.lst \
       --make-bed --output-chr 'chrM' --out AxiomGT1v2.comp_merge.deDup.sexConfirm.public_ids
```

