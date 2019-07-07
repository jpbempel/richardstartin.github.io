---
ID: 10295
post_title: Matrix Multiplication Revisited
author: Richard Startin
post_excerpt: ""
layout: post
permalink: http://richardstartin.uk/mmm-revisited/
published: true
post_date: 2018-01-15 17:05:57
---
In a <a href="http://richardstartin.uk/multiplying-matrices-fast-and-slow/" rel="noopener" target="_blank">recent post</a>, I took a look at matrix multiplication in pure Java, to see if it can go faster than reported in <a href="https://astojanov.github.io/publications/preprint/004_cgo18-simd.pdf" rel="noopener" target="_blank">SIMD Intrinsics on Managed Language Runtimes</a>. I found faster implementations than the paper's benchmarks implied was possible. Nevertheless, I found that there were some limitations in Hotspot's autovectoriser that I didn't expect to see, even in JDK10. Were these limitations somehow fundamental, or can other compilers do better with essentially the same input? 

I took a look at the code generated by GCC's autovectoriser to see what's possible in C/C++ without resorting to complicated intrinsics. For a bit of fun, I went over to the dark side to squeeze out <del>some</del> a lot of extra performance, which gave inspiration to a simple vectorised Java implementation which can maintain intensity as matrix size increases.

<h3>Background</h3>

The paper reports a 5x improvement in matrix multiplication throughput as a result of using LMS generated intrinsics. Using GCC as LMS's backend, I easily reproduced very good throughput, but I found two Java implementations better than the paper's baseline. The best performing Java implementation proposed in the paper was <code language="java">blocked</code>. This post is not about the LMS benchmarks, but this code is this post's inspiration.

<code class="language-java">
public void blocked(float[] a, float[] b, float[] c, int n) {
  int BLOCK_SIZE = 8; 
  // GOOD: attempts to do as much work in submatrices
  // GOOD: tries to avoid bringing data through cache multiple times
  for (int kk = 0; kk < n; kk += BLOCK_SIZE) {
    for (int jj = 0; jj < n; jj += BLOCK_SIZE) {
      for (int i = 0; i < n; i++) {
        for (int j = jj; j < jj + BLOCK_SIZE; ++j) {
          // BAD: manual unrolling, bypasses optimisations
          float sum = c[i * n + j]; 
          for (int k = kk; k < kk + BLOCK_SIZE; ++k) {
            // BAD: second read (k * n) requires a gather - bad for cache, bad for dTLB
            // BAD: horizontal sums are inefficient
            sum += a[i * n + k] * b[k * n + j]; 
          }
          c[i * n + j] = sum;
         }
       }
    }
  }
}
</code>

I proposed the following implementation for improved cache efficiency and expected it to vectorise automatically.

<code class="language-java">public void fast(float[] a, float[] b, float[] c, int n) {
   // GOOD: 2x faster than "blocked" - why?
   int in = 0;
   for (int i = 0; i < n; ++i) {
       int kn = 0;
       for (int k = 0; k < n; ++k) {
           float aik = a[in + k];
           // MIXED: passes over c[in:in+n] multiple times per k-value, "free" if n is small
           // MIXED: reloads b[kn:kn+n] repeatedly for each i, bad if n is large, "free" if n is small
           // BAD: doesn't vectorise but should
           for (int j = 0; j < n; ++j) {
               c[in + j] += aik * b[kn + j]; // sequential writes and reads, cache and vectoriser friendly
           }
           kn += n;
       }
       in += n;
    }
}
</code>

My code actually doesn't vectorise, even in JDK10, which really surprised me because the <a href="http://richardstartin.uk/autovectorised-fma-in-jdk10/">inner loop vectorises if the offsets are always zero</a>. In any case, there is a simple hack involving the use of buffers, which unfortunately thrashes the cache, but narrows the field significantly.

<code class="language-java">  
  public void fastBuffered(float[] a, float[] b, float[] c, int n) {
    float[] bBuffer = new float[n];
    float[] cBuffer = new float[n];
    int in = 0;
    for (int i = 0; i < n; ++i) {
      int kn = 0;
      for (int k = 0; k < n; ++k) {
        float aik = a[in + k];
        System.arraycopy(b, kn, bBuffer, 0, n);
        saxpy(n, aik, bBuffer, cBuffer);
        kn += n;
      }
      System.arraycopy(cBuffer, 0, c, in, n);
      Arrays.fill(cBuffer, 0f);
      in += n;
    }
  }
