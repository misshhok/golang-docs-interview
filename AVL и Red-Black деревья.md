# AVL и Red-Black деревья

Самобалансирующиеся бинарные деревья поиска, гарантирующие O(log n) операции.

## Проблема обычного BST

```
Вставить: 1, 2, 3, 4, 5

Результат:
1
 \
  2
   \
    3     ← Вырождается в список O(n)
     \
      4
       \
        5
```

**Решение:** автоматическая балансировка после каждой операции.

## AVL Дерево

**Изобретатели:** Adelson-Velsky и Landis (1962)

**Свойство:** Для каждого узла высота левого и правого поддеревьев отличается не более чем на 1.

```
Balance Factor (BF) = height(left) - height(right)

BF ∈ {-1, 0, 1}  ← Всегда!
```

### Структура

```go
type AVLNode struct {
    Val    int
    Left   *AVLNode
    Right  *AVLNode
    Height int  // Высота поддерева
}

type AVLTree struct {
    Root *AVLNode
}
```

### Вспомогательные функции

```go
func height(node *AVLNode) int {
    if node == nil {
        return 0
    }
    return node.Height
}

func updateHeight(node *AVLNode) {
    node.Height = 1 + max(height(node.Left), height(node.Right))
}

func balanceFactor(node *AVLNode) int {
    if node == nil {
        return 0
    }
    return height(node.Left) - height(node.Right)
}
```

### Повороты (Rotations)

#### Right Rotation (правый поворот)

```
     y               x
    / \             / \
   x   C    →      A   y
  / \                 / \
 A   B               B   C

Когда: BF(y) = 2, BF(x) = 1 (Left-Left случай)
```

```go
func rightRotate(y *AVLNode) *AVLNode {
    x := y.Left
    B := x.Right

    // Поворот
    x.Right = y
    y.Left = B

    // Обновить высоты
    updateHeight(y)
    updateHeight(x)

    return x  // Новый корень
}
```

#### Left Rotation (левый поворот)

```
   x                 y
  / \               / \
 A   y      →      x   C
    / \           / \
   B   C         A   B

Когда: BF(x) = -2, BF(y) = -1 (Right-Right случай)
```

```go
func leftRotate(x *AVLNode) *AVLNode {
    y := x.Right
    B := y.Left

    // Поворот
    y.Left = x
    x.Right = B

    // Обновить высоты
    updateHeight(x)
    updateHeight(y)

    return y  // Новый корень
}
```

#### Left-Right Rotation

```
     z               z               x
    / \             / \             / \
   y   D    →      x   D    →      y   z
  / \             / \             / \ / \
 A   x           y   C           A  B C  D
    / \         / \
   B   C       A   B

Когда: BF(z) = 2, BF(y) = -1
```

```go
func leftRightRotate(z *AVLNode) *AVLNode {
    z.Left = leftRotate(z.Left)   // Левый поворот на y
    return rightRotate(z)          // Правый поворот на z
}
```

#### Right-Left Rotation

```
   x               x               y
  / \             / \             / \
 A   z    →      A   y    →      x   z
    / \             / \         / \ / \
   y   D           B   z       A  B C  D
  / \                 / \
 B   C               C   D

Когда: BF(x) = -2, BF(z) = 1
```

```go
func rightLeftRotate(x *AVLNode) *AVLNode {
    x.Right = rightRotate(x.Right)  // Правый поворот на z
    return leftRotate(x)             // Левый поворот на x
}
```

### Вставка

```go
func (t *AVLTree) Insert(val int) {
    t.Root = insert(t.Root, val)
}

func insert(node *AVLNode, val int) *AVLNode {
    // 1. Обычная BST вставка
    if node == nil {
        return &AVLNode{Val: val, Height: 1}
    }

    if val < node.Val {
        node.Left = insert(node.Left, val)
    } else if val > node.Val {
        node.Right = insert(node.Right, val)
    } else {
        return node  // Дубликаты не вставляем
    }

    // 2. Обновить высоту
    updateHeight(node)

    // 3. Проверить баланс
    bf := balanceFactor(node)

    // 4. Балансировка (4 случая)

    // Left-Left
    if bf > 1 && val < node.Left.Val {
        return rightRotate(node)
    }

    // Right-Right
    if bf < -1 && val > node.Right.Val {
        return leftRotate(node)
    }

    // Left-Right
    if bf > 1 && val > node.Left.Val {
        node.Left = leftRotate(node.Left)
        return rightRotate(node)
    }

    // Right-Left
    if bf < -1 && val < node.Right.Val {
        node.Right = rightRotate(node.Right)
        return leftRotate(node)
    }

    return node
}
```

