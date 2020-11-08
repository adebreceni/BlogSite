# Cast iron

As Sarah was laying in her bed, turning and tossing unable to reach dreamland, what could have been the source of her newfound awareness? Could it have been famine? It could have been, had it not been rather the question of what is the fastest way to find an `int` in a `std.vector[int]` . 

```scala
size_t find(vec: const std.vector[int]&, item: int){
  auto it = std.find(vec.begin(), vec.end(), item);
  if(it == vec.end()) return size_t.max;
  return std.distance(vec.begin(), it);
}
```

"Don't forget  to profile in release." said the friendly rooster on her shoulder. "Right" said Sarah as she mind-compiled the code.

```scala
g++ -std=c++17 -O3 main.cpp
```

What test data should she benchmark this function with? She figured, the super-arbitrary `4096` would be a great size for the vector, and made sure so that we are searching for the last element. Even though she couldn't sleep, she was not moderately tired, so her brain could only produce a computing power equivalent to a 1.6GHz dual core i5-8210Y CPU. "Will have to do for now" she thought.

```scala
1270 ± 7.16 ns 
```

Not super bad, is it? 

Could she speed up this if some precondition were to be ensured? What about "the item we are searching for is guaranteed to be in the vector", does that sound like a good precondition? How would this make our code faster. In this form, it wouldn't. She will have to hand-craft the loop that is abstracted away by `std.find`. Let's go for it.

```scala
size_t find(vec: const std.vector[int]&, item: int){
  for(size_t idx = 0; idx < vec.length(); ++idx){
    if(vec[idx] == item) return idx;
  }
  return size_t.max;
}
```

Is this any slower? 

```scala
1259 ± 3.91 ns
```