</code>

I left the problem looking like this, with the "JDKX vectorised" lines using the algorithm above with a buffer hack:

<img src="http://richardstartin.uk/wp-content/uploads/2017/12/Plot-52-2.png" alt="" width="700" height="500" class="alignnone size-full wp-image-10222" />

<h3>GCC Autovectorisation</h3>

The Java code is very easy to translate into C/C++. Before looking at performance I want to get an idea of what GCC's autovectoriser does. I want to see the code generated at GCC <a href="https://gcc.gnu.org/onlinedocs/gcc/Optimize-Options.html" rel="noopener" target="_blank">optimisation level 3</a>, with unrolled loops, FMA, and AVX2, which can be seen as follows:

<pre>g++ -mavx2 -mfma -march=native -funroll-loops -O3 -S mmul.cpp</pre>

The generated assembly code can be seen in full context <a href="https://github.com/richardstartin/cppavxbenchmarks/blob/master/mmul.s" rel="noopener" target="_blank">here</a>. Let's look at the <code language="java">mmul_saxpy</code> routine first:

<code class="language-cpp">static void mmul_saxpy(const int n, const float* left, const float* right, float* result) {
    int in = 0;
    for (int i = 0; i < n; ++i) {
        int kn = 0;
        for (int k = 0; k < n; ++k) {
            float aik = left[in + k];
            for (int j = 0; j < n; ++j) {
                result[in + j] += aik * right[kn + j];
            }
            kn += n;
        }
        in += n;
    }
}
</code>

This routine uses SIMD instructions, which means in principle any other compiler could do this too. The inner loop has been unrolled, but this is only by virtue of the <code language="java">-funroll-loops</code> flag. C2 does this sort of thing as standard, but only for hot loops. In general you might not want to unroll loops because of the impact on code size, and it's great that a JIT compiler can decide only to do this when it's profitable.

<code class="language-cpp">.L9:
	vmovups	(%rdx,%rax), %ymm4
	vfmadd213ps	(%rbx,%rax), %ymm3, %ymm4
	addl	$8, %r10d
	vmovaps	%ymm4, (%r11,%rax)
	vmovups	32(%rdx,%rax), %ymm5
	vfmadd213ps	32(%rbx,%rax), %ymm3, %ymm5
	vmovaps	%ymm5, 32(%r11,%rax)
	vmovups	64(%rdx,%rax), %ymm1
	vfmadd213ps	64(%rbx,%rax), %ymm3, %ymm1
	vmovaps	%ymm1, 64(%r11,%rax)
	vmovups	96(%rdx,%rax), %ymm2
	vfmadd213ps	96(%rbx,%rax), %ymm3, %ymm2
	vmovaps	%ymm2, 96(%r11,%rax)
	vmovups	128(%rdx,%rax), %ymm4
	vfmadd213ps	128(%rbx,%rax), %ymm3, %ymm4
	vmovaps	%ymm4, 128(%r11,%rax)
	vmovups	160(%rdx,%rax), %ymm5
	vfmadd213ps	160(%rbx,%rax), %ymm3, %ymm5
	vmovaps	%ymm5, 160(%r11,%rax)
	vmovups	192(%rdx,%rax), %ymm1
	vfmadd213ps	192(%rbx,%rax), %ymm3, %ymm1
	vmovaps	%ymm1, 192(%r11,%rax)
	vmovups	224(%rdx,%rax), %ymm2
	vfmadd213ps	224(%rbx,%rax), %ymm3, %ymm2
	vmovaps	%ymm2, 224(%r11,%rax)
	addq	$256, %rax
	cmpl	%r10d, 24(%rsp)
	ja	.L9
</code>

The <code language="java">mmul_blocked</code> routine is compiled to quite convoluted assembly. It has a huge problem with the expression <code language="java">right[k * n + j]</code>, which requires a gather and is almost guaranteed to create 8 cache misses per block for large matrices. Moreover, this inefficiency gets much worse with problem size.

<code class="language-cpp">static void mmul_blocked(const int n, const float* left, const float* right, float* result) {
    int BLOCK_SIZE = 8;
    for (int kk = 0; kk < n; kk += BLOCK_SIZE) {
        for (int jj = 0; jj < n; jj += BLOCK_SIZE) {
            for (int i = 0; i < n; i++) {
                for (int j = jj; j < jj + BLOCK_SIZE; ++j) {
                    float sum = result[i * n + j];
                    for (int k = kk; k < kk + BLOCK_SIZE; ++k) {
                        sum += left[i * n + k] * right[k * n + j]; // second read here requires a gather
                    }
                    result[i * n + j] = sum;
                }
            }
        }
    }
}
</code>

