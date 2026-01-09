# Скользящее окно (Sliding Window)

Техника для задач с подмассивами или подстроками фиксированной или динамической длины.

## Концепция

Вместо проверки всех подмассивов O(n²), скользим окно по массиву O(n).

```
Массив: [1, 2, 3, 4, 5]
Размер окна: 3

Окно 1: [1, 2, 3] → sum = 6
Окно 2:    [2, 3, 4] → sum = 9
Окно 3:       [3, 4, 5] → sum = 12
```

## Типы окон

### 1. Фиксированный размер окна

Окно всегда размера k.

#### Maximum Sum Subarray of Size K

```go
func maxSumSubarray(arr []int, k int) int {
    if len(arr) < k {
        return 0
    }

    // Первое окно
    windowSum := 0
    for i := 0; i < k; i++ {
        windowSum += arr[i]
    }

    maxSum := windowSum

    // Скользящее окно
    for i := k; i < len(arr); i++ {
        windowSum = windowSum - arr[i-k] + arr[i]  // Убрать левый, добавить правый
        maxSum = max(maxSum, windowSum)
    }

    return maxSum
}

// maxSumSubarray([2,1,5,1,3,2], 3) → 9 ([5,1,3])
```

#### Average of Subarrays of Size K

```go
func findAverages(arr []int, k int) []float64 {
    result := []float64{}
    windowSum := 0

    for i := 0; i < len(arr); i++ {
        windowSum += arr[i]

        if i >= k-1 {
            result = append(result, float64(windowSum)/float64(k))
            windowSum -= arr[i-k+1]
        }
    }

    return result
}

// findAverages([1,3,2,6,-1,4,1,8,2], 5) → [2.2, 2.8, 2.4, 3.6, 2.8]
```

### 2. Динамический размер окна

Размер окна изменяется в зависимости от условия.

#### Longest Substring Without Repeating Characters

```go
func lengthOfLongestSubstring(s string) int {
    charMap := make(map[byte]int)
    left := 0
    maxLen := 0

    for right := 0; right < len(s); right++ {
        ch := s[right]

        // Если символ повторяется, сдвинуть left
        if idx, ok := charMap[ch]; ok && idx >= left {
            left = idx + 1
        }

        charMap[ch] = right
        maxLen = max(maxLen, right-left+1)
    }

    return maxLen
}

// lengthOfLongestSubstring("abcabcbb") → 3 ("abc")
// lengthOfLongestSubstring("bbbbb") → 1 ("b")
```

#### Minimum Window Substring (LeetCode 76)

```go
func minWindow(s string, t string) string {
    if len(s) == 0 || len(t) == 0 {
        return ""
    }

    // Подсчитать частоты в t
    tCount := make(map[byte]int)
    for i := 0; i < len(t); i++ {
        tCount[t[i]]++
    }

    required := len(tCount)
    formed := 0

    windowCounts := make(map[byte]int)
    left, right := 0, 0

    // [minLen, left, right]
    ans := []int{-1, 0, 0}

    for right < len(s) {
        ch := s[right]
        windowCounts[ch]++

        if count, ok := tCount[ch]; ok && windowCounts[ch] == count {
            formed++
        }

        // Сжать окно
        for left <= right && formed == required {
            ch := s[left]

            // Обновить результат
            if ans[0] == -1 || right-left+1 < ans[0] {
                ans[0] = right - left + 1
                ans[1] = left
                ans[2] = right
            }

            windowCounts[ch]--
            if count, ok := tCount[ch]; ok && windowCounts[ch] < count {
                formed--
            }

            left++
        }

        right++
    }

    if ans[0] == -1 {
        return ""
    }

    return s[ans[1] : ans[2]+1]
}

// minWindow("ADOBECODEBANC", "ABC") → "BANC"
```

#### Longest Substring with At Most K Distinct Characters

```go
func lengthOfLongestSubstringKDistinct(s string, k int) int {
    if k == 0 || len(s) == 0 {
        return 0
    }

    charCount := make(map[byte]int)
    left, maxLen := 0, 0

    for right := 0; right < len(s); right++ {
        charCount[s[right]]++

        // Сжать окно если > k distinct
        for len(charCount) > k {
            charCount[s[left]]--
            if charCount[s[left]] == 0 {
                delete(charCount, s[left])
            }
            left++
        }

        maxLen = max(maxLen, right-left+1)
    }

    return maxLen
}

// lengthOfLongestSubstringKDistinct("eceba", 2) → 3 ("ece")
```

#### Max Consecutive Ones III (LeetCode 1004)

```go
func longestOnes(nums []int, k int) int {
    left, zeros, maxLen := 0, 0, 0

    for right := 0; right < len(nums); right++ {
        if nums[right] == 0 {
            zeros++
        }

        // Сжать окно если слишком много нулей
        for zeros > k {
            if nums[left] == 0 {
                zeros--
            }
            left++
        }

        maxLen = max(maxLen, right-left+1)
    }

    return maxLen
}

// longestOnes([1,1,1,0,0,0,1,1,1,1,0], 2) → 6
```

### 3. Окно с условием на сумму/произведение