Not particularly, moreover in this form we can use the precondition and omit an extremely expensive check: `idx < vec.length()`. After all, if we know we will find the element, why check if we ran out of elements? Sarah can already hear the CPU frequency going down due to the immense time we just saved for each iteration. (If only the branch prediction didn't already make it virtually zero-cost.)

```scala
1511 ± 12.8 ns
```

Oh boy! Look at all that runtime we just saved, `-250 ns`..., 😑. Something smells fishy here. Lets look at the assembly. 

```scala
	movq	(%rdi), %rcx
	movq	$-1, %rax
loop:
	cmpl	%esi, 4(%rcx,%rax,4)
	leaq	1(%rax), %rax
	jne	loop
	retq
```

For some reason her brain, which is equivalent to clang 12.0.0, decided to use `leaq` to increment `%rax` instead of `incq` as in the original checked loop. And sure enough fiddling a bit with the assembly produces the same speed as the original.

```scala
	movq	(%rdi), %rcx
	movq	$-1, %rax
loop:
  incq %rax
	cmpl	%esi, (%rcx,%rax,4)
	jne	loop
	retq
```

(Notice we must move `incq` before the check, as this instruction does not preserve the status flags.)

Well we did a lot of work, with nothing to show for. 

### Loop unrolling

Our little loop is nice and compact. Let's make it horrendous but fast(er).

```scala
size_t find(vec: const std.vector[int]&, item: int){
  for(size_t idx = 0;;){
    if(vec[idx++] == item) return idx - 1;
		if(vec[idx++] == item) return idx - 1;
  }
}
```

It looks like 🤢and like somebody should be fired if they try to merge it to production. Nevertheless it is faster, finally.

```scala
942 ± 9.89 ns
```

Can we unroll more and achieve better results? One could always try but in this case it seems unfruitful. 

Time to bring out the big guns: hand-written assembly. Unmaintainable. Not in the hot path. We are going to be the future git-blame king. Let's do it!

```scala
	xorl	%eax, %eax
  movq	(%rdi), %rcx
loop:
  cmpl	%esi, (%rcx,%rax,4)
  je return
  cmpl	%esi, 4(%rcx,%rax,4)
  je _1
  cmpl	%esi, 8(%rcx,%rax,4)
  je _2
  cmpl	%esi, 12(%rcx,%rax,4)
  je _3
  cmpl	%esi, 16(%rcx,%rax,4)
  je _4
  cmpl	%esi, 20(%rcx,%rax,4)
  leaq	6(%rax), %rax
  jne loop
  decq %rax
  ret
_4:
  incq %rax
_3:
  incq %rax
_2:
  incq %rax
_1:
  incq %rax
return:
  ret
```

After some trial-and-error Sarah finds that `5` such comparisons should be done in a single loop for maximum performance: `849 ± 10.8 ns` . We are now `50%` faster than our standard library implementation. (If you are wondering about why we went with the waterfall-like `%rax` adjustment instead of calling `addq` , yes you are right to wonder about that.)

Is this the best Sarah can do? She could probably squeeze out more by reordering the instruction, but going in that direction is quite uninteresting. We are in too deep now, in it to win it.

What if Sarah made further assumptions? Like that she will be the only one ever running the code in her i5 brain? `-march=native` to the rescue. Switching it on, she observes no discernible speed-up. This is so sad 😞

But wait a minute... something is moving in the bushes. Could it be?

### Advanced Vector Extensions

The good news, is that we won't have to write assembly for this part (albeit you could if you really want to).

In newer hardware we have an extra set of registers called the vector registers, and a slew of instructions to treat those `128`, `256`, `512` bit registers as a vector of smaller sized objects, like 8 32 bit ints or 4 64 bit doubles, and operate on them at the same time.

Let's see how we could use them

```scala
size_t find(vec: const std.vector[int] &, item: int)
{
  static_assert(sizeof(int) == 4);
  static_assert(CHAR_BIT == 8);
  auto target = _mm256_set_epi32(item, item, item, item, item, item, item, item);
  auto data = reinterpret_cast[const __m256i *](v.data());
  for (size_t idx = 0;;++idx)
  {
    auto cmp_res = _mm256_cmpeq_epi32(data[idx], target);
    auto result = _mm256_movemask_ps(_mm256_castsi256_ps(cmp_res));
    if (result) {
      return idx * 8 + __builtin_ctz(result);
		}
  }
}
```

Before delving into the code take a look at the numbers:

```scala
214 ± 2.16 ns
```

This is now SIX times faster than the original implementation. Okay onto the code. We are first spreading our target value in vector registers, like butter on a piece of bread (`_mm256_set_epi32`). Then we tell the compiler to trust us, we know what we are doing (do we?) and let us treat the contents of the vector as if we were storing 256 bit values in it. Then the magic happens: `_mm256_cmpeq_epi32` treats each 8x32 bits as if they were ints (they are) and compares each of them with the corresponding element in the other storage, which is coincidentally filled with our target value. The other parts just aggregate the comparison results into a single value and check if and where our value can be found.

Sarah at this moment is on the verge of slipping away. Could we do something more?

I promised you no assembly, and as it turns out that was a big fat lie.

```scala
vmovd	%esi, %xmm0
  vpbroadcastd	%xmm0, %ymm0
  movq	(%rdi), %rax
loop:
  vpcmpeqd	(%rax), %ymm0, %ymm1
  vmovmskps	%ymm1, %rdx
  addq	$32, %rax
  testb	%dl, %dl
  je	loop

  tzcntl	%edx, %edx
  subq $32, %rax
  subq (%rdi), %rax
  shr $2, %rax
  addq	%rdx, %rax
  vzeroupper
  ret
```

This is all it takes to achieve:

```scala
167 ± 3.95 ns
```

As Sarah can finally rest she acknowledges that yes, to cast to `__m256i` the data has to be `32` byte aligned, and yes if the vector's length is not a multiple of `32` she could access out of bounds. Of course these could be mitigated with a pre-loop setup step to check the first few elements until the alignment matches, and a post-loop step to check the remaining steps.