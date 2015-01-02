---
layout: post
title: BadBFD - part II
---

Introduction
===========

So [last time](https://github.com/orchid-hybrid/orchid-hybrid.github.io/blob/master/_posts/2014-10-20-bufgix-in-binutils.md) I wrote about a pretty scary out of bounds read in binutils that I investigated. Well lcamtuf is back with something even worse..

<img src="http://i.imgur.com/6TvwhWe.png" />

```
$ xxd strings-bfd-badptr 
0000000: 7f45 4c46 0101 0100 0000 0000 0000 0000  .ELF............
0000010: 0200 0300 0100 0000 7480 0408 3400 0000  ........t...4...
0000020: a400 0000 0000 0000 3400 2000 0200 2800  ........4. ...(.
0000030: 0400 0300 0100 0000 0000 0000 0080 0408  ................
0000040: 0080 0408 8000 0000 8000 0000 0500 0000  ................
0000050: 0010 0000 0100 0000 8000 0000 8090 0408  ................
0000060: 8090 0408 0c00 0000 0c00 0000 0600 0000  ................
0000070: 0010 0000 b801 0000 00bb 2a00 0000 cd80  ..........*.....
0000080: 6865 6c6c 6f20 776f 726c 6400 002e 7368  hello world...sh
0000090: 7374 7274 6162 002e 7465 7874 002e 6461  strtab..text..da
00000a0: 7461 0000 0000 0000 0000 0000 0000 0000  ta..............
00000b0: 0000 0000 0000 0000 0000 0000 0000 0000  ................
00000c0: 0000 0000 0000 0000 0000 0000 0b00 0000  ................
00000d0: 0100 0060 e802 0000 7480 0408 7400 0000  ...`....t...t...
00000e0: 0c00 0000 0000 0000 0000 0000 0400 0000  ................
00000f0: 0000 0000 1b00 0000 1100 0000 0300 0000  ................
0000100: 8090 0408 8000 0000 0cff ef00 0000 0000  ................
0000110: 0000 0000 0400 0000 0400 0000 0100 0000  ................
0000120: 0300 0000 0000 0000 0000 0100 8c00 0000  ................
0000130: 9700 0000 0000 0000 0000 deff 0000 0019  ................
0000140: 4444 4444                                DDDD
$ file strings-bfd-badptr 
strings-bfd-badptr: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), statically linked, stripped
```

This pretends to be an elf file but, of course isn't. It crashes strings: do not try this at home, use an appropriate sandbox!

```
$ strings strings-bfd-badptr
Program received signal SIGSEGV, Segmentation fault.
```

gdb is really helpful tracking down what's going on...

```
$ gdb strings
(gdb) r strings-bfd-badptr
Program received signal SIGSEGV, Segmentation fault.
bfd_section_from_shdr (abfd=abfd@entry=0x6db240, shindex=shindex@entry=2) at elf.c:1950
1950			  && (s = idx->shdr->bfd_section) != NULL
(gdb) bt
#0  bfd_section_from_shdr (abfd=abfd@entry=0x6db240, shindex=shindex@entry=2)
    at elf.c:1950
#1  0x0000000000456724 in bfd_elf32_object_p (abfd=0x6db240) at elfcode.h:801
#2  0x0000000000409d6e in bfd_check_format_matches (abfd=abfd@entry=0x6db240, 
    format=format@entry=bfd_object, matching=matching@entry=0x0) at format.c:306
#3  0x000000000040a3b7 in bfd_check_format (abfd=abfd@entry=0x6db240, 
    format=format@entry=bfd_object) at format.c:95
#4  0x0000000000402896 in strings_object_file (file=0x7fffffffe3a7 "strings-bfd-badptr")
    at strings.c:376
#5  strings_file (file=0x7fffffffe3a7 "strings-bfd-badptr") at strings.c:419
#6  main (argc=2, argv=0x7fffffffe048) at strings.c:286
(gdb) quit
```

So the crash occurs at elf.c:1950 - let's look at this code:

```c
    case SHT_GROUP:
      if (! IS_VALID_GROUP_SECTION_HEADER (hdr, GRP_ENTRY_SIZE))
	return FALSE;
      if (!_bfd_elf_make_section_from_shdr (abfd, hdr, name, shindex))
	return FALSE;
      if (hdr->contents != NULL)
	{
	  Elf_Internal_Group *idx = (Elf_Internal_Group *) hdr->contents;
	  unsigned int n_elt = hdr->sh_size / GRP_ENTRY_SIZE;
	  asection *s;

	  if (idx->flags & GRP_COMDAT)
	    hdr->bfd_section->flags
	      |= SEC_LINK_ONCE | SEC_LINK_DUPLICATES_DISCARD;

	  /* We try to keep the same section order as it comes in.  */
	  idx += n_elt;
	  while (--n_elt != 0)
	    {
	      --idx;

	      if (idx->shdr != NULL
		  && (s = idx->shdr->bfd_section) != NULL ///////////////////// BOOM!
		  && elf_next_in_group (s) != NULL)
		{
		  elf_next_in_group (hdr->bfd_section) = s;
		  break;
		}
	    }
	}
      break;
