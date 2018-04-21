```
████████╗██████╗ ██╗   ██╗██╗   ██╗ █████╗ ██████╗ ██╗
╚══██╔══╝██╔══██╗██║   ██║██║   ██║██╔══██╗██╔══██╗██║
   ██║   ██████╔╝██║   ██║██║   ██║███████║██████╔╝██║
   ██║   ██╔══██╗██║   ██║╚██╗ ██╔╝██╔══██║██╔══██╗██║
   ██║   ██║  ██║╚██████╔╝ ╚████╔╝ ██║  ██║██║  ██║██║
   ╚═╝   ╚═╝  ╚═╝ ╚═════╝   ╚═══╝  ╚═╝  ╚═╝╚═╝  ╚═╝╚═╝
```

Structural variant comparison tool for VCFs

Given benchmark and comparsion sets of SVs, calculate the sensitivity/specificity/f-measure.

Spiral Genetics, 2018


Installation
============

Truvari requires the following modules:

  $ pip install pyvcf Levenshtein swalign intervaltree progressbar2


Quick start
===========

  $ ./truvari.py -b base_calls.vcf -c compare_calls.vcf -o output_dir/


Outputs
=======

  * tp-call.vcf -- annotated true positive calls from the COMP
  * tp-base.vcf -- anotated true positive calls form the BASE
  * fn.vcf -- false negative calls from BASE
  * fp.vcf -- false positive calls from COMP
  * base-filter.vcf -- size filtered calls from BASE
  * call-filter.vcf -- size filtered calls from COMP
  * summary.txt -- json output of performance stats
  * log.txt -- run log
  * giab_report.txt -- (optional) Summary of GIAB benchmark calls. See below for details.


Methodology
===========

```
Input:
    BaseCall - Benchmark TruthSet of SVs
    CompCalls - Comparison SVs from another program
Build IntervalTree of CompCalls
For each BaseCall:
  Fetch CompCalls overlapping within *refdist*. 
    If typematch and LevDistRatio >= *pctsim* \
    and SizeRatio >= *pctsize* and PctRecOvl >= *pctovl*: 
      Add CompCall to list of Neighbors
  Sort list of Neighbors by TruScore ((2*sim + 1*size + 1*ovl) / 3.0)
  Take CompCall with highest TruScore and BaseCall as TPs
  Only use a CompCall once
  If no neighbors: BaseCall is FN
For each CompCall:
  If not used: mark as FP
```

Matching Parameters
--------------------
<table><tr><th>Parameter</th><th>Default</th><th>Definition</th>
<tr><td>refdist</td><td>500</td>
<td>Maximum distance comparison calls must be within from base call's start/end</td></tr>
<tr><td>pctsim</td><td>0.7</td>
<td>Levenshtein distance ratio between the REF or ALT sequence of base and comparison call.
Longer sequence of the two is used.</td></tr>
<tr><td>pctsize</td><td>0.7</td>
<td>Ratio of min(base_size, comp_size)/max(base_size, comp_size)</td></tr>
<tr><td>pctovl</td><td>0.0</td>
<td>Ratio of two calls' (overlapping bases)/(longest span)</td></tr>
<tr><td>typeignore</td><td>False</td>
<td>Types don't need to match to compare calls.</td>
</table>

Comparing VCFs without sequence resolved calls
----------------------------------------------

If the base or comp vcfs do not have sequence resolved calls, simply set `--pctsim=0` to turn off
sequence comparison.

Difference between --sizemin and --sizefilt
-------------------------------------------

`--sizemin` is the minimum size of a base call or comparison call to be considered.  

`--sizefilt` is the minimum size of a call to be added into the IntervalTree for searching. It should
be less than `sizemin` for edge case variants.

For example: `sizemin` is 50 and `sizefilt` is 30. A 50bp base call is 98% similar to a 49bp call at 
the same position.

These two calls should be considered matching. If we instead removed calls less than `sizemin`, we'd
incorrectly classify the 50bp base call as a false negative.

This does have the side effect of artificially inflating specificity. If that same 49bp call in the
above were below the similarity threshold, it would not be classified as a FP due to the `sizemin`
threshold. So we're giving the call a better chance to be useful and less chance to be detrimental
to final statistics.

Definition of annotations added to TP vcfs
--------------------------------------------
<table>
<tr><th>Anno                   </th><th>Definition</th></tr>
<tr><td>TruScore	       </td><td>Truvari score for similarity of match. `((2*sim + 1*size + 1*ovl) / 3.0)`</td></tr>
<tr><td>PctSeqSimilarity       </td><td>Pct sequence similarity between this variant and its closest match</td></tr>
<tr><td>PctSizeSimilarity      </td><td>Pct size similarity between this variant and it's closest match</td></tr>
<tr><td>PctRecOverlap          </td><td>Percent reciprocal overlap of the two calls' coordinates</td></tr>
<tr><td>StartDistance          </td><td>Distance of this call's start from matching  call's start</td></tr>
<tr><td>EndDistance            </td><td>Distance of this call's end from matching  call's end</td></tr>
<tr><td>SizeDiff               </td><td>Difference in size(basecall) and size(compcall)</td></tr>
<tr><td>NumNeighbors           </td><td>Number of comparison calls that were in the neighborhood (REFDIST) of the base call</td></tr>
<tr><td>NumThresholdNeighbors  </td><td>Number of comparison calls that passed threshold matching of the base call</td></tr>
</table>

NumNeighbors and NumThresholdNeighbors are also added to the FN vcf.

Using the GIAB Report
---------------------

When running against the GIAB SV v0.5 benchmark (link below), you can create a detailed report of 
calls summarized by the GIAB VCF's SVTYPE, SVLEN, Technology, and Repeat annotations.

To create this report.

1. Run Truvari with the flag `--giabreport`.
2. In your output directory, you will find a file named `giab_report.txt`.
3. Next, make a copy of the 
[Truvari Report Template Google Sheet](https://docs.google.com/spreadsheets/d/1T3EdpyLO1Kq-bJ8SDatqJ5nP_wwFKCrH0qhxorvTVd4/edit?usp=sharing).
4. Finally, paste ALL of the information inside `giab_report.txt` into the "RawData" tab. Be careful not 
to alter the report text in any way. If successul, the "Formatted" tab you will have a fully formated report.

This currently only works with GIAB SV v0.5. Work will need to be done to ensure Truvari can parse future 
GIAB SV releases.

ftp://ftp-trace.ncbi.nlm.nih.gov/giab/ftp/data/AshkenazimTrio/analysis/NIST_UnionSVs_12122017/

VCF Header contigs & Exclude Bed 
-----------

Before parsing any files, the set of contigs in VCF headers of the comparison calls are subtracted from the
base calls set of contigs. This gives us a set of contigs to exclude from this analysis. Any comparison call 
on a contig not present in the base calls' VCF is not examined for TP/FP/FN.

An `--excludebed` file may be provided to remove calls in regions from comparison. This is functionally 
equivalent to running `bedtools subtract -A -a calls.vcf -b exclude.bed`. 

It is important to note that any size/gt filtered numbers in the summary do not account for these excluded 
regions.


