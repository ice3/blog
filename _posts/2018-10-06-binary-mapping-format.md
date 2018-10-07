---
layout: post
title: PAF I save 90 % of disc space
date: 2018-06-18
published: true
tags: draft overlapper long-read compression
---

{% include setup %}

# PAF I save 90 % of disc space

Durring last week Shaun Jack post this on twitter :

<blockquote class="twitter-tweet" data-lang="fr">
<p lang="en" dir="ltr">
I have a 1.2 TB PAF.gz file of minimap2 all-vs-all alignments of 18 flowcells of Oxford Nanopore reads. Yipes. I believe that&#39;s my first file to exceed a terabyte. Is there a better way? Perhaps removing the subsumed reads before writing the all-vs-all alignments to disk?</p>&mdash; Shaun Jackman (@sjackman) <a href="https://twitter.com/sjackman/status/1047729989318139904?ref_src=twsrc%5Etfw">4 octobre 2018</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

For people not working on long-read assembly, firstly welcome, secondly minimap is a very good mapper used to find similar region between long reads, its output is in PAF for Pairwise Alignment Format, this format is presented in [minimap2 man page](https://lh3.github.io/minimap2/minimap2.html#10). Roughly it's a tsv file, which, for each similar region found (called before match) [pas compris?], the format stores the two reads names, reads lengths, begin and end position of the match, plus some other informations.  

This tweet creates some discussion and a third solution was proposed, use classic mapping against reference compression [compressed ?] format, filter some match, creation of a new binary compressed format to store all-vs-all mapping.

## Use mapping reference file

Torsten Seemann and me suggest to use sam minimap outupt and compress it in BAM/CRAM but after little investigation apprently it isn't working [pas compris?] well.

<blockquote class="twitter-tweet" data-lang="fr">
<p lang="en" dir="ltr">My trouble to convert bam in cram is solved thank to <a href="https://twitter.com/RayanChikhi?ref_src=twsrc%5Etfw">@RayanChikhi</a> !<br><br>minimap output without sequence compress in cram,noref (i.e. tmp_short.cram) is little bit heavier than classic paf compress in gz.<br><br>So it&#39;s probably time for a Binary Alignment Format. <a href="https://t.co/W02wj7tf2I">pic.twitter.com/W02wj7tf2I</a></p>&mdash; Pierre Marijon (@pierre_marijon) <a href="https://twitter.com/pierre_marijon/status/1047798695822024704?ref_src=twsrc%5Etfw">4 octobre 2018</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Ok even removing unnecessary fields in the sam format (sequence and mapping field), and with a better compression solution, this isn't better than a PAF format file compressed by gzip. Maybe with larger file, this solution could be save more space.

## Filter

Heng Li say :

<blockquote class="twitter-tweet" data-lang="fr">
<p lang="en" dir="ltr">Minimap2 reports hits, not only overlaps. The great majority of hits are non-informative to asm. Hits involving repeats particularly hurt. For asm, there are ways to reduce them, but that needs evaluation. I won&#39;t go into that route soon because ... 1/2</p>&mdash; Heng Li (@lh3lh3) <a href="https://twitter.com/lh3lh3/status/1047823011527753728?ref_src=twsrc%5Etfw">4 octobre 2018</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Minimap didn't make any assumption about what the user wants to do with read matching, and it's a very good thing but some times you store too much information for your usage. So a filter overlap before storing it on your disk could be a good idea [pas compris?].

A little awk, bash, python, {choose your langage} script could make this job perfectly.

Alex Di Genov [suggest to use minimap2 API](https://twitter.com/digenoma/status/1047852263111385088) to build a special minimap with intigrate filter, this solution have probably better performance than *ad hoc *script but it's less flexible you need use minimap2.

My solution is a little soft in rust [fpa (Filter Pairwise Alignment)](https://github.com/natir/fpa), fpa take as input a path [je sais pas là] or mhap, and can filter the match by:
- type: containment, internal-match, dovetail
- length: match is upper or lower than a threshold
- read name: match against a regex, it's a read match against him

fpa is avaible in bioconda and in cargo.

Ok, so filter match is easy and we have an already available solution. 

## A Binary Alignment Fromat

<blockquote class="twitter-tweet" data-lang="fr">
<p lang="en" dir="ltr">Is it time for a Binary Alignment Format that uses integer compression techniques?</p>&mdash; Rayan Chikhi (@RayanChikhi) <a href="https://twitter.com/RayanChikhi/status/1047773219086897153?ref_src=twsrc%5Etfw">4 octobre 2018</a>
</blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

This is the question they initiate this blog post.

Here I just want to present a little investigation I made, about how we can compress Pairwise Alignment, I will call this format jPAF and it's just a POC and this never changes. [pas compris? quoi qui never change]

Roughly jPAF is a json compressed format, so it isn't really binary, but I just want test some idea so it's cool.

We have 3 main object in this json:
- header\_index: a dict associating an index to an header name
- reads\_index: associating a read name and her length to an index
- matchs: a list of matches, a match header\_index

A little example are probably better:

```
{
   "header_index":{
      "0":"read_a",
      "1":"begin_a",
      "2":"end_a",
      "3":"strand",
      "4":"read_b",
      "5":"begin_b",
      "6":"end_b",
      "7":"nb_match",
      "8":"match_length",
      "9":"qual"
   },
   "read_index":[
      {
         "name":"1_2",
         "len":5891
      },
      {
         "name":"2_3",
         "len":4490
      },
   ],
   "match":[
      [
         0,
         1642,
         5889,
         "+",
         1,
         1,
         4248,
         4247,
         4247,
         0,
         "tp:A:S"
      ],
   ]
}
```

Here we have a PAF like header, two reads 1_2 and 2_3 with 5891 and 4490 base respectively and one overlap with length 4247 base in same strand between them. [pas compris?]

jPAF are fully inspired by PAF, they share the same fields names, I just take the PAF, convert it in json and add two little tricks to save space.

The first trick is to store the read name and length only one time.
The second trick is more of a json trick, first time each record are a dictionary with keyname associate to value, with this solution baf is heaviest than paf [pas compris?]. If I associate a field name to an index, I can store the records in a table not in a json object and therefore, avoid redondancy.

Ok, so now, I have a pretty cool format, avoiding some repetitions without any information loss, but does it really saves space or not ?

## Result

Dataset: For this, I use the same dataset as in my previous blog post. It consist of two real datasets, a pacbio one and a nanopore one.

Mapping : I run minimap2 mapping with preset ava-pb and ava-ont for pacbio and nanopore respectively.

This table presents file size and space savings against some other file types. sam bam and cram file are long or short, in long we keep all data present in minimap2 output, in short we replace sequence and quality by a \*.

If you want to replicate all these results, just follow the instructions you can found in this [github repro](https://github.com/natir/jPAF).

## Discussion

Ok minimap generate too many matches [pas compris?], but it's easy to remove unuseful matches, with fpa or with *ad hoc* tools. The format design to store mapping against reference, aren't better than an actual format compressed with a generalist algorithm.

The main problem of jPaf is many the quality, read\_index it's save disk space but time and memory performances, you can't stream out this format, you need wait until all alignments are found before writing, when you need to read the file, you need to keep read\_index in RAM any time. [pas compris?]

But I wrote it in two hours, the format is lossless and they save **33 to 64 %** of disk space, depending of the compression used. 

Actually I'm not sure we need binary compressed format for store pairwise alignement against read, but in future with more and more long-read data we will probably need it. And this result encourages me to continue to think about this problem and read more things around bam/cram canu [canu?] ovlStore.

If you want search with me [pas compris] and discussion about Pairwise Aligment format comment section is available.
