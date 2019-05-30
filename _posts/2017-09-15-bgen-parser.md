---
layout: post-no-feature
title: "Parsing BGEN files in C++"
description: "Playing around with binary files and the zlib c++ libraries."
category: "work"
tags: [c++, bgen]
---




> <center> The code is available on github <a href="github.com/agilly/b-fast/">here</a>.</center>

### Introduction
The full release of UK Biobank data was made available recently, marking the definitive start of the post-GWAS era: it is now unrealistically slow and cumbersome to process association data using the manual mapping and reducing approach of the early 00's. One big issue being the size of the released data (500k samples over 90M variants), the data were released in the [well-documented](http://www.well.ox.ac.uk/~gav/bgen_format/bgen_format_v1.2.html) BGEN format. The [original repo](https://bitbucket.org/gavinband/bgen/src) contains some example code and a tentative library of sorts, but functionality is very limited. For example, the toy `bgen_to_vcf.cpp` file is close to 300 lines long (and as of f2083028f07e0d9d36cbbf3eb136754580d9c7c4 contains a mistake in the VCF header). Out of curiosity, and to refresh my C++, here's an attempt at hacking those BGEN files and providing a slightly more high-level interface. 

### Reading the header

The header is as follows:


**length(bytes)**|**description**|**symbol**|**assertion**
:-----:|:-----:|:-----:|:-----:
4|offset|\\(o\\)| $$L_h + L_{SI} = o $$|
4|length of header|$$L_h$$| 
4|number of variants|$$M$$| 
4|number of samples|$$N$$| 
4|magic number| | `== "bgen" || == 0`|
$$L_h-20$$ | free data (empty in UKBB) | | 
4 | flags ||

`flags` is a bitmaks determining the format specification (v1.1, v1.2 or v1.3), the compression algorithm used for the records and the presence or absence of a sample identifier block of size $$L_{SI}$$. The sample identifier block is absent in the UKBB release files. This is followed by an empty space made of an arbitrary number of 0's; the total number of bytes from the beginning of the file to the first variant descriptor is $$o + 4$$.

First we create a struct to contain the header info. All variables are read and stored in the types corresponding to their `sizeof` for easy writing, except `magic`, the magic `b`, `g`, `e`, `n` `char*` that is cast to a string, and the `freedata` block, which is likely to be a string as well.

```cpp
struct file_info{
	file_info();
	file_info(uint32_t header_offset, uint32_t size_of_header, uint32_t num_variants, uint32_t num_samples, string magic, int freedata_size, string freedata, uint32_t flags, uint32_t empty_space);
	uint32_t header_offset;
	uint32_t size_of_header;
	uint32_t num_variants;
	uint32_t num_samples;
	string magic;
	int freedata_size;
	string freedata;
	uint32_t flags;
	unsigned short int compression;
	bool bgen_version;
	bool sample_info_included;
	uint32_t empty_space;
};
```
