# BFS и DFS

Два основных алгоритма обхода графов и деревьев.

## Breadth-First Search (BFS)

**Поиск в ширину** - обходит уровень за уровнем.

```
Дерево:
      1
     / \
    2   3
   / \
  4   5

BFS: 1 → 2 → 3 → 4 → 5 (по уровням)
```

### Реализация для дерева

```go
func bfs(root *TreeNode) []int {
    if root == nil {
        return []int{}
    }

    result := []int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]

        result = append(result, node.Val)

        if node.Left != nil {
            queue = append(queue, node.Left)
        }
        if node.Right != nil {
            queue = append(queue, node.Right)
        }
    }

    return result
}
```

### Реализация для графа

```go
func bfsGraph(graph map[int][]int, start int) []int {
    visited := make(map[int]bool)
    queue := []int{start}
    visited[start] = true
    result := []int{}

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]

        result = append(result, node)

        for _, neighbor := range graph[node] {
            if !visited[neighbor] {
                visited[neighbor] = true
                queue = append(queue, neighbor)
            }
        }
    }

    return result
}

// graph = {0: [1, 2], 1: [0, 3], 2: [0, 3], 3: [1, 2]}
// bfsGraph(graph, 0) → [0, 1, 2, 3]
```

### Level Order Traversal

```go
func levelOrder(root *TreeNode) [][]int {
    if root == nil {
        return [][]int{}
    }

    result := [][]int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        levelSize := len(queue)
        level := []int{}

        for i := 0; i < levelSize; i++ {
            node := queue[0]
            queue = queue[1:]

            level = append(level, node.Val)

            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }

        result = append(result, level)
    }

    return result
}

// levelOrder(root) → [[1], [2,3], [4,5]]
```

## Depth-First Search (DFS)

**Поиск в глубину** - идёт до конца, потом backtrack.

```
Дерево:
      1
     / \
    2   3
   / \
  4   5

DFS: 1 → 2 → 4 → 5 → 3 (в глубину)
```

### Рекурсивная реализация (дерево)

```go
func dfs(root *TreeNode) []int {
    result := []int{}
    dfsHelper(root, &result)
    return result
}

func dfsHelper(node *TreeNode, result *[]int) {
    if node == nil {
        return
    }

    *result = append(*result, node.Val)  // Preorder
    dfsHelper(node.Left, result)
    dfsHelper(node.Right, result)
}
```

### Итеративная реализация (дерево)

```go
func dfsIterative(root *TreeNode) []int {
    if root == nil {
        return []int{}
    }

    result := []int{}
    stack := []*TreeNode{root}

    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        result = append(result, node.Val)

        // Сначала правый (чтобы левый обработался раньше)
        if node.Right != nil {
            stack = append(stack, node.Right)
        }
        if node.Left != nil {
            stack = append(stack, node.Left)
        }
    }

    return result
}
```

### Реализация для графа

```go
func dfsGraph(graph map[int][]int, start int) []int {
    visited := make(map[int]bool)
    result := []int{}

    var dfs func(int)
    dfs = func(node int) {
        if visited[node] {
            return
        }

        visited[node] = true
        result = append(result, node)

        for _, neighbor := range graph[node] {
            dfs(neighbor)
        }
    }

    dfs(start)
    return result
}
```

### Итеративный DFS для графа

```go
func dfsGraphIterative(graph map[int][]int, start int) []int {
    visited := make(map[int]bool)
    stack := []int{start}
    result := []int{}

    for len(stack) > 0 {
        node := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if visited[node] {
            continue
        }

        visited[node] = true
        result = append(result, node)

        // Добавляем соседей в обратном порядке
        neighbors := graph[node]
        for i := len(neighbors) - 1; i >= 0; i-- {
            if !visited[neighbors[i]] {
                stack = append(stack, neighbors[i])
            }
        }
    }

    return result
}
```

## Сравнение BFS vs DFS

| Свойство | BFS | DFS |
|----------|-----|-----|
| Структура | Queue | Stack (рекурсия) |
| Порядок | По уровням | В глубину |
| Кратчайший путь | ✅ (невзвешенный граф) | ❌ |
| Память | O(ширина) | O(высота) |
| Обнаружение цикла | Да | Да |