#### Subarray Product Less Than K

```go
func numSubarrayProductLessThanK(nums []int, k int) int {
    if k <= 1 {
        return 0
    }

    product := 1
    left, count := 0, 0

    for right := 0; right < len(nums); right++ {
        product *= nums[right]

        // Сжать окно если product >= k
        for product >= k {
            product /= nums[left]
            left++
        }

        // Все подмассивы заканчивающиеся на right
        count += right - left + 1
    }

    return count
}

// numSubarrayProductLessThanK([10,5,2,6], 100) → 8
```

#### Minimum Size Subarray Sum

```go
func minSubArrayLen(target int, nums []int) int {
    left, sum := 0, 0
    minLen := len(nums) + 1

    for right := 0; right < len(nums); right++ {
        sum += nums[right]

        // Сжать окно пока sum >= target
        for sum >= target {
            minLen = min(minLen, right-left+1)
            sum -= nums[left]
            left++
        }
    }

    if minLen == len(nums)+1 {
        return 0
    }

    return minLen
}

// minSubArrayLen(7, [2,3,1,2,4,3]) → 2 ([4,3])
```

## Шаблон Sliding Window

```go
func slidingWindow(arr []int) int {
    left, result := 0, 0
    window := make(map[int]int)  // Или другая структура

    for right := 0; right < len(arr); right++ {
        // Добавить arr[right] в окно
        window[arr[right]]++

        // Сжать окно если нарушено условие
        for windowIsInvalid() {
            window[arr[left]]--
            if window[arr[left]] == 0 {
                delete(window, arr[left])
            }
            left++
        }

        // Обновить результат
        result = max(result, right-left+1)
    }

    return result
}
```

## Примеры задач

### Permutation in String (LeetCode 567)

```go
func checkInclusion(s1 string, s2 string) bool {
    if len(s1) > len(s2) {
        return false
    }

    s1Count := make(map[byte]int)
    for i := 0; i < len(s1); i++ {
        s1Count[s1[i]]++
    }

    windowCount := make(map[byte]int)
    matches := 0
    required := len(s1Count)

    for right := 0; right < len(s2); right++ {
        ch := s2[right]
        windowCount[ch]++

        if windowCount[ch] == s1Count[ch] {
            matches++
        }

        // Фиксированное окно размера len(s1)
        if right >= len(s1) {
            leftCh := s2[right-len(s1)]
            if windowCount[leftCh] == s1Count[leftCh] {
                matches--
            }
            windowCount[leftCh]--
        }

        if matches == required {
            return true
        }
    }

    return false
}

// checkInclusion("ab", "eidbaooo") → true
```

### Fruit Into Baskets (LeetCode 904)

```go
func totalFruit(fruits []int) int {
    basket := make(map[int]int)
    left, maxFruits := 0, 0

    for right := 0; right < len(fruits); right++ {
        basket[fruits[right]]++

        // Максимум 2 типа фруктов
        for len(basket) > 2 {
            basket[fruits[left]]--
            if basket[fruits[left]] == 0 {
                delete(basket, fruits[left])
            }
            left++
        }

        maxFruits = max(maxFruits, right-left+1)
    }

    return maxFruits
}

// totalFruit([1,2,1]) → 3
// totalFruit([0,1,2,2]) → 3
```

### Longest Repeating Character Replacement

```go
func characterReplacement(s string, k int) int {
    charCount := make(map[byte]int)
    left, maxCount, maxLen := 0, 0, 0

    for right := 0; right < len(s); right++ {
        charCount[s[right]]++
        maxCount = max(maxCount, charCount[s[right]])

        // Если нужно заменить больше k символов
        if right-left+1-maxCount > k {
            charCount[s[left]]--
            left++
        }

        maxLen = max(maxLen, right-left+1)
    }

    return maxLen
}

// characterReplacement("AABABBA", 1) → 4 ("AABA")
```

## Сложность

| Операция | Time | Space |
|----------|------|-------|
| Fixed Window | O(n) | O(1) |
| Dynamic Window | O(n) | O(k) |
| With HashMap | O(n) | O(k) |

где k = размер алфавита или уникальных элементов

## Когда использовать

**Sliding Window подходит когда:**
- Ищем подмассив/подстроку
- Нужна максимальная/минимальная длина
- Условие на окно (сумма, частота, distinct)
- Задачи с "longest", "shortest", "maximum", "minimum"

**Признаки задачи:**
- "Longest substring..."
- "Maximum sum subarray..."
- "Minimum window..."
- "At most K distinct..."

## Best Practices

1. ✅ HashMap для частот символов
2. ✅ Два указателя (left, right)
3. ✅ Сжимайте окно когда условие нарушено
4. ✅ Обновляйте результат после сжатия
5. ✅ Фиксированное окно проще динамического
6. ❌ Не забывайте удалять из HashMap при count=0
7. ❌ Проверяйте граничные случаи (пустая строка, k=0)

## Связанные темы

- [[Два указателя (Two Pointers)]]
- [[HashMap - Реализация и особенности]]
- [[Алгоритмы на строках]]
- [[Алгоритмическая сложность (Big O)]]
