

deML: Maximum likelihood demultiplexing for NGS data
==========================================================

QUESTIONS :
   gabriel [dot] reno [ at sign ] gmail.com


About
----------------------

deML is a program for maximum likelihood demultiplexing
of next-generation sequencing data. 


Downloading:
----------------------


Make sure you have zlib.h installed. Go to https://github.com/grenaud/deML and either:

1. Download ZIP 

or

2. Do a "git clone  https://github.com/grenaud/deML.git"


Installation for unix based systems :
----------------------

Mac users need to use the terminal to install and run deML.

1. make sure you have "cmake" and "git" installed, check for it by typing " git --version" and "cmake --version"

2. Build the submodules and main code by typing :
    make





running the program:
----------------------

To launch the program simply type:

    src/deML






 
Format for input
----------------------

The two main inputs that are required are the 

1. Input sequences
2. Indices used in the experiment

----  Input sequences  ----

deML needs your sequences along with the sequenced index in either two formats:

a. BAM where the first index is specified as the XI tag and the quality as YI. 
   XJ and YJ contain the sequence and quality for the second index if it's there

b. fastq file where the forward reads are in one file, reverse in another file.
   The index must be in fastq format. If a second index is present, it must be in its own
   file. If you lost the quality information for the indices (they are present in the defline
   but no quality), you can still use deML but it will not provide optimal results. 
   Use a baseline quality for the bases of the index.


----  Indices used in the experiment  ----

You can either specify the actual sequences used for multiplexing:

```
#Index1	Index2	Name
AATTCAA	CATCCGG	RG1
CGCGCAG	TCATGGT	RG2
AAGGTCT	AGAACCG	RG3
ACTGGAC	TGGAATA	RG4
AGCAGGT	CAGGAGG	RG5
GTACCGG	AATACCT	RG6
GGTCAAG	CGAATGC	RG7
AATGATG	TTCGCAA	RG8
AGTCAGA	AATTCAA	RG9
```

or you can also specify the raw indices:

```
#Index1	Index2	Name
341	33	RG1
342	34	RG2
343	35	RG3
344	36	RG4
345	37	RG5
346	38	RG6
347	39	RG7
348	40	RG8
349	41	RG9
```


However, these numbers must match those in the webform/config.json file. You can alwys modify that file if you use different indices.


Test data:
----------------------

For raw BAM files:

    src/deML -i testData/index.txt -o testData/demultiplexed.bam testData/todemultiplex.bam

For FASTQ files

    src/deML -i testData/index.txt -f testData/todemultiplex.fq1.gz  -r testData/todemultiplex.fq2.gz -if1 testData/todemultiplex.i1.gz  -if2 testData/todemultiplex.i2.gz   -o testData/demultiplexed.


Explanation for the scores
----------------------

deML works by computing the likelihood of stemming from potential samples and assigns a read to the most likely sample. To measure the confidence in the assignment, deML reports 3 different scores (Z_0, Z_1 and Z_2). If you have BAM as input, the output BAM file will include 3 different flags. 


Z_0  The likelihood for the top read group on a PHRED scale. Considering the sequence of this read group as the template to sequence and using the principle of independence between bases, this likelihood is computed as:

```
Z_0= -10 * log_10{ PROD_{1...length indices} P(base read i|base template i)  }
```

The P(base read i|base template i) is given as (1-e_i) if both bases match or e_i otherwise where e_i is the predicted error rate for base i.

Since it is on a likelihood of assignment on a PHRED scale, the lower the Z_0 score, the higher the confidence. Around 0, the probability in assignment to that read group is around 1.



Z_1 Is the likelihood of misassignment on a PHRED scale. Let M be the number of potential read groups are present and let t be the read group with the highestZ_0. If Z_0_i is the Z_0 score for sample i, the Z_1 score is computed by:

```
                    |    SUM_i={1..M} (10^(Z_0_i)) - 10^(Z_0_t)   |
Z_1= -10*log_10     |    --------------------------------------   |
                    |          SUM_i={1..M} (10^(Z_0_i))          |
```


