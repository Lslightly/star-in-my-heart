---
tags:
  - Go
  - regex
  - simd
url: https://github.com/coregx/coregex
---

SIMD支持的regex表达式Go正则库。

## 示例

```go
var hasAVX2 = cpu.X86.HasAVX2
func Memchr(haystack []byte, needle byte) int {
	// Empty check
	if len(haystack) == 0 {
		return -1
	}

	// Use AVX2 implementation if available and input is large enough to amortize overhead.
	// For small inputs (< 32 bytes), the setup cost of SIMD outweighs the benefits.
	if hasAVX2 && len(haystack) >= 32 {
		return memchrAVX2(haystack, needle)
	}

	// Fallback to generic implementation for small inputs or non-AVX2 CPUs
	return memchrGeneric(haystack, needle)
}
```

在x86平台没有avx2时，回退到正常Go实现。

## 优化总结文档

总共有9个核心优化，相比于Rust包还要快，见[链接](https://github.com/coregx/coregex/blob/164c33d00431ead54b3d5386b55285b4679d82b2/docs/OPTIMIZATIONS.md)。