This compiles to assembly with the unrolled vectorised loop below:

<code class="language-cpp">.L114:
	cmpq	%r10, %r9
	setbe	%cl
	cmpq	56(%rsp), %r8
	setnb	%dl
	orl	%ecx, %edx
	cmpq	%r14, %r9
	setbe	%cl
	cmpq	64(%rsp), %r8
	setnb	%r15b
	orl	%ecx, %r15d
	andl	%edx, %r15d
	cmpq	%r11, %r9
	setbe	%cl
	cmpq	48(%rsp), %r8
	setnb	%dl
	orl	%ecx, %edx
	andl	%r15d, %edx
	cmpq	%rbx, %r9
	setbe	%cl
	cmpq	40(%rsp), %r8
	setnb	%r15b
	orl	%ecx, %r15d
	andl	%edx, %r15d
	cmpq	%rsi, %r9
	setbe	%cl
	cmpq	32(%rsp), %r8
	setnb	%dl
	orl	%ecx, %edx
	andl	%r15d, %edx
	cmpq	%rdi, %r9
	setbe	%cl
	cmpq	24(%rsp), %r8
	setnb	%r15b
	orl	%ecx, %r15d
	andl	%edx, %r15d
	cmpq	%rbp, %r9
	setbe	%cl
	cmpq	16(%rsp), %r8
	setnb	%dl
	orl	%ecx, %edx
	andl	%r15d, %edx
	cmpq	%r12, %r9
	setbe	%cl
	cmpq	8(%rsp), %r8
	setnb	%r15b
	orl	%r15d, %ecx
	testb	%cl, %dl
	je	.L111
	leaq	32(%rax), %rdx
	cmpq	%rdx, %r8
	setnb	%cl
	cmpq	%rax, %r9
	setbe	%r15b
	orb	%r15b, %cl
	je	.L111
	vmovups	(%r8), %ymm2
	vbroadcastss	(%rax), %ymm0
	vfmadd132ps	(%r14), %ymm2, %ymm0
	vbroadcastss	4(%rax), %ymm1
	vfmadd231ps	(%r10), %ymm1, %ymm0
	vbroadcastss	8(%rax), %ymm3
	vfmadd231ps	(%r11), %ymm3, %ymm0
	vbroadcastss	12(%rax), %ymm4
	vfmadd231ps	(%rbx), %ymm4, %ymm0
	vbroadcastss	16(%rax), %ymm5
	vfmadd231ps	(%rsi), %ymm5, %ymm0
	vbroadcastss	20(%rax), %ymm2
	vfmadd231ps	(%rdi), %ymm2, %ymm0
	vbroadcastss	24(%rax), %ymm1
	vfmadd231ps	0(%rbp), %ymm1, %ymm0
	vbroadcastss	28(%rax), %ymm3
	vfmadd231ps	(%r12), %ymm3, %ymm0
	vmovups	%ymm0, (%r8)
</code>

<h3>Benchmarks</h3>

I implemented a suite of <a href="https://github.com/richardstartin/cppavxbenchmarks/blob/master/mmul.cpp" rel="noopener" target="_blank">benchmarks</a> to compare the implementations. You can run them, but since they measure throughput and intensity averaged over hundreds of iterations per matrix size, the full run will take several hours. 

<pre>g++ -mavx2 -mfma -march=native -funroll-loops -O3 mmul.cpp -o mmul.exe && ./mmul.exe > results.csv</pre>

The <code language="java">saxpy</code> routine wins, with <code language="java">blocked</code> fading fast after a middling start.

<img src="http://richardstartin.uk/wp-content/uploads/2018/01/Plot-60.png" alt="" width="1077" height="605" class="alignnone size-full wp-image-10361" />

