# Два указателя (Two Pointers)

Техника использования двух указателей для эффективного обхода массива.

## Концепция

Вместо вложенных циклов O(n²), используем два указателя O(n).

```
Массив: [1, 2, 3, 4, 5]

Встречные указатели:
left →        ← right
[1, 2, 3, 4, 5]

Попутные указатели:
slow → fast →
[1, 2, 3, 4, 5]
```

## Типы паттернов

### 1. Встречные указатели (Opposite Direction)

Начинаются с концов, движутся навстречу.

#### Reverse Array

```go
func reverse(arr []int) {
    left, right := 0, len(arr)-1

    for left < right {
        arr[left], arr[right] = arr[right], arr[left]
        left++
        right--
    }
}

// reverse([1,2,3,4,5]) → [5,4,3,2,1]
```

#### Two Sum II (отсортированный массив)

```go
func twoSum(numbers []int, target int) []int {
    left, right := 0, len(numbers)-1

    for left < right {
        sum := numbers[left] + numbers[right]

        if sum == target {
            return []int{left + 1, right + 1}  // 1-indexed
        }

        if sum < target {
            left++   // Нужна большая сумма
        } else {
            right--  // Нужна меньшая сумма
        }
    }

    return nil
}

// twoSum([2,7,11,15], 9) → [1,2]
```

#### Valid Palindrome

```go
func isPalindrome(s string) bool {
    left, right := 0, len(s)-1

    for left < right {
        // Пропустить не-буквы
        for left < right && !isAlphanumeric(s[left]) {
            left++
        }
        for left < right && !isAlphanumeric(s[right]) {
            right--
        }

        if left < right {
            if toLower(s[left]) != toLower(s[right]) {
                return false
            }
            left++
            right--
        }
    }

    return true
}

func isAlphanumeric(ch byte) bool {
    return (ch >= 'a' && ch <= 'z') || (ch >= 'A' && ch <= 'Z') || (ch >= '0' && ch <= '9')
}

func toLower(ch byte) byte {
    if ch >= 'A' && ch <= 'Z' {
        return ch + 32
    }
    return ch
}

// isPalindrome("A man, a plan, a canal: Panama") → true
```

#### Container With Most Water

```go
func maxArea(height []int) int {
    left, right := 0, len(height)-1
    maxWater := 0

    for left < right {
        width := right - left
        minHeight := min(height[left], height[right])
        area := width * minHeight

        maxWater = max(maxWater, area)

        // Двигаем меньшую высоту
        if height[left] < height[right] {
            left++
        } else {
            right--
        }
    }

    return maxWater
}

// maxArea([1,8,6,2,5,4,8,3,7]) → 49
```

#### 3Sum

```go
func threeSum(nums []int) [][]int {
    sort.Ints(nums)
    result := [][]int{}

    for i := 0; i < len(nums)-2; i++ {
        // Пропустить дубликаты
        if i > 0 && nums[i] == nums[i-1] {
            continue
        }

        left, right := i+1, len(nums)-1
        target := -nums[i]

        for left < right {
            sum := nums[left] + nums[right]

            if sum == target {
                result = append(result, []int{nums[i], nums[left], nums[right]})

                // Пропустить дубликаты
                for left < right && nums[left] == nums[left+1] {
                    left++
                }
                for left < right && nums[right] == nums[right-1] {
                    right--
                }

                left++
                right--
            } else if sum < target {
                left++
            } else {
                right--
            }
        }
    }

    return result
}

// threeSum([-1,0,1,2,-1,-4]) → [[-1,-1,2],[-1,0,1]]
```

### 2. Быстрый и медленный указатели (Fast & Slow)

#### Remove Duplicates from Sorted Array

```go
func removeDuplicates(nums []int) int {
    if len(nums) == 0 {
        return 0
    }

    slow := 0

    for fast := 1; fast < len(nums); fast++ {
        if nums[fast] != nums[slow] {
            slow++
            nums[slow] = nums[fast]
        }
    }

    return slow + 1
}

// removeDuplicates([1,1,2,2,3]) → 3, nums=[1,2,3,...]
```

#### Move Zeroes

```go
func moveZeroes(nums []int) {
    slow := 0

    // Переместить все ненулевые элементы вперёд
    for fast := 0; fast < len(nums); fast++ {
        if nums[fast] != 0 {
            nums[slow] = nums[fast]
            slow++
        }
    }

    // Заполнить остаток нулями
    for i := slow; i < len(nums); i++ {
        nums[i] = 0
    }
}

// moveZeroes([0,1,0,3,12]) → [1,3,12,0,0]
```

#### Cycle Detection (Floyd's Algorithm)