## Применение BFS

### 1. Кратчайший путь в невзвешенном графе

```go
func shortestPath(graph map[int][]int, start, end int) int {
    if start == end {
        return 0
    }

    visited := make(map[int]bool)
    queue := []int{start}
    visited[start] = true
    distance := 0

    for len(queue) > 0 {
        levelSize := len(queue)
        distance++

        for i := 0; i < levelSize; i++ {
            node := queue[0]
            queue = queue[1:]

            for _, neighbor := range graph[node] {
                if neighbor == end {
                    return distance
                }

                if !visited[neighbor] {
                    visited[neighbor] = true
                    queue = append(queue, neighbor)
                }
            }
        }
    }

    return -1  // Нет пути
}
```

### 2. Number of Islands (LeetCode 200)

```go
func numIslands(grid [][]byte) int {
    if len(grid) == 0 {
        return 0
    }

    rows, cols := len(grid), len(grid[0])
    count := 0

    for i := 0; i < rows; i++ {
        for j := 0; j < cols; j++ {
            if grid[i][j] == '1' {
                bfsIsland(grid, i, j)
                count++
            }
        }
    }

    return count
}

func bfsIsland(grid [][]byte, startRow, startCol int) {
    rows, cols := len(grid), len(grid[0])
    queue := [][2]int{{startRow, startCol}}
    grid[startRow][startCol] = '0'

    dirs := [][2]int{{-1, 0}, {1, 0}, {0, -1}, {0, 1}}

    for len(queue) > 0 {
        cell := queue[0]
        queue = queue[1:]
        row, col := cell[0], cell[1]

        for _, dir := range dirs {
            newRow, newCol := row+dir[0], col+dir[1]

            if newRow >= 0 && newRow < rows &&
                newCol >= 0 && newCol < cols &&
                grid[newRow][newCol] == '1' {

                grid[newRow][newCol] = '0'
                queue = append(queue, [2]int{newRow, newCol})
            }
        }
    }
}
```

### 3. Binary Tree Right Side View

```go
func rightSideView(root *TreeNode) []int {
    if root == nil {
        return []int{}
    }

    result := []int{}
    queue := []*TreeNode{root}

    for len(queue) > 0 {
        levelSize := len(queue)

        for i := 0; i < levelSize; i++ {
            node := queue[0]
            queue = queue[1:]

            // Последний элемент уровня
            if i == levelSize-1 {
                result = append(result, node.Val)
            }

            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
    }

    return result
}
```

## Применение DFS

### 1. Path Sum (все пути)

```go
func pathSum(root *TreeNode, targetSum int) [][]int {
    result := [][]int{}
    path := []int{}

    var dfs func(*TreeNode, int)
    dfs = func(node *TreeNode, sum int) {
        if node == nil {
            return
        }

        path = append(path, node.Val)
        sum += node.Val

        // Лист
        if node.Left == nil && node.Right == nil {
            if sum == targetSum {
                // Копировать path
                temp := make([]int, len(path))
                copy(temp, path)
                result = append(result, temp)
            }
        } else {
            dfs(node.Left, sum)
            dfs(node.Right, sum)
        }

        // Backtrack
        path = path[:len(path)-1]
    }

    dfs(root, 0)
    return result
}
```

### 2. Validate Binary Search Tree

```go
func isValidBST(root *TreeNode) bool {
    return validate(root, nil, nil)
}

func validate(node *TreeNode, min, max *int) bool {
    if node == nil {
        return true
    }

    if min != nil && node.Val <= *min {
        return false
    }
    if max != nil && node.Val >= *max {
        return false
    }

    return validate(node.Left, min, &node.Val) &&
        validate(node.Right, &node.Val, max)
}
```

### 3. Clone Graph (LeetCode 133)