<div class="table-holder">
<table class="table table-bordered table-hover table-condensed">
<thead><tr><th>name</th>
<th>size</th>
<th>throughput (ops/s)</th>
<th>flops/cycle</th>
</tr></thead>
<tbody><tr>
<td>blocked</td>
<td align="right">64</td>
<td align="right">22770.2</td>
<td align="right">4.5916</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">64</td>
<td align="right">25638.4</td>
<td align="right">5.16997</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">128</td>
<td align="right">2736.9</td>
<td align="right">4.41515</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">128</td>
<td align="right">4108.52</td>
<td align="right">6.62783</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">192</td>
<td align="right">788.132</td>
<td align="right">4.29101</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">192</td>
<td align="right">1262.45</td>
<td align="right">6.87346</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">256</td>
<td align="right">291.728</td>
<td align="right">3.76492</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">256</td>
<td align="right">521.515</td>
<td align="right">6.73044</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">320</td>
<td align="right">147.979</td>
<td align="right">3.72997</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">320</td>
<td align="right">244.528</td>
<td align="right">6.16362</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">384</td>
<td align="right">76.986</td>
<td align="right">3.35322</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">384</td>
<td align="right">150.441</td>
<td align="right">6.55264</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">448</td>
<td align="right">50.4686</td>
<td align="right">3.4907</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">448</td>
<td align="right">95.0752</td>
<td align="right">6.57594</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">512</td>
<td align="right">30.0085</td>
<td align="right">3.09821</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">512</td>
<td align="right">65.1842</td>
<td align="right">6.72991</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">576</td>
<td align="right">22.8301</td>
<td align="right">3.35608</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">576</td>
<td align="right">44.871</td>
<td align="right">6.59614</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">640</td>
<td align="right">15.5007</td>
<td align="right">3.12571</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">640</td>
<td align="right">32.3709</td>
<td align="right">6.52757</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">704</td>
<td align="right">12.2478</td>
<td align="right">3.28726</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">704</td>
<td align="right">25.3047</td>
<td align="right">6.79166</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">768</td>
<td align="right">8.69277</td>
<td align="right">3.02899</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">768</td>
<td align="right">19.8011</td>
<td align="right">6.8997</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">832</td>
<td align="right">7.29356</td>
<td align="right">3.23122</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">832</td>
<td align="right">15.3437</td>
<td align="right">6.7976</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">896</td>
<td align="right">4.95207</td>
<td align="right">2.74011</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">896</td>
<td align="right">11.9611</td>
<td align="right">6.61836</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">960</td>
<td align="right">3.4467</td>
<td align="right">2.34571</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">960</td>
<td align="right">9.25535</td>
<td align="right">6.29888</td>
</tr>
<tr>
<td>blocked</td>
<td align="right">1024</td>
<td align="right">2.02289</td>
<td align="right">1.67082</td>
</tr>
<tr>
<td>saxpy</td>
<td align="right">1024</td>
<td align="right">6.87039</td>
<td align="right">5.67463</td>
</tr>
</tbody></table>
</div>

With GCC autovectorisation, <code language="java">saxpy</code> performs well, maintaining intensity as size increases, albeit well below the theoretical capacity. It would be nice if similar code could be JIT compiled in Java.

<h3>Intel Intrinsics</h3>

To understand the problem space a bit better, I find out how fast matrix multiplication can get without domain expertise by handcrafting an algorithm with intrinsics. My laptop's Skylake <a href="https://ark.intel.com/products/88967/Intel-Core-i7-6700HQ-Processor-6M-Cache-up-to-3_50-GHz" rel="noopener" target="_blank">chip</a> (turbo boost and hyperthreading disabled) is capable of 32 SP flops per cycle per core - Java and the LMS  implementation previously fell a long way short of that. It was difficult getting beyond 4f/c with Java, and LMS peaked at almost 6f/c before quickly tailing off. GCC autovectorisation achieved and maintained 7f/c.

To start, I'll take full advantage of the facility to align the matrices on 64 byte intervals, since I have 64B cache lines, though this might just be voodoo. I take the <code language="java">saxpy</code> routine and replace its kernel with intrinsics. Because of the <code language="java">-funroll-loops</code> option, this will get unrolled without effort.

<code class="language-cpp">static void mmul_saxpy_avx(const int n, const float* left, const float* right, float* result) {
    int in = 0;
    for (int i = 0; i < n; ++i) {
        int kn = 0;
        for (int k = 0; k < n; ++k) {
            __m256 aik = _mm256_set1_ps(left[in + k]);
            int j = 0;
            for (; j < n; j += 8) {
                _mm256_store_ps(result + in + j, _mm256_fmadd_ps(aik, _mm256_load_ps(right + kn + j), _mm256_load_ps(result + in + j)));
            }
            for (; j < n; ++j) {
                result[in + j] += left[in + k] * right[kn + j];
            }
            kn += n;
        }
        in += n;
    }
}
</code>