```go
type ListNode struct {
    Val  int
    Next *ListNode
}

func hasCycle(head *ListNode) bool {
    if head == nil {
        return false
    }

    slow, fast := head, head

    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next

        if slow == fast {
            return true  // Цикл обнаружен
        }
    }

    return false
}
```

#### Find Cycle Start

```go
func detectCycle(head *ListNode) *ListNode {
    slow, fast := head, head

    // Найти встречу
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next

        if slow == fast {
            // Найти начало цикла
            slow = head
            for slow != fast {
                slow = slow.Next
                fast = fast.Next
            }
            return slow
        }
    }

    return nil
}
```

#### Middle of Linked List

```go
func middleNode(head *ListNode) *ListNode {
    slow, fast := head, head

    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }

    return slow  // Медленный указатель на середине
}
```

### 3. Раздвигающиеся указатели

#### Longest Palindromic Substring

```go
func longestPalindrome(s string) string {
    if len(s) == 0 {
        return ""
    }

    start, maxLen := 0, 0

    for i := 0; i < len(s); i++ {
        // Нечетная длина (центр - один символ)
        len1 := expandAroundCenter(s, i, i)

        // Четная длина (центр - два символа)
        len2 := expandAroundCenter(s, i, i+1)

        length := max(len1, len2)

        if length > maxLen {
            maxLen = length
            start = i - (length-1)/2
        }
    }

    return s[start : start+maxLen]
}

func expandAroundCenter(s string, left, right int) int {
    for left >= 0 && right < len(s) && s[left] == s[right] {
        left--
        right++
    }

    return right - left - 1
}

// longestPalindrome("babad") → "bab" или "aba"
```

## Partition Array

### Quick Select (nth smallest)

```go
func findKthLargest(nums []int, k int) int {
    return quickSelect(nums, 0, len(nums)-1, len(nums)-k)
}

func quickSelect(nums []int, left, right, k int) int {
    if left == right {
        return nums[left]
    }

    pivotIdx := partition(nums, left, right)

    if k == pivotIdx {
        return nums[k]
    } else if k < pivotIdx {
        return quickSelect(nums, left, pivotIdx-1, k)
    } else {
        return quickSelect(nums, pivotIdx+1, right, k)
    }
}

func partition(nums []int, left, right int) int {
    pivot := nums[right]
    i := left

    for j := left; j < right; j++ {
        if nums[j] <= pivot {
            nums[i], nums[j] = nums[j], nums[i]
            i++
        }
    }

    nums[i], nums[right] = nums[right], nums[i]
    return i
}
```

### Sort Colors (Dutch National Flag)

```go
func sortColors(nums []int) {
    left, mid, right := 0, 0, len(nums)-1

    for mid <= right {
        if nums[mid] == 0 {
            nums[left], nums[mid] = nums[mid], nums[left]
            left++
            mid++
        } else if nums[mid] == 1 {
            mid++
        } else {  // nums[mid] == 2
            nums[mid], nums[right] = nums[right], nums[mid]
            right--
        }
    }
}

// sortColors([2,0,2,1,1,0]) → [0,0,1,1,2,2]
```

## Сложность

| Паттерн | Time | Space |
|---------|------|-------|
| Two Pointers | O(n) | O(1) |
| Fast & Slow | O(n) | O(1) |
| Expand | O(n²) | O(1) |

## Когда использовать

**Two Pointers подходит когда:**
- Массив отсортирован
- Ищем пару элементов
- Нужно изменить массив in-place
- Ищем подмассив с условием

**Не подходит когда:**
- Массив не отсортирован (нужна HashMap)
- Нужны все комбинации

## Tips & Tricks

### 1. Отсортировать если нужно

```go
// Если не отсортирован, отсортируйте
sort.Ints(nums)
left, right := 0, len(nums)-1
```

### 2. Пропускать дубликаты

```go
// После нахождения ответа
for left < right && nums[left] == nums[left+1] {
    left++
}
for left < right && nums[right] == nums[right-1] {
    right--
}
```

### 3. Условия движения

```go
// Если сумма меньше target → увеличить left
// Если сумма больше target → уменьшить right
```

## Best Practices

1. ✅ Two pointers для отсортированных массивов
2. ✅ Fast & slow для linked list
3. ✅ Expand для палиндромов
4. ✅ Проверяйте границы (`left < right`)
5. ✅ Пропускайте дубликаты для уникальных результатов
6. ❌ Не используйте для неотсортированных (HashMap лучше)
7. ❌ Помните про граничные случаи

## Связанные темы

- [[Скользящее окно (Sliding Window)]]
- [[Сортировки - Быстрая, слиянием, пузырьком]]
- [[Связные списки]]
- [[Алгоритмическая сложность (Big O)]]
