---
layout: post
title: a bug in binutils
---

On twitter @lcamtuf posted a scary thing:

<img src=http://i.imgur.com/bUiXrfV.png></img>

When you run strings on it it causes a SIGSEGV, that kind of crash could very likely lead to an exploit! The same crash occurs when you run objdump -d and strip on the file. strip leaves some weird looking tmp files around since it didn't exit normally.

This is because it's triggering a crash in libbfd, which those 3 programs use.

One program which does not crash on this binary is od -x:

    $ od -x stringme 
    0000000 3353 3330 0020 0000 ad00 ff00 80ad 0000
    0000020 0c04 0400 ff7d 0080 3353 3330 0020 0000
    0000040 adf8 ff00 b0c8 10ad 0c04 aead adad e803
    0000060 ada2 0007 000c 00fa fa00 adad adad 00ad
    0000100 3600 3400 0000 0200 adf6 adad 00ae ffff
    0000120 00ff 0200 adf6 adad adae 03ad a2e8 00ad
    0000140 adad aead adad ad87 ada2 0000 000c a034
    0000160 adae adad adad 00ad f600 adad aead ff00
    0000200 ffff e400 f602 0cad 3400 e600 00ff
    0000215

I don't have a good guess what this file is, maybe created by fuzzing or just really carefully built to crash binutils?

So I recompiled binutils in my sandbox to investigate the crash. Triggering the crash inside GDB points me to [bfd/srec.c line 557](https://github.com/bneumeier/binutils/blob/master/bfd/srec.c#L557):

		while (bytes > 0)
		  {
		    check_sum += HEX (data);
		    data += 2;
		    bytes--;
		  }

First I made sure this the crash was isolated to this part of code by putting continues on various parts. Satisfied that this was the correct place to look I read some of the surrounding code and decided that this part was correct (two steps of data corresponds to one step of bytes).

So it's time for printf debugging. The value of bytes just before entering this loop is -1, that isn't right! Bytes is calculated in the following way:

	    if (bfd_bread (hdr, (bfd_size_type) 3, abfd) != 3)
	      goto error_return;

	     ....

	    check_sum = bytes = HEX (hdr + 1);

The 'hdr' data comes out as `hdr[0]=33` `hdr[1]=30` `hdr[2]=33` which is ascii for `303` - we can notice these numbers in the original file too. HEX (hdr+1) then codes "03" into 3. Now the idea is to trace through the relevant path of code (the case which ends up at that while loop) watching the variable `bytes` with '3' in mind...

	      case '3':
		      check_sum += HEX (data);
		      address = HEX (data);
		      data += 2;
		      --bytes;                           // BYTES becomes 2 here
		      /* Fall through.  */
	      case '2':
		      check_sum += HEX (data);
		      address = (address << 8) | HEX (data);
		      data += 2;
		      --bytes;                          // BYTES becomes 1 here
		      /* Fall through.  */
	      case '1':
		      check_sum += HEX (data);
		      address = (address << 8) | HEX (data);
		      data += 2;
		      check_sum += HEX (data);
		      address = (address << 8) | HEX (data);
		      data += 2;
		      bytes -= 2;                       // BYTES becomes -1 here, but
          ...                               // it's an unsigned integer so
          ...                               // rolls over and becomes gigantic
          ...                               // this makes the while loop access
          ...                               // way way outside bounds!
          while (bytes > 0)
		      {
		        check_sum += HEX (data);
		        data += 2;
		        bytes--;
		      }


So there's an easy fix to this! Add checks to make sure bytes is not too small at each fallthrough. e.g.

    if(bytes < 2) goto error_return;

I made this patch and checked that it fixes the crash, it even gives the strings of the binary now.

    $ strings stingme
    S303
    S303

The crash is caused by reading way out of bounds, I do not know how this can be used to get code execution: any ideas?