This code is actually not a lot faster, if at all, than the basic <code language="java">saxpy</code> above: a lot of aggressive optimisations have already been applied.

<h3>Combining Blocked and SAXPY</h3> 

What makes <code language="java">blocked</code> so poor is the gather and the cache miss, not the concept of blocking itself. A limiting factor for <code language="java">saxpy</code> performance is that the ratio of loads to floating point operations is too high. With this in mind, I tried combining the blocking idea with <code language="java">saxpy</code>, by implementing <code language="java">saxpy</code> multiplications for smaller sub-matrices. This results in a different algorithm with fewer loads per floating point operation, and the inner two loops are swapped. It avoids the gather and the cache miss in <code language="java">blocked</code>. Because the matrices are in row major format, I make the width of the blocks much larger than the height. Also, different heights and widths make sense depending on the size of the matrix, so I choose them dynamically. The design constraints are to avoid gathers and horizontal reduction.

<code class="language-cpp">static void mmul_tiled_avx(const int n, const float *left, const float *right, float *result) {
    const int block_width = n >= 256 ? 512 : 256;
    const int block_height = n >= 512 ? 8 : n >= 256 ? 16 : 32;
    for (int row_offset = 0; row_offset < n; row_offset += block_height) {
        for (int column_offset = 0; column_offset < n; column_offset += block_width) {
            for (int i = 0; i < n; ++i) {
                for (int j = column_offset; j < column_offset + block_width && j < n; j += 8) {
                    __m256 sum = _mm256_load_ps(result + i * n + j);
                    for (int k = row_offset; k < row_offset + block_height && k < n; ++k) {
                        sum = _mm256_fmadd_ps(_mm256_set1_ps(left[i * n + k]), _mm256_load_ps(right + k * n + j), sum);
                    }
                    _mm256_store_ps(result + i * n + j, sum);
                }
            }
        }
    }
}
</code> 

You will see in the benchmark results that this routine really doesn't do very well compared to <code language="java">saxpy</code>. Finally, I unroll it, which <em>is</em> profitable despite setting <code language="java">-funroll-loops</code> because there is slightly more to this than an unroll. This is a sequence of vertical reductions which have no data dependencies.

<code class="language-cpp">static void mmul_tiled_avx_unrolled(const int n, const float *left, const float *right, float *result) {
    const int block_width = n >= 256 ? 512 : 256;
    const int block_height = n >= 512 ? 8 : n >= 256 ? 16 : 32;
    for (int column_offset = 0; column_offset < n; column_offset += block_width) {
        for (int row_offset = 0; row_offset < n; row_offset += block_height) {
            for (int i = 0; i < n; ++i) {
                for (int j = column_offset; j < column_offset + block_width && j < n; j += 64) {
                    __m256 sum1 = _mm256_load_ps(result + i * n + j);
                    __m256 sum2 = _mm256_load_ps(result + i * n + j + 8);
                    __m256 sum3 = _mm256_load_ps(result + i * n + j + 16);
                    __m256 sum4 = _mm256_load_ps(result + i * n + j + 24);
                    __m256 sum5 = _mm256_load_ps(result + i * n + j + 32);
                    __m256 sum6 = _mm256_load_ps(result + i * n + j + 40);
                    __m256 sum7 = _mm256_load_ps(result + i * n + j + 48);
                    __m256 sum8 = _mm256_load_ps(result + i * n + j + 56);
                    for (int k = row_offset; k < row_offset + block_height && k < n; ++k) {
                        __m256 multiplier = _mm256_set1_ps(left[i * n + k]);
                        sum1 = _mm256_fmadd_ps(multiplier, _mm256_load_ps(right + k * n + j), sum1);
                        sum2 = _mm256_fmadd_ps(multiplier, _mm256_load_ps(right + k * n + j + 8), sum2);
                        sum3 = _mm256_fmadd_ps(multiplier, _mm256_load_ps(right + k * n + j + 16), sum3);
                        sum4 = _mm256_fmadd_ps(multiplier, _mm256_load_ps(right + k * n + j + 24), sum4);
                        sum5 = _mm256_fmadd_ps(multiplier, _mm256_load_ps(right + k * n + j + 32), sum5);
                        sum6 = _mm256_fmadd_ps(multiplier, _mm256_load_ps(right + k * n + j + 40), sum6);
                        sum7 = _mm256_fmadd_ps(multiplier, _mm256_load_ps(right + k * n + j + 48), sum7);
                        sum8 = _mm256_fmadd_ps(multiplier, _mm256_load_ps(right + k * n + j + 56), sum8);
                    }
                    _mm256_store_ps(result + i * n + j, sum1);
                    _mm256_store_ps(result + i * n + j + 8, sum2);
                    _mm256_store_ps(result + i * n + j + 16, sum3);
                    _mm256_store_ps(result + i * n + j + 24, sum4);
                    _mm256_store_ps(result + i * n + j + 32, sum5);
                    _mm256_store_ps(result + i * n + j + 40, sum6);
                    _mm256_store_ps(result + i * n + j + 48, sum7);
                    _mm256_store_ps(result + i * n + j + 56, sum8);
                }
            }
        }
    }
}
</code>

