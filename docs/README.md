# About

# welcome to my blog.

```html
<p>This is a paragraph</p>
<a href="//docsify.js.org/">Docsify</a>
```

```bash
echo "hello"
```

```php
function getAdder(int $x): int 
{
    return 123;
}
```

```go
func merge(nums1 []int, m int, nums2 []int, n int) {
    i := m - 1
    j := n - 1
    // 主题思路：ij两个指针倒着扫描，谁大要谁
    // 细节判断：i,j不能越界（一个<0，就要另一个）
    for k := m + n - 1; k >= 0; k-- {
        if j < 0 || (i >= 0 && nums1[i] >= nums2[j]) {
            nums1[k] = nums1[i]
            i--
        } else {
            nums1[k] = nums2[j]
            j--
        }
    }
}
```