```go
type Node struct {
    Val       int
    Neighbors []*Node
}

func cloneGraph(node *Node) *Node {
    if node == nil {
        return nil
    }

    visited := make(map[*Node]*Node)

    var dfs func(*Node) *Node
    dfs = func(node *Node) *Node {
        if clone, ok := visited[node]; ok {
            return clone
        }

        clone := &Node{Val: node.Val}
        visited[node] = clone

        for _, neighbor := range node.Neighbors {
            clone.Neighbors = append(clone.Neighbors, dfs(neighbor))
        }

        return clone
    }

    return dfs(node)
}
```

### 4. Course Schedule (Cycle Detection)

```go
func canFinish(numCourses int, prerequisites [][]int) bool {
    // Построить граф
    graph := make(map[int][]int)
    for _, prereq := range prerequisites {
        course, pre := prereq[0], prereq[1]
        graph[course] = append(graph[course], pre)
    }

    visited := make(map[int]int)  // 0=не посещен, 1=в пути, 2=завершен

    var hasCycle func(int) bool
    hasCycle = func(course int) bool {
        if visited[course] == 1 {
            return true  // Цикл!
        }
        if visited[course] == 2 {
            return false  // Уже проверен
        }

        visited[course] = 1  // Отметить в пути

        for _, prereq := range graph[course] {
            if hasCycle(prereq) {
                return true
            }
        }

        visited[course] = 2  // Завершен
        return false
    }

    for i := 0; i < numCourses; i++ {
        if hasCycle(i) {
            return false
        }
    }

    return true
}
```

## Топологическая сортировка

### Kahn's Algorithm (BFS)

```go
func topologicalSort(n int, edges [][]int) []int {
    graph := make(map[int][]int)
    inDegree := make([]int, n)

    // Построить граф
    for _, edge := range edges {
        from, to := edge[0], edge[1]
        graph[from] = append(graph[from], to)
        inDegree[to]++
    }

    // Найти узлы с inDegree = 0
    queue := []int{}
    for i := 0; i < n; i++ {
        if inDegree[i] == 0 {
            queue = append(queue, i)
        }
    }

    result := []int{}

    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        result = append(result, node)

        for _, neighbor := range graph[node] {
            inDegree[neighbor]--
            if inDegree[neighbor] == 0 {
                queue = append(queue, neighbor)
            }
        }
    }

    if len(result) != n {
        return []int{}  // Есть цикл
    }

    return result
}
```

### DFS-based Topological Sort

```go
func topologicalSortDFS(n int, edges [][]int) []int {
    graph := make(map[int][]int)
    for _, edge := range edges {
        from, to := edge[0], edge[1]
        graph[from] = append(graph[from], to)
    }

    visited := make([]bool, n)
    stack := []int{}

    var dfs func(int)
    dfs = func(node int) {
        visited[node] = true

        for _, neighbor := range graph[node] {
            if !visited[neighbor] {
                dfs(neighbor)
            }
        }

        stack = append(stack, node)
    }

    for i := 0; i < n; i++ {
        if !visited[i] {
            dfs(i)
        }
    }

    // Развернуть stack
    for i, j := 0, len(stack)-1; i < j; i, j = i+1, j-1 {
        stack[i], stack[j] = stack[j], stack[i]
    }

    return stack
}
```

## Сложность

| Алгоритм | Time | Space |
|----------|------|-------|
| BFS | O(V + E) | O(V) |
| DFS | O(V + E) | O(V) |

где V = вершины, E = рёбра

## Когда использовать

**BFS:**
- Кратчайший путь (невзвешенный граф)
- Level order traversal
- Ближайшие соседи
- Все узлы на расстоянии k

**DFS:**
- Поиск пути
- Обнаружение циклов
- Топологическая сортировка
- Connected components
- Backtracking задачи

## Best Practices

1. ✅ BFS для кратчайших путей
2. ✅ DFS для обнаружения циклов
3. ✅ Всегда используйте visited для графов
4. ✅ Рекурсия для деревьев, итерация для графов
5. ✅ Проверяйте границы для матриц
6. ❌ Не забывайте обрабатывать disconnected компоненты
7. ❌ Остерегайтесь stack overflow в DFS

## Связанные темы

- [[Графы - Представление]]
- [[Деревья - Основы]]
- [[Стек и очередь]]
- [[Алгоритмическая сложность (Big O)]]