This final implementation is fast, and is probably as good as I am going to manage, without reading papers. This should be a CPU bound problem because the algorithm is O(n^3) whereas the problem size is O(n^2). But the flops/cycle decreases with problem size in all of these implementations. It's possible that this could be amelioarated by a better dynamic tiling policy. I'm unlikely to be able to fix that. 

It does make a huge difference being able to go very low level - handwritten intrinsics with GCC unlock awesome throughput - but it's quite hard to actually get to the point where you can beat a good optimising compiler. Mind you, there are harder problems to solve this, and you may well be a domain expert. 

The benchmark results summarise this best:

<img src="http://richardstartin.uk/wp-content/uploads/2018/01/Plot-62.png" alt="" width="1078" height="605" class="alignnone size-full wp-image-10388" />


<div class="table-holder">
<table class="table table-bordered table-hover table-condensed">
<thead><tr><th>name</th>
<th>size</th>
<th>throughput (ops/s)</th>
<th>flops/cycle</th>
</tr></thead>
<tbody><tr>
<td>saxpy_avx</td>
<td align="right">64</td>
<td align="right">49225.7</td>
<td align="right">9.92632</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">64</td>
<td align="right">33680.5</td>
<td align="right">6.79165</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">64</td>
<td align="right">127936</td>
<td align="right">25.7981</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">128</td>
<td align="right">5871.02</td>
<td align="right">9.47109</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">128</td>
<td align="right">4210.07</td>
<td align="right">6.79166</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">128</td>
<td align="right">15997.6</td>
<td align="right">25.8072</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">192</td>
<td align="right">1603.84</td>
<td align="right">8.73214</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">192</td>
<td align="right">1203.33</td>
<td align="right">6.55159</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">192</td>
<td align="right">4383.09</td>
<td align="right">23.8638</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">256</td>
<td align="right">633.595</td>
<td align="right">8.17689</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">256</td>
<td align="right">626.157</td>
<td align="right">8.0809</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">256</td>
<td align="right">1792.52</td>
<td align="right">23.1335</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">320</td>
<td align="right">284.161</td>
<td align="right">7.1626</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">320</td>
<td align="right">323.197</td>
<td align="right">8.14656</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">320</td>
<td align="right">935.571</td>
<td align="right">23.5822</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">384</td>
<td align="right">161.517</td>
<td align="right">7.03508</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">384</td>
<td align="right">188.215</td>
<td align="right">8.19794</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">384</td>
<td align="right">543.235</td>
<td align="right">23.6613</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">448</td>
<td align="right">99.1987</td>
<td align="right">6.86115</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">448</td>
<td align="right">118.588</td>
<td align="right">8.2022</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">448</td>
<td align="right">314</td>
<td align="right">21.718</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">512</td>
<td align="right">70.0296</td>
<td align="right">7.23017</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">512</td>
<td align="right">73.2019</td>
<td align="right">7.55769</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">512</td>
<td align="right">197.815</td>
<td align="right">20.4233</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">576</td>
<td align="right">46.1944</td>
<td align="right">6.79068</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">576</td>
<td align="right">50.6315</td>
<td align="right">7.44294</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">576</td>
<td align="right">126.045</td>
<td align="right">18.5289</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">640</td>
<td align="right">33.8209</td>
<td align="right">6.81996</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">640</td>
<td align="right">37.0288</td>
<td align="right">7.46682</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">640</td>
<td align="right">92.784</td>
<td align="right">18.7098</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">704</td>
<td align="right">24.9096</td>
<td align="right">6.68561</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">704</td>
<td align="right">27.7543</td>
<td align="right">7.44912</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">704</td>
<td align="right">69.0399</td>
<td align="right">18.53</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">768</td>
<td align="right">19.5158</td>
<td align="right">6.80027</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">768</td>
<td align="right">21.532</td>
<td align="right">7.50282</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">768</td>
<td align="right">54.1763</td>
<td align="right">18.8777</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">832</td>
<td align="right">12.8635</td>
<td align="right">5.69882</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">832</td>
<td align="right">14.6666</td>
<td align="right">6.49766</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">832</td>
<td align="right">37.9592</td>
<td align="right">16.8168</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">896</td>
<td align="right">12.0526</td>
<td align="right">6.66899</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">896</td>
<td align="right">13.3799</td>
<td align="right">7.40346</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">896</td>
<td align="right">34.0838</td>
<td align="right">18.8595</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">960</td>
<td align="right">8.97193</td>
<td align="right">6.10599</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">960</td>
<td align="right">10.1052</td>
<td align="right">6.87725</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">960</td>
<td align="right">21.0263</td>
<td align="right">14.3098</td>
</tr>
<tr>
<td>saxpy_avx</td>
<td align="right">1024</td>
<td align="right">6.73081</td>
<td align="right">5.55935</td>
</tr>
<tr>
<td>tiled_avx</td>
<td align="right">1024</td>
<td align="right">7.21214</td>
<td align="right">5.9569</td>
</tr>
<tr>
<td>tiled_avx_unrolled</td>
<td align="right">1024</td>
<td align="right">12.7768</td>
<td align="right">10.5531</td>
</tr>
</tbody></table>
</div>

