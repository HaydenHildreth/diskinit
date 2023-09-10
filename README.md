/*                                                             /
/*                            initDISK.c                       /
/*                                                             /
/*                      Copyright (c) 2001                     /
/*                      tom ehlert                             /
/*                      All Rights Reserved                    /
/*                                                             /
/* This file is part of DOS-C.                                 /
/*                                                             /
/* DOS-C is free software; you can redistribute it and/or      /
/* modify it under the terms of the GNU General Public License /
/* as published by the Free Software Foundation; either version/
/* 2, or (at your option) any later version.                   /
/*                                                             /
/* DOS-C is distributed in the hope that it will be useful, but/
/* WITHOUT ANY WARRANTY; without even the implied warranty of  /
/* MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See   /
/* the GNU General Public License for more details.            /
/*                                                             /
/* You should have received a copy of the GNU General Public   /
/* License along with DOS-C; see the file COPYING.  If not,    /
/* write to the Free Software Foundation, 675 Mass Ave,        /
/* Cambridge, MA 02139, USA.                                   /
/****************************************************************/

#include "portab.h"
#include "debug.h"
#include "init-mod.h"
#include "dyndata.h"

UBYTE InitDiskTransferBuffer[SEC_SIZE] BSS_INIT({0});
COUNT nUnits BSS_INIT(0);

/*
   Rev 1.0   13 May 2001  tom ehlert
Initial revision.

this module implements the disk scanning for DOS accesible partitions
the drive letter ordering is somewhat chaotic, but like MSDOS does it.

this module expects to run with CS = INIT_TEXT, like other init_code,
but SS = DS = DATA = DOS_DS, unlike other init_code.

history:
1.0 extracted the disk init code from DSK.C
    added LBA support
    moved code to INIT_TEXT
    done the funny code segment stuff to switch between INIT_TEXT and TEXT
    added a couple of snity checks for partitions

***************************************************************************

Implementation note:
this module needs some interfacing to INT 13
how to implement them
   
a) using inline assembly 
       _ASM mov ax,0x1314

b) using assembly routines in some external FLOPPY.ASM

c) using the funny TURBO-C style
       _AX = 0x1314

d) using intr(intno, &regs) method.

why not?

a) this is my personal favorite, combining the best aof all worlds.
   TURBO-C does support inline assembly, but only by using TASM,
   which is not free. 
   so - unfortunately- its excluded.

b) keeping funny memory model in sync with external assembly
   routines is everything, but not fun

c) you never know EXACT, what the compiler does, if its a bit
   more complicated. does
     _DL = drive & 0xff    
     _BL = driveParam.chs.Sector;
   destroy any other register? sure? _really_ sure?
   at least, it has it's surprises.
   and - I found a couple of optimizer induced bugs (TC 2.01)
     believe me.
   it was coded - and operational that way.
   but - there are many surprises waiting there. so I opted against.


d) this method is somewhat clumsy and certainly not the
   fastest way to do things.
   on the other hand, this is INIT code, executed once.
   and since it's the only portable method, I opted for it.

e) and all this is my private opinion. tom ehlert.


Some thoughts about LBA vs. CHS. by Bart Oldeman 2001/Nov/11
Matthias Paul writes in www.freedos.org/freedos/news/technote/113.html:
(...) MS-DOS 7.10+, which will always access logical drives in a type
05h extended partition via CHS, even if the individual logical drives
in there are of LBA type, or go beyond 8 Gb... (Although this workaround
is sometimes used in conjunction with OS/2, using a 05h partition going
beyond 8 Gb may cause MS-DOS 7.10 to hang or corrupt your data...) (...)

Also at http://www.win.tue.nl/~aeb/partitions/partition_types-1.html:
(...) 5 DOS 3.3+ Extended Partition
  Supports at most 8.4 GB disks: with type 5 DOS/Windows will not use the
  extended BIOS call, even if it is available. (...)

So MS-DOS 7.10+ is brain-dead in this respect, but we knew that ;-)
However there is one reason to use old-style CHS calls:
some programs intercept int 13 and do not support LBA addressing. So
it is worth using CHS if possible, unless the user asks us not to,
either by specifying a 0x0c/0x0e/0x0f partition type or enabling
the ForceLBA setting in the fd kernel (sys) config. This will make
multi-sector reads and BIOS computations more efficient, at the cost
of some compatibility.

However we need to be safe, and with varying CHS at different levels
that might be difficult. Hence we _only_ trust the LBA values in the
partition tables and the heads and sectors values the BIOS gives us.
After all these are the values the BIOS uses to process our CHS values.
So unless the BIOS is buggy, using CHS on one partition and LBA on another
should be safe. The CHS values in the partition table are NOT trusted.
We print a warning if there is a mismatch with the calculated values.

The CHS values in the boot sector are used at a higher level. The CHS
that DOS uses in various INT21/AH=44 IOCTL calls are converted to LBA
using the boot sector values and then converted back to CHS using BIOS
values if necessary. Internally we do LBA as much as possible.

However if the partition extends beyond cylinder 1023 and is not labelled
as one of the LBA types, we can't use CHS and print a warning, using LBA
instead if possible, and otherwise refuse to use it.

As for EXTENDED_LBA vs. EXTENDED, FreeDOS makes no difference. This is
boot time - there is no reason not to use LBA for reading partition tables,
and the MSDOS 7.10 behaviour is not desirable.

Note: for floppies we need the boot sector values though and the boot sector
code does not use LBA addressing yet.

Conclusion: with all this implemented, FreeDOS should be able to gracefully
handle and read foreign hard disks moved across computers, whether using
CHS or LBA, strengthening its role as a rescue environment.
