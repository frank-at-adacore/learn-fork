Types
=======


===============
Integer-Based Types
===============

Integers are C workhorse type as they are use everywhere to encode all king of information. 
In Ada we usually constraint or specialize types for many reasons:
   - Improve semantics
   - Static analysis
   - Efficiency both in space and time

The question one should ask when defining an integer based type is what semantic should it expose.

To illustrate many uses cases let's make a thin and thick binding of the *nix mmap function.
The man page says: "mmap() creates a new mapping in the virtual address space of the calling process."
On linux you can find its definition under "sys/mann.h"

extern void *mmap (void *  __addr,       /* starting address for the mapping */
                   size_t   __len,       /* number of bytes to be mapped */
                   int      __prot,      /* kind of access permitted. */
		             int      __flags,     /* visibility to other processes mapping the same region */
                   int      __fd,        /* file descriptor */
                   __off_t  __offset) __THROW;

============
Thin binding
============

with Interfaces.C;

subtype Size_T is Interfaces.C.Unsigned_Long;
subtype Off_T is Interfaces.C.Long;

function Mmap (Addr  : System.Address; 
               Len   : Size_T;
               Prot  : Interfaces.C.Int; 
               Flags : Interfaces.C.Int; 
               Fd    : Interfaces.C.Int;
               Off   : Off_T) return System.Address 
      with Import        => True, 
           Convention    => C, 
           External_Name => "mmap";

Void* are represented by System.Address because, after all, that's what they are. 
You will find a detailed discussion on how to handle void* in the Special Case section of this document.

In Ada, basic numeric C types are found in the package Interfaces.C. 
If you dig in the Linux source files you find that Size_T is a typedef to unsigned long. 
To preserve the type name, Size_t, but resolve to the correct fundamental type, unsigned long, 
the Ada way is to define a subtype Size_T as an Interfaces.C.Unsigned_Long. The same logic applies for Off_T.

============
Thick binding
============


============
Notion of flags
============

Let's detail the protection parameter of mmap:

int      __prot,      /* kind of access permitted. */

Here are the possible values found in mman-common.h:

#define PROT_READ	      0x1		   /* Page can be read.  */
#define PROT_WRITE	   0x2		   /* Page can be written.  */
#define PROT_EXEC	      0x4		   /* Page can be executed.  */
#define PROT_NONE	      0x0		   /* Page can not be accessed.  */
#define PROT_GROWSDOWN	0x01000000	/* Extend change to start of growsdown vma (mprotect only).  */
#define PROT_GROWSUP	   0x02000000	/* Extend change to start of growsup vma (mprotect only).  */

This is a typical flags use case. Lets replace bitwise arithmetic with some flag semantics.
In Ada you can define a custom type of given size and make it use a specific representation.
Here we define a Boot_T type, scoped to the Mman package, as an enumeration (Y, N) holding on 1 bit.
We also add a numeric representation to the enumeration elements T => 0, Y => 1.

package Mman is

   type Flag_T is (No, Yes) with Size => 1;                  -- enum Flag_T holding on 1 bit
   for Flag_T use (No => 0, Yes => 1);                       -- "represented" by numeric value 0, 1

   type Protection_T is record
      None      : Flag_T;
      Read      : Flag_T;
      Write     : Flag_T;
      Exec      : Flag_T;
      Growsdown : Flag_T;
      Growsup   : Flag_T;
   end record with Size => Interfaces.C.Int'Size;         -- same size as original C type
   for Protection_T use record                            -- specify mapping of elements
      None      at 0 range  0 ..  0;    -- 16#0000_0000#
      Read      at 0 range  1 ..  1;    -- 16#0000_0001#
      Write     at 0 range  2 ..  2;    -- 16#0000_0002#
      Exec      at 0 range  3 ..  3;    -- 16#0000_0004#
      Growsdown at 0 range  28 ..  28;  -- 16#0100_0000#
      Growsup   at 0 range  29 ..  29;  -- 16#0200_0000#
   end record;

end Mman;

In C, one would do:

prot |= (1U<<1);   // can read
prot &= ~(1U<<2);  // can not write

Our thick Ada binding gives us the ability to express:

prot.Read := Yes;  -- or prot.Read := 1;
prot.Write := No; -- or prot.Write := 0;


Note that __flags modeling would benefit from the same approach


============
Notion of Identifiers
============

Let's bind the file descriptor parameter of mmap:

int      __fd,        /* file descriptor */

If you try ulimit -nH on your *nix system you will get the maximum allowed number of open files by process.
On my Linux, it returns 1048576. Typically on a 64bit system an int ranges from -2**63 .. 2**63-1;
In decimal the maximum positive value is 9223372036854775807, which is by far greater than the maximum allowed on my system 1048576.

Everywhere you look for a semantic definition of a file descriptor you get something along this:
"File descriptors typically have non-negative integer values, with negative values being reserved to indicate "no value" or error conditions."

It can be argue that 






Thin binding

void* __addr // preferred starting address for the mapping

size_t __len // number of bytes to be mapped
typedef unsigned long size_t

int __prot // kind of access is permitted. This argument may be logical ‘OR’ of the following flags PROT_READ | PROT_WRITE | PROT_EXEC | PROT_NONE.
/usr/include/asm-generic/mman-common.h
#define PROT_READ	0x1		/* page can be read */
#define PROT_WRITE	0x2		/* page can be written */
#define PROT_EXEC	0x4		/* page can be executed */
#define PROT_SEM	0x8		/* page may be used for atomic ops */
#define PROT_NONE	0x0		/* page can not be accessed */
#define PROT_GROWSDOWN	0x01000000	/* mprotect flag: extend change to start of growsdown vma */
#define PROT_GROWSUP	0x02000000	/* mprotect flag: extend change to end of growsup vma */

int __flags
/usr/include/linux/mman.h
#define MAP_SHARED	0x01		/* Share changes */
#define MAP_PRIVATE	0x02		/* Changes are private */
#define MAP_SHARED_VALIDATE 0x03	/* share + validate extension flags */

int __fd
int fd = open(status_file, O_RDONLY); // check qnx how they defined fd type

TBD

.. toctree::
   :maxdepth: 2

   Integer-Based Types <types/integer_based_types>

   a) c --> Ada, should use full Integer type as anything between -2**8 .. 2**8-1 can be passed?
   b) Ada --> c, should use the most constrained type of integer for the problem at hand?
   - Is doing so considered a thick binding?
   - Is the conversion to full integer on C side always guaranteed?
   "Guidelines for Ada/SPARK Binding Design and Implementation":
   Does the thick binding avoid unnecessary use of implicit-size integer types (like Integer, Natural, Positive, etc.) 
   or explicit-size but otherwise-undifferentiated integer types (like Interfaces.Unsigned_32)?
   If not, could it be made to use integer types defined for the specific purpose for which they are used?




   Enumerated Types <types/enumerated_types>
   Strings <types/strings>
   Arrays <types/arrays>


QUESTIONS for Michael: Should we make a copy of "Guidelines for Ada/SPARK Binding Design and Implementation" 
to anotate each section to make sure we covered it. Also notes for each section on where in our document it should go.