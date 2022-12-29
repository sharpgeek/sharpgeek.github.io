---
layout: post
title:  "Optimal numeric types in embedded projects"
categories: iar optimization
---

I had a project where I used the IAR Compiler for Arm to build the application for a Cortex-M3 target. At the earlier stages, I had to come up with some rough estimates to decide which numeric type to use, knowing that the processor was lacking a dedicated FPU.

The following snippet mainly helped to reveal the resource consumption difference between `int` and `float` numeric types.

```c
#include <stdio.h>
#include <time.h>

#ifndef NUMTYPE
#define NUMTYPE float
#endif

#define PI (3.141593f)
#define SIZE (200)

#define __FORMATTER(num) _Generic( (num), \
                                   float:"Result: %f\n", \
                                   default:"Result: %d\n")
typedef NUMTYPE data_t;

void main()
{
    unsigned int r[SIZE];
    data_t area[SIZE];

    for (size_t i = 0; i < SIZE; i++) {
        r[i] = (unsigned int)( (i + 10u) * 2u );
    }

    clock_t t = clock();

    for (size_t i = 0; i < sizeof(area)/sizeof(data_t); i++) {
        area[i] = (data_t)( PI * r[i] * r[i] );
        printf(__FORMATTER(area[i]), area[i]);
    }

    t = clock() - t;

    unsigned int elapsed = ((double)t)/CLOCKS_PER_SEC;
    printf("Elapsed time: %u ms.\n", elapsed);
}
```

From the bash shell, we can compile the compile unit for the two use cases, generating a list for each one of them:

```bash
LINKER_CONFIG=/opt/iarsystems/bxarm/arm/config/linker/TexasInstruments/LM3S6965.icf;

i=float
iccarm -DNUMTYPE=$i -lC main-$i --cpu=cortex-m3 --thumb -r main.c -o main.o;
ilinkarm --semihosting --map demo-$i.map --config $LINKER_CONFIG main.o -o demo-$i.elf;

i=int
iccarm -DNUMTYPE=$i -lC main-$i --cpu=cortex-m3 --thumb -r main.c -o main.o;
ilinkarm --semihosting --map demo-$i.map --config $LINKER_CONFIG main.o -o demo-$i.elf;
```

The contents of `main-float.lst` and `main-int.lst` will show that they have essentially the same approximate resource consumption:

- `main-float.lst`:

```
   Maximum stack usage in bytes:

   .cstack Function
   ------- --------
    1624   main
      1624   -> __aeabi_d2uiz
      1624   -> __aeabi_ddiv
      1624   -> __aeabi_dmul
      1624   -> __aeabi_f2d
      1624   -> __aeabi_fmul
      1624   -> __aeabi_ui2d
      1624   -> __aeabi_ui2f
      1624   -> clock
      1624   -> printf


   Section sizes:

   Bytes  Function/Label
   -----  --------------
      12  ?_0
      24  ?_1
     172  main


  36 bytes in section .rodata
 172 bytes in section .text

 172 bytes of CODE  memory
  36 bytes of CONST memory
```

- `main-int.lst`:

```
   Maximum stack usage in bytes:

   .cstack Function
   ------- --------
    1624   main
      1624   -> __aeabi_d2uiz
      1624   -> __aeabi_ddiv
      1624   -> __aeabi_dmul
      1624   -> __aeabi_f2iz
      1624   -> __aeabi_fmul
      1624   -> __aeabi_ui2d
      1624   -> __aeabi_ui2f
      1624   -> clock
      1624   -> printf


   Section sizes:

   Bytes  Function/Label
   -----  --------------
      12  ?_0
      24  ?_1
     168  main


  36 bytes in section .rodata
 168 bytes in section .text

 168 bytes of CODE  memory
  36 bytes of CONST memory
```


From a terminal, run QEMU:
```bash
qemu-system-arm -cpu cortex-m3 -M lm3s6965evb -nographic -semihosting-config enable=on,target=native -gdb tcp::3333 -S -kernel ./demo-float.elf
```

From a second terminal, run `gdb-multiarch -q ./demo-float.elf` and enter the commands:
```gdb
target remote :1234
continue
```

Then after repeating the process for the integer demo (`demo-int.elf`), got the following timer results:
- Using `float`, elapsed time was ~7000 ms.
- Using `int`, elapsed time was ~3000 ms.

As expected, even when using a rough estimate such as the one provided by the ANSI C `CLOCKS_PER_SEC` from `<time.h>`, software-based `float` clearly becomes expensive in low end devices (with no FPU). Many times `int`, while faster and computationally cheaper, provides somewhat good enough results. Later I will run it on an actual target so we can get precise measurements.