```

The crash occurs at the "BOOM!" location. Debugging printfs shows that `hdr->sh_size` is HUGE, in hexadecimal it's `0x00efff0c` which you can spot in the dangerous file itself on this line:

`0000100: 8090 0408 8000 0000 0cff ef00 0000 0000  ................`

After dumping the binary to hex using xxd, editing the large number into something really small like '3', then re-create it with `xdd -r` strings no longer crashes on it. Great!

```
$ strings strings-bfd-badptr-edited
BFD: strings-bfd-badptr-edited: no group info for section .text
BFD: strings-bfd-badptr-edited: no group info for section .text
hello world
.shstrtab
.text
.data
DDDD
```

Looking into the [elf32 file format](http://www.bottomupcs.com/elf.html) [a little bit](http://www.skyfree.org/linux/references/ELF_Format.pdf) while looking at the code we can see what's going on here: This particular elf section header is claiming that the section is huge (way bigger than the rest of the file) - the next iteration of the loop then derefs the non-NULL but very much out of bounds pointer `idx->shdr` causing a segfault.

I noticed VALID_GROUP_SECTION_HEADER as a candidate for fixing this:

```c
#define IS_VALID_GROUP_SECTION_HEADER(shdr, minsize)	\
	(   (shdr)->sh_type == SHT_GROUP		\
	 && (shdr)->sh_size >= minsize			\
	 && (shdr)->sh_entsize == GRP_ENTRY_SIZE	\
	 && ((shdr)->sh_size % GRP_ENTRY_SIZE) == 0)
```

makes sure sh_size is large enough but there's no upper bound! If you stick some dumb upper bound on like 0x100 it fixes the crash but (A) How would we know what number to use? (B) This only covers group section headers, the same problem might occur for other section headers. We can do better..

So I go through the entire file looking for every place that sh_size gets initialized adding a printf, I need to locate where the evil number 0x00efff0c gets read into sh_size so that I can follow control flow from there on to know where to put a bounds check:

```
  this_hdr->sh_size = asect->size;
  shstrtab_hdr->sh_size = _bfd_elf_strtab_size (elf_shstrtab (abfd));
  symtab_hdr->sh_size = symtab_hdr->sh_entsize * (symcount + 1);
  ...
```

None of them get triggered!

Back to the backtrace: 

```
#1  0x0000000000456724 in bfd_elf32_object_p (abfd=0x6db240) at elfcode.h:801
```

it's the call to elf_swap_shdr_in from elf_object_p that does it, now elf_object_p does a bunch of good sanity checks before diving into elf.c we can add a test that makes sure that the section header wont be so large it goes beyond the file bounds. Hopefully we can ask the bfd structure a bound the filesize, I found this in bfdio.c:

```c
/*
FUNCTION
	bfd_get_size

SYNOPSIS
	file_ptr bfd_get_size (bfd *abfd);

DESCRIPTION
	Return the file size (as read from file system) for the file
	associated with BFD @var{abfd}.

	The initial motivation for, and use of, this routine is not
	so we can get the exact size of the object the BFD applies to, since
	that might not be generally possible (archive members for example).
	It would be ideal if someone could eventually modify
	it so that such results were guaranteed.

	Instead, we want to ask questions like "is this NNN byte sized
	object I'm about to try read from file offset YYY reasonable?"
	As as example of where we might do this, some object formats
	use string tables for which the first <<sizeof (long)>> bytes of the
	table contain the size of the table itself, including the size bytes.
	If an application tries to read what it thinks is one of these
	string tables, without some way to validate the size, and for
	some reason the size is wrong (byte swapping error, wrong location
	for the string table, etc.), the only clue is likely to be a read
	error when it tries to read the table, or a "virtual memory
	exhausted" error when it tries to allocate 15 bazillon bytes
	of space for the 15 bazillon byte table it is about to read.
	This function at least allows us to answer the question, "is the
	size reasonable?".
*/
```

Bingo! This is exactly what we want. So here's an attempt at a patch:

```c
      /* Read in the rest of the section header table and convert it
	 to internal form.  */
      for (shindex = 1; shindex < i_ehdrp->e_shnum; shindex++)
	{
	  if (bfd_bread (&x_shdr, sizeof x_shdr, abfd) != sizeof (x_shdr))
	    goto got_no_match;
	  elf_swap_shdr_in (abfd, &x_shdr, i_shdrp + shindex);

          // XXXX: sanity check offset and size
          { unsigned int filesize = bfd_get_size(abfd);
          if (i_shdrp[shindex].sh_offset >= filesize
              || i_shdrp[shindex].sh_offset + i_shdrp[shindex].sh_size >= filesize) {
            goto got_wrong_format_error;
          } } // XXXX

	  /* Sanity check sh_link and sh_info.  */
	  if (i_shdrp[shindex].sh_link >= num_sec)
	    {
	      /* PR 10478: Accept Solaris binaries with a sh_link
		 field set to SHN_BEFORE or SHN_AFTER.  */
	      switch (ebd->elf_machine_code)
```

and it does fix the crash:

```
$ strings strings-bfd-badptr
hello world
.shstrtab
.text
.data
DDDD
$ objdump -d strings-bfd-badptr
objdump: strings-bfd-badptr: File format not recognised
```

This fixes the particular crash of a hugely out of bounds claimed section header size but it's not totally clear if it fixes the problem entirely: maybe if the number `0x00efff0c` is something much smaller (within bounds) but still extremal it could still cause a segfault or some kind of memory corruption. This will need to be investigated carefully.

A quick check of enumerating all small numbers around the size of the file with the following command gives evidence that the patch is ok:

```
for i in `seq 0 255` ; do h=`printf "%02x" $i` ; printf "\x$h" | dd of=strings-bfd-badptr-mech bs=1 seek=264 count=1 conv=notrunc &> /dev/null ; strings strings-bfd-badptr-mech  ; done
```


**Update**: There's a thread about these bug on a mailing list here: http://comments.gmane.org/gmane.comp.security.oss.general/14512