### Удаление

```go
func (t *AVLTree) Delete(val int) {
    t.Root = deleteNode(t.Root, val)
}

func deleteNode(node *AVLNode, val int) *AVLNode {
    // 1. Обычное BST удаление
    if node == nil {
        return nil
    }

    if val < node.Val {
        node.Left = deleteNode(node.Left, val)
    } else if val > node.Val {
        node.Right = deleteNode(node.Right, val)
    } else {
        // Узел найден
        if node.Left == nil {
            return node.Right
        }
        if node.Right == nil {
            return node.Left
        }

        // Два ребёнка: inorder successor
        successor := findMin(node.Right)
        node.Val = successor.Val
        node.Right = deleteNode(node.Right, successor.Val)
    }

    if node == nil {
        return nil
    }

    // 2. Обновить высоту
    updateHeight(node)

    // 3. Балансировка
    bf := balanceFactor(node)

    // Left-Left
    if bf > 1 && balanceFactor(node.Left) >= 0 {
        return rightRotate(node)
    }

    // Left-Right
    if bf > 1 && balanceFactor(node.Left) < 0 {
        node.Left = leftRotate(node.Left)
        return rightRotate(node)
    }

    // Right-Right
    if bf < -1 && balanceFactor(node.Right) <= 0 {
        return leftRotate(node)
    }

    // Right-Left
    if bf < -1 && balanceFactor(node.Right) > 0 {
        node.Right = rightRotate(node.Right)
        return leftRotate(node)
    }

    return node
}

func findMin(node *AVLNode) *AVLNode {
    for node.Left != nil {
        node = node.Left
    }
    return node
}
```

## Red-Black Tree (Красно-чёрное дерево)

**Более гибкая балансировка** чем AVL.

### Свойства

1. Каждый узел - красный или чёрный
2. Корень - чёрный
3. Все листья (nil) - чёрные
4. Красный узел не может иметь красного родителя
5. Все пути от узла до листьев содержат одинаковое количество чёрных узлов

```
       B(10)
      /     \
   R(5)    B(20)
   / \      / \
B(3) B(7) R(15) R(25)
```

### Структура

```go
type Color bool

const (
    Red   Color = false
    Black Color = true
)

type RBNode struct {
    Val    int
    Color  Color
    Left   *RBNode
    Right  *RBNode
    Parent *RBNode
}

type RBTree struct {
    Root *RBNode
    NIL  *RBNode  // Sentinel для листьев
}

func NewRBTree() *RBTree {
    nil := &RBNode{Color: Black}
    return &RBTree{
        Root: nil,
        NIL:  nil,
    }
}
```

### Вставка (упрощенная)

```go
func (t *RBTree) Insert(val int) {
    node := &RBNode{
        Val:   val,
        Color: Red,  // Всегда вставляем красным!
        Left:  t.NIL,
        Right: t.NIL,
    }

    // BST вставка
    parent := t.NIL
    current := t.Root

    for current != t.NIL {
        parent = current
        if val < current.Val {
            current = current.Left
        } else {
            current = current.Right
        }
    }

    node.Parent = parent

    if parent == t.NIL {
        t.Root = node
    } else if val < parent.Val {
        parent.Left = node
    } else {
        parent.Right = node
    }

    // Фиксим нарушения
    t.insertFixup(node)
}

func (t *RBTree) insertFixup(node *RBNode) {
    for node.Parent.Color == Red {
        if node.Parent == node.Parent.Parent.Left {
            uncle := node.Parent.Parent.Right

            // Случай 1: дядя красный
            if uncle.Color == Red {
                node.Parent.Color = Black
                uncle.Color = Black
                node.Parent.Parent.Color = Red
                node = node.Parent.Parent
            } else {
                // Случай 2: node - правый ребёнок
                if node == node.Parent.Right {
                    node = node.Parent
                    t.leftRotate(node)
                }

                // Случай 3: node - левый ребёнок
                node.Parent.Color = Black
                node.Parent.Parent.Color = Red
                t.rightRotate(node.Parent.Parent)
            }
        } else {
            // Зеркальные случаи
            // ... (аналогично, но left ↔ right)
        }
    }

    t.Root.Color = Black
}
```