<h3>Can we do better in Java?</h3>

Writing genuinely fast code gives an indication of how little of the processor Java actually utilises, but is it possible to bring this knowledge over to Java? The <code language="java">saxpy</code> based implementations in my previous post performed well for small to medium sized matrices. Once the matrices grow, however, they become too big to be allowed to pass through cache multiple times: we need hot, small cached data to be replenished from the larger matrix. Ideally we wouldn't need to make any copies, but it seems that the autovectoriser doesn't like offsets: <code language="java">System.arraycopy</code> is a reasonably fast compromise. The basic sequential read pattern is validated: even native code requiring a gather does not perform well for this problem. The best effort C++ code translates almost verbatim into this Java code, which is quite fast for large matrices.

<code class="language-java">
public void tiled(float[] a, float[] b, float[] c, int n) {
    final int bufferSize = 512;
    final int width = Math.min(n, bufferSize);
    final int height = Math.min(n, n >= 512 ? 8 : n >= 256 ? 16 : 32);
    float[] sum = new float[bufferSize];
    float[] vector = new float[bufferSize];
    for (int rowOffset = 0; rowOffset < n; rowOffset += height) {
      for (int columnOffset = 0; columnOffset < n; columnOffset += width) {
        for (int i = 0; i < n; ++i) {
          for (int j = columnOffset; j < columnOffset + width && j < n; j += width) {
            int stride = Math.min(n - columnOffset, bufferSize);
            // copy to give autovectorisation a hint
            System.arraycopy(c, i * n + j, sum, 0, stride);
            for (int k = rowOffset; k < rowOffset + height && k < n; ++k) {
              float multiplier = a[i * n + k];
              System.arraycopy(b, k * n  + j, vector, 0, stride);
              for (int l = 0; l < stride; ++l) {
                sum[l] = Math.fma(multiplier, vector[l], sum[l]);
              }
            }
            System.arraycopy(sum, 0, c, i * n + j, stride);
          }
        }
      }
    }
  }
</code>

Benchmarking it using the same <a href="https://github.com/richardstartin/simdbenchmarks/blob/master/src/main/java/com/openkappa/simd/mmm/MMM.java" rel="noopener" target="_blank">harness</a> used in the previous post, the performance is ~10% higher for large arrays than my previous best effort. Still, the reality is that this is too slow to be useful. If you need to do linear algebra, use C/C++ for the time being!

<img src="http://richardstartin.uk/wp-content/uploads/2018/01/Plot-64.png" alt="" width="1079" height="606" class="alignnone size-full wp-image-10427" />