The higher the Z_1 score, the lower the probability of misassignment and the higher the confidence. This score is not reported if only a single read group is found.



Z_2 Is a log odds ratio of mispairing on a logarithmic scale. It is only computed when two indices are used (P7 and P5). It is computed as:

```
                  |   SUM_{i!=j} ( P7_i) (P5_j)  )   |
Z_2  = 10* log_10 |   ------------------------------ |
                  |   SUM_{i==j} ( P7_i) (P5_j)  )   |

```

An approximation is made to speed up computations.  The higher the Z_2 score, the higher the odds ratio of mispairing and the lower the confidence. This score is not reported for runs with single indices and where the log ratio is less than 0 thus making the mispairing scenario unlikely.  


Summary file:
----------------------


If you select the -s or --summary option to obtain a tally, here are the columns and their meaning:



|   Field      |                                      Meaning                                                   |
| -------------|------------------------------------------------------------------------------------------------|
| RG           | Name of read group or sample                                                                   |
| total        | Number of reads assigned to this sample including those failing quality control                |
| total%       | Percentage of reads in "total" as a fraction of the all the reads                              |
| assigned     | Number of reads assigned to this sample that pass quality control                              |
| assigned%    | Percentage of reads in "assigned" as a fraction of the "total"                                 |
| unknown      | Reads that were assigned to this sample but failed Z_0 (prob. of correctness for this sample)  |
| unknown%     | Percentage of reads in "unknown" as a fraction of the "total"                                  |
| conflict     | Reads that were assigned to this sample but failed Z_1 (prob. of misassignment)                |
| conflict%    | Percentage of reads in "conflict" as a fraction of the "total"                                 |
| wrong        | Reads that were assigned to this sample but failed Z_2 (prob. of mispairing, double index only)|
| wrong%       | Percentage of reads in "wrong" as a fraction of the "total"                                    |

Remember that the program will ALWAYS assign a read to a read group if it finds one in the list for the set number of mismatches.  Whether or not the assignment will be allowed to pass quality control will determine if they will make it to the final set.
If you still do not understand assigned, unknown, conflict and wrong, read the following section.

What does assigned, unknown, conflict and wrong really mean?
----------------------

Here is an "Explain like I'm Five" for every category:

* "unknown": It's more likely that you belong to that sample than any other but that probability is low, so you fail ex: P[sample1] = 0.2 P[sample2] = 0.02 P[sample3] = 0.01  and 0.2 is considered low

* "conflict": It's more likely that you belong to that sample than any other but the probability that you belong to another sample is almost equally likely, so you fail ex: P[sample1] = 0.2 P[sample2] = 0.19 P[sample3] = 0.01 

* "wrong": It's more likely that you belong to that sample than any other but it seems your indices (in the case of double indexing) are mispaired so you fail. ex: ex: P[sample1|index1] = 0.1 P[sample1|index2] = 0.01 P[sample2|index1] = 0.01 P[sample2|index2] = 0.1 

* "assigned": Congrats! You have none of the problems listed above ex: P[sample1] = 0.8 P[sample2] = 0.02 P[sample3] = 0.01 


I get "Cannot write to file outputFolder/prefixRGXYZ.fq.gz", what is wrong?
----------------------

There are two possibilities:
1. either you do not have write permission in the directory in which you're trying to write
2. you have reached the limit of open file descriptors for their current system.

determine which of the two is the most likely, tried to create a file called exactly outputFolder/prefixRGXYZ.fq.gz using:

    touch outputFolder/prefixRGXYZ.fq.gz

if this returns an error then you likely do not have the permission to write to this file. Otherwise it is potentially the latter. Run:

    ulimit -n

and most system this command to return 1024. To increase the number of file descriptors use the following command:

    ulimit -n 1024

depending on the system this may require administrator privileges.  However, increasing the limit beyond that requires someone with administrator privileges.



