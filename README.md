# MindTheGap 

| **Linux** | **Mac OSX** |
|-----------|-------------|
[![Build Status](https://ci.inria.fr/gatb-core/view/MindTheGap/job/tool-mindthegap-build-debian7-64bits-gcc-4.7/badge/icon)](https://ci.inria.fr/gatb-core/view/MindTheGap/job/tool-mindthegap-build-debian7-64bits-gcc-4.7/) | [![Build Status](https://ci.inria.fr/gatb-core/view/MindTheGap/job/tool-mindthegap-build-macos-10.9.5-gcc-4.2.1/badge/icon)](https://ci.inria.fr/gatb-core/view/MindTheGap/job/tool-mindthegap-build-macos-10.9.5-gcc-4.2.1/)

[![License](http://img.shields.io/:license-affero-blue.svg)](http://www.gnu.org/licenses/agpl-3.0.en.html)

# What is MindTheGap ?

MindTheGap  performs detection and assembly of DNA insertion variants in NGS read datasets with respect to a reference genome. It takes as input a set of reads and a reference genome. It outputs two sets of FASTA sequences: one is the set of breakpoints of detected insertion sites, the other is the set of assembled insertions for each breakpoint.

# Getting the latest source code

## Requirements

CMake 2.6+; see http://www.cmake.org/cmake/resources/software.html

c++ compiler; compilation was tested with gcc and g++ version>=4.5 (Linux) and clang version>=4.1 (Mac OSX).

## Instructions

    # get a local copy of MindTheGap source code
    git clone --recursive https://github.com/GATB/MindTheGap.git
    
    # compile the code an run a simple test on your computer
    cd MindTheGap
    sh INSTALL

# USER MANUAL	 

## Description

MindTheGap is a software that performs detection and assembly of genomic insertion variants in NGS read datasets with respect to a reference genome.

It takes as input a set of reads and a reference genome. It outputs two sets of FASTA sequences: one is the set of breakpoints of detected insertion sites, the other is the set of assembled insertions for each breakpoint. For each breakpoint, MindTheGap either returns a single insertion sequence (when there is no assembly ambiguity), or a set of candidate insertion sequences (due to ambiguities) or nothing at all (when the insertion is too complex to be assembled).

MindTheGap performs de novo assembly using the GATB library and inspired from algorithms from Minia. Hence, the computational resources required to run MindTheGap are significantly lower than that of other assemblers (for instance it uses less than 6GB of main emory for analyzing a full human NGS dataset).

Since version 1.0.0, MindTheGap can detect other types of variants, not only insertion events. These are homozygous SNPs and homozygous deletions of any size. They are detected by the find module of MindTheGap and are output separately in a VCF file. Importantly, even if the user is not interested in these types of variants, it is worth to detect them  since it can improve the recall of the insertion event detection algorithm : it is now possible to find insertion events that are located at less than k nucleotides from an other such variant.
	
## Usage

MindTheGap is composed of two main modules : breakpoint detection (find module) and the local assembly of insertions (fill module). Both steps are implemented in a single executable, MindTheGap, and can be run independently by specifying the module name as follows :

    MindTheGap <module> [module options] 

1. **Basic command lines**

        #Find module:
        MindTheGap find (-in <reads.fq> | -graph <graph.h5>) -ref <reference.fa> [options]
        #To get help:
        MindTheGap find -help
	
        #Fill module:
        MindTheGap fill (-in <reads.fq> | -graph <graph.h5>) -bkpt <breakpoints.fa> [options]
        #To get help:
        MindTheGap fill -help

2. **Input read data**

   For both modules, read dataset(s) are first indexed in a De Bruijn graph. The input format of read dataset(s) is either the read files themselves, or the already computed de bruijn graph in hdf5 format (.h5). In the first case, the option is `-in` and the user can provide the de Bruijn graph building options, in the second case the option is -graph and only options for the detection or assembly are to be given.  
   NOTE: options `-in` and `-graph` are mutually exclusive, and one of these is mandatory.
	
   If the input is composed of several read files, they can be provided as a list of file paths separated by a comma or as a "file of file" (fof), that is a text file containing on each line the path to each read file. All read files will treated as if concatenated in a single sample. The read file format can be fasta, fastq or gzipped. 
		
3. **de Bruijn graph creation options**

   In addition to input read set(s), the de Bruijn graph creation uses two main parameters, `-kmer-size` and `-abundance-min`: 

   * `-kmer-size`: the k-mer size [default '31']. By default, the largest kmer-size allowed is 128. To use k>128, you will need to re-compile MindTheGap with the two commands in the build directory: `cmake -DKSIZE_LIST="32 64 96 256" ..` and then `make`. To go back to default, replace 256 by 128. Note that increasing the range between two consecutive kmer-sizes in the list can have an impact on the size of the output h5 files (but none on the results).

   * `-abundance-min`: the minimal abundance threshold, k-mers having less than this number of occurrences are discarded from the graph [default 'auto', ie. automatically inferred from the dataset]. 

   * `-abundance-max`: the maximal abundance threshold, k-mers having more than this number of occurrences are discarded from the graph [default '2147483647' ie. no limit].
	
4. **Find module specific options**
    
    In addition to the read or graph files, the find module has one mandatory option `-ref` and several optional options:
    * `-ref`: the path to the reference genome file (in fasta format).
    * `-max-rep`: maximal repeat size allowed for fuzzy sites  [default '5']. 
    * `-snp-min-val`: minimal number of kmers to validate a SNP [default '5']. A SNP is validated if it is validated by at least this number of consecutive overlapping kmers. This corresponds also to the minimal distance between two consecutive SNPs to be able to detect both of them.
    * `-het-max-occ`: maximal number of occurrences of a (k-1)mer in the reference genome allowed for heterozyguous insertion breakpoints  [default '1']. In order to detect an heterozyguous insertion breakpoints, both flanking k-1-mers, at each side of the insertion site, must have strictly less than this number of occurrences in the reference genome. This prevents false positive predictions inside repeated regions. Warning : increasing this parameter may lead to numerous false positives (genomic approximate repeats).
    * `-no-[type]`: to disable the detection of certain types of variants.
    * `-[type]-only`: to detect only certain types of variants.
    
    NOTE: MindTheGap can find mainly homozygous variants, except for insertion variants where it can also find heterozygous variants. Therefore -homo-only and -hete-only only applies to insertion variants.

5. **Fill module specific options**
    
    In addition to the read or graph files, the fill module has one mandatory option `-bkpt` and several optional options:	
    * `-bkpt`: the breakpoint file path. This is one of the output of the Find module and contains for each detected insertion site its left and right kmers from and to which the local assembly will be performed (see section E for details about the format).
    * `-max-nodes`: maximum number of nodes in contig graph (nt)  [default '100']. This arguments limits the computational time, this is especially usefull for complex genomes.
    * `-max-length`: maximum length of insertions (nt)  [default '10000']. This arguments limits the computational time, this is especially usefull for complex genomes.

6. **MindTheGap Output**
    
    All the output files are prefixed either by a default name: "MindTheGap_Expe-[date:YY:MM:DD-HH:mm]" or by a user defined prefix (option `-out` of MindTheGap)
    Both MindTheGap modules generate the graph file if reads were given as input: 
    * a graph file (`.h5`). This is a binary file, to obtain information stored in it, you can use the utility program dbginfo located in your bin directory or in ext/gatb-core/bin/.
    
    `MindTheGap find` generates the following output files:
    * a breakpoint file (`.breakpoints`) in fasta format. It contains the breakpoint sequences of each detected insertion site.    Each insertion site corresponds to 2 consecutive entries in the fasta file : sequences are the left and right side flanking kmers.
    * a variant file (`.vcf`) in vcf format. It contains SNPs and deletion events.
    
    `MindTheGap fill` generates the following output files:
    * a sequence file (`.insertions.fasta`) in fasta format. It contains the inserted sequences that were successfully assembled. In the header, position on the reference genome is 
    * an insertion variant file (`.insertions.vcf`) in vcf format. This file resumes insertion position information already contained in the fasta file, but in a vcf format. It is not self-sufficient, inserted sequences if larger than XX bp are not written in the vcf but are referred to their fasta id in the fasta file.
	
7. **Computational resources options**
    
    Additional options are related to computational runtime and memory:
    * `-nb-cores`: number of cores to be used for computation (graph creation and breakpoint detection) [default '0', ie. all available cores will be used].
    * `-max-memory`: max RAM memory for the graph creation (in MBytes)  [default '2000']. Increasing the memory will speed up the graph creation phase.
    * `-max-disk`: max usable disk space for the graph creation (in MBytes)  [default '0', ie. automatically set]. Kmers are counted by writting temporary files on the disk, to speed up the counting you can increase the usable disk space.



## Details on output formats

1. Breakpoint format
    
    A breakpoint file is output by MindTheGap find and is required by MindTheGap fill. This is a plain text file in fasta format, where each insertion site (or gap to fill) corresponds to two consecutive fasta entries: left kmer and right kmer. Sequences are kmer (with k being the same value used in the de bruijn graph). MindTheGap fill will try to find a path in the de bruijn graph from the left kmer to the right one. 
    In the breakpoint file output by MindTheGap find, one can find useful information in the fasta headers, such as the genomic position of the insertion site and its genotype (detected by the homozygous or heterozygous algorithm). A typical header is as follows: 
        
        >bkpt5_left_kmer_chr1_pos_39114_repeat_0_HOM
        #bkpt5 : this is the id of the insertion event. 
        #chr1_pos_39114 : the position of the insertion site is on chr1 at position 39114 (position just before the insertion, 0-based). 
        #repeat_0 : is the size of the repeated sequence at the breakpoint (here 0 means it is a clean insertion site).
        #HOM : it was detected by the homozygous algorithm.
	
2. VCF variant format

    TODO following the VCF guide version X.x...
	
3. Assembled insertion format
    
    MindTheGap fill outputs a file in fasta format containing the obtained inserted sequences. The output sequences do not contain the breakpoint kmers. For each insertion breakpoint for which the filling succeeded, one can find in this file either one or several sequences with the following header:
    
        >bkpt5 insertion_len_59_chr1_pos_39114_repeat_0_HOM
        #same info as in the breakpoint file
        #len_59 : the length in bp of the inserted sequence, here 59 bp

    If more than one sequence is assembled for a given breakpoint, the header is as follows:
    
        >bkpt5 insertion_len_59_chr1_pos_39114_repeat_0_HOM 2/3
        #this is the second sequence out of 3    
 


## Full example

TODO


## Utility programs

Either in your bin/ directory or in ext/gatb-core/bin/, you can find additional utility programs :
* dbginfo : to get information about a graph stored in a .h5 file
* dbgh5 : to build a graph from read set(s) and obtain a .h5 file
* h5dump : to extract all data stored in a .h5 file
	
## Publication

MindTheGap: integrated detection and assembly of short and long insertions. Guillaume Rizk, Anaïs Gouin, Rayan Chikhi and Claire Lemaitre. Bioinformatics 2014 30(24):3451-3457. http://bioinformatics.oxfordjournals.org/content/30/24/3451

 

# Contact

To contact a developer, request help, etc: https://gatb.inria.fr/contact/