### Повороты для RB-дерева

```go
func (t *RBTree) leftRotate(x *RBNode) {
    y := x.Right
    x.Right = y.Left

    if y.Left != t.NIL {
        y.Left.Parent = x
    }

    y.Parent = x.Parent

    if x.Parent == t.NIL {
        t.Root = y
    } else if x == x.Parent.Left {
        x.Parent.Left = y
    } else {
        x.Parent.Right = y
    }

    y.Left = x
    x.Parent = y
}

func (t *RBTree) rightRotate(y *RBNode) {
    x := y.Left
    y.Left = x.Right

    if x.Right != t.NIL {
        x.Right.Parent = y
    }

    x.Parent = y.Parent

    if y.Parent == t.NIL {
        t.Root = x
    } else if y == y.Parent.Right {
        y.Parent.Right = x
    } else {
        y.Parent.Left = x
    }

    x.Right = y
    y.Parent = x
}
```

## AVL vs Red-Black

| Свойство | AVL | Red-Black |
|----------|-----|-----------|
| Балансировка | Строгая (BF ≤ 1) | Гибкая |
| Высота | ~1.44 log n | ~2 log n |
| Поиск | Быстрее | Медленнее |
| Вставка | Медленнее (больше поворотов) | Быстрее |
| Удаление | Медленнее | Быстрее |
| Память | +1 int (height) | +1 bit (color) |

**AVL для:**
- Много поисков
- Редкие вставки/удаления
- Нужна максимальная скорость поиска

**Red-Black для:**
- Много вставок/удалений
- Поиск не критичен
- Используется в map в C++, Java, Linux kernel

## Сравнение структур данных

| Операция | BST (worst) | AVL | Red-Black | Hash Table |
|----------|-------------|-----|-----------|------------|
| Поиск | O(n) | O(log n) | O(log n) | O(1)* |
| Вставка | O(n) | O(log n) | O(log n) | O(1)* |
| Удаление | O(n) | O(log n) | O(log n) | O(1)* |
| Min/Max | O(n) | O(log n) | O(log n) | O(n) |
| Range | O(n) | O(log n + k) | O(log n + k) | - |
| Упорядоченность | ✅ | ✅ | ✅ | ❌ |

\* Амортизированное

## Применение в реальности

### Go

```go
// Нет встроенного AVL/RB-дерева в Go
// Можно использовать сторонние библиотеки:

// github.com/emirpasic/gods
import "github.com/emirpasic/gods/trees/avltree"

tree := avltree.NewWithIntComparator()
tree.Put(5, "e")
tree.Put(3, "c")
tree.Put(7, "g")

value, found := tree.Get(3)  // "c", true
tree.Remove(3)
```

### C++ STL

```cpp
#include <map>   // Red-Black дерево
#include <set>

std::map<int, string> m;
m[5] = "five";

std::set<int> s;
s.insert(5);
```

### Java

```java
import java.util.TreeMap;  // Red-Black дерево
import java.util.TreeSet;

TreeMap<Integer, String> map = new TreeMap<>();
map.put(5, "five");

TreeSet<Integer> set = new TreeSet<>();
set.add(5);
```

## Best Practices

1. ✅ Используйте AVL для read-heavy приложений
2. ✅ Используйте Red-Black для write-heavy
3. ✅ Hash Table для точного поиска без порядка
4. ✅ Сторонние библиотеки вместо своей реализации
5. ❌ Не реализуйте сами для production (сложно!)
6. ❌ Не используйте если не нужна упорядоченность

## Когда использовать

**Самобалансирующиеся деревья нужны когда:**
- Данные должны быть упорядочены
- Range queries
- Нужен k-й элемент
- Predecessor/Successor
- Гарантия O(log n) в худшем случае

**Альтернативы:**
- Hash Table - для точного поиска
- Обычный slice + sort - для небольших данных
- B-Tree - для баз данных

## Связанные темы

- [[Бинарные деревья поиска]]
- [[Деревья - Основы]]
- [[HashMap - Реализация и особенности]]
- [[Алгоритмическая сложность (Big O)]]