<div class="table-holder">
<table class="table table-bordered table-hover table-condensed">
<thead><tr><th title="Field #1">Benchmark</th>
<th title="Field #2">Mode</th>
<th title="Field #3">Threads</th>
<th title="Field #4">Samples</th>
<th title="Field #5">Score</th>
<th title="Field #6">Score Error (99.9%)</th>
<th title="Field #7">Unit</th>
<th title="Field #8">Param: size</th>
<th title="Field #9">flops/cycle</th>
</tr></thead>
<tbody><tr>
<td>fastBuffered</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">53.331195</td>
<td align="right">0.270526</td>
<td>ops/s</td>
<td align="right">448</td>
<td align="right">3.688688696</td>
</tr>
<tr>
<td>fastBuffered</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">34.365765</td>
<td align="right">0.16641</td>
<td>ops/s</td>
<td align="right">512</td>
<td align="right">3.548072999</td>
</tr>
<tr>
<td>fastBuffered</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">26.128264</td>
<td align="right">0.239719</td>
<td>ops/s</td>
<td align="right">576</td>
<td align="right">3.840914622</td>
</tr>
<tr>
<td>fastBuffered</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">19.044509</td>
<td align="right">0.139197</td>
<td>ops/s</td>
<td align="right">640</td>
<td align="right">3.84031059</td>
</tr>
<tr>
<td>fastBuffered</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">14.312154</td>
<td align="right">1.045093</td>
<td>ops/s</td>
<td align="right">704</td>
<td align="right">3.841312378</td>
</tr>
<tr>
<td>fastBuffered</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">7.772745</td>
<td align="right">0.074598</td>
<td>ops/s</td>
<td align="right">768</td>
<td align="right">2.708411991</td>
</tr>
<tr>
<td>fastBuffered</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">6.276182</td>
<td align="right">0.067338</td>
<td>ops/s</td>
<td align="right">832</td>
<td align="right">2.780495238</td>
</tr>
<tr>
<td>fastBuffered</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">4.8784</td>
<td align="right">0.067368</td>
<td>ops/s</td>
<td align="right">896</td>
<td align="right">2.699343067</td>
</tr>
<tr>
<td>fastBuffered</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">4.068907</td>
<td align="right">0.038677</td>
<td>ops/s</td>
<td align="right">960</td>
<td align="right">2.769160387</td>
</tr>
<tr>
<td>fastBuffered</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">2.568101</td>
<td align="right">0.108612</td>
<td>ops/s</td>
<td align="right">1024</td>
<td align="right">2.121136502</td>
</tr>
<tr>
<td>tiled</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">56.495366</td>
<td align="right">0.584872</td>
<td>ops/s</td>
<td align="right">448</td>
<td align="right">3.907540754</td>
</tr>
<tr>
<td>tiled</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">30.884954</td>
<td align="right">3.221017</td>
<td>ops/s</td>
<td align="right">512</td>
<td align="right">3.188698735</td>
</tr>
<tr>
<td>tiled</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">15.580581</td>
<td align="right">0.412654</td>
<td>ops/s</td>
<td align="right">576</td>
<td align="right">2.290381075</td>
</tr>
<tr>
<td>tiled</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">9.178969</td>
<td align="right">0.841178</td>
<td>ops/s</td>
<td align="right">640</td>
<td align="right">1.850932038</td>
</tr>
<tr>
<td>tiled</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">12.229763</td>
<td align="right">0.350233</td>
<td>ops/s</td>
<td align="right">704</td>
<td align="right">3.282408783</td>
</tr>
<tr>
<td>tiled</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">9.371032</td>
<td align="right">0.330742</td>
<td>ops/s</td>
<td align="right">768</td>
<td align="right">3.265334889</td>
</tr>
<tr>
<td>tiled</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">7.727068</td>
<td align="right">0.277969</td>
<td>ops/s</td>
<td align="right">832</td>
<td align="right">3.423271628</td>
</tr>
<tr>
<td>tiled</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">6.076451</td>
<td align="right">0.30305</td>
<td>ops/s</td>
<td align="right">896</td>
<td align="right">3.362255222</td>
</tr>
<tr>
<td>tiled</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">4.916811</td>
<td align="right">0.2823</td>
<td>ops/s</td>
<td align="right">960</td>
<td align="right">3.346215151</td>
</tr>
<tr>
<td>tiled</td>
<td>thrpt</td>
<td>1</td>
<td align="right">10</td>
<td align="right">3.722623</td>
<td align="right">0.26486</td>
<td>ops/s</td>
<td align="right">1024</td>
<td align="right">3.074720008</td>
</tr>
</tbody></table>
</div>