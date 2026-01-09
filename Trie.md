# Trie

Trie (префиксное дерево) - древовидная структура данных для хранения строк с общими префиксами.

## Концепция

```
Слова: "cat", "car", "card", "care", "dog", "dodge"

        root
       /    \
      c      d
      |      |
      a      o
     / \     |
    t   r    g
        |     \
        d(e)   e
        |
        (e)

() - конец слова
```

Каждый узел представляет символ. Путь от корня до узла - префикс.

## Структура

```go
type TrieNode struct {
    children map[rune]*TrieNode
    isEnd    bool  // Конец слова
}

type Trie struct {
    root *TrieNode
}

func NewTrie() *Trie {
    return &Trie{
        root: &TrieNode{
            children: make(map[rune]*TrieNode),
        },
    }
}
```

## Операции

### Insert (вставка)

```go
func (t *Trie) Insert(word string) {
    node := t.root

    for _, ch := range word {
        if _, ok := node.children[ch]; !ok {
            node.children[ch] = &TrieNode{
                children: make(map[rune]*TrieNode),
            }
        }
        node = node.children[ch]
    }

    node.isEnd = true
}

// Пример
trie := NewTrie()
trie.Insert("cat")
trie.Insert("car")
trie.Insert("card")
```

**Сложность:** O(m), где m - длина слова

### Search (поиск точного совпадения)

```go
func (t *Trie) Search(word string) bool {
    node := t.root

    for _, ch := range word {
        if _, ok := node.children[ch]; !ok {
            return false
        }
        node = node.children[ch]
    }

    return node.isEnd
}

// trie.Search("cat") → true
// trie.Search("ca") → false (не isEnd)
// trie.Search("catch") → false
```

**Сложность:** O(m)

### StartsWith (поиск префикса)

```go
func (t *Trie) StartsWith(prefix string) bool {
    node := t.root

    for _, ch := range prefix {
        if _, ok := node.children[ch]; !ok {
            return false
        }
        node = node.children[ch]
    }

    return true  // Префикс найден (isEnd не важен)
}

// trie.StartsWith("ca") → true
// trie.StartsWith("car") → true
// trie.StartsWith("cat") → true
// trie.StartsWith("dog") → false
```

**Сложность:** O(m)

### Delete (удаление)

```go
func (t *Trie) Delete(word string) bool {
    return deleteHelper(t.root, word, 0)
}

func deleteHelper(node *TrieNode, word string, index int) bool {
    if node == nil {
        return false
    }

    // Дошли до конца слова
    if index == len(word) {
        if !node.isEnd {
            return false  // Слово не существует
        }

        node.isEnd = false

        // Можно удалить узел если нет детей
        return len(node.children) == 0
    }

    ch := rune(word[index])
    child, ok := node.children[ch]
    if !ok {
        return false
    }

    shouldDeleteChild := deleteHelper(child, word, index+1)

    if shouldDeleteChild {
        delete(node.children, ch)

        // Можно удалить текущий узел если:
        // - Не конец другого слова
        // - Нет других детей
        return !node.isEnd && len(node.children) == 0
    }

    return false
}
```

### Поиск всех слов с префиксом

```go
func (t *Trie) FindWordsWithPrefix(prefix string) []string {
    node := t.root

    // Найти узел префикса
    for _, ch := range prefix {
        if _, ok := node.children[ch]; !ok {
            return []string{}
        }
        node = node.children[ch]
    }

    // DFS для сбора всех слов
    words := []string{}
    dfs(node, prefix, &words)
    return words
}

func dfs(node *TrieNode, current string, words *[]string) {
    if node.isEnd {
        *words = append(*words, current)
    }

    for ch, child := range node.children {
        dfs(child, current+string(ch), words)
    }
}

// trie.FindWordsWithPrefix("car") → ["car", "card", "care"]
```

## Оптимизация: массив вместо map

Для только ASCII строчных букв:

```go
type TrieNode struct {
    children [26]*TrieNode  // a-z
    isEnd    bool
}

func (t *Trie) Insert(word string) {
    node := t.root

    for _, ch := range word {
        index := ch - 'a'  // 'a' → 0, 'b' → 1, ...

        if node.children[index] == nil {
            node.children[index] = &TrieNode{}
        }

        node = node.children[index]
    }

    node.isEnd = true
}

func (t *Trie) Search(word string) bool {
    node := t.root

    for _, ch := range word {
        index := ch - 'a'

        if node.children[index] == nil {
            return false
        }

        node = node.children[index]
    }

    return node.isEnd
}
```

**Преимущества:**
- Быстрее (O(1) доступ)
- Меньше аллокаций

**Недостатки:**
- Больше памяти (26 указателей на узел)
- Только для ограниченного алфавита

## Применение

### 1. Autocomplete (автодополнение)

```go
func (t *Trie) Autocomplete(prefix string) []string {
    return t.FindWordsWithPrefix(prefix)
}

// Пользователь вводит "ca"
// → ["cat", "car", "card", "care"]
```

### 2. Spell Checker

```go
func (t *Trie) SpellCheck(word string) bool {
    return t.Search(word)
}

// Проверить есть ли слово в словаре
```

### 3. Word Search в сетке

```go
// LeetCode 212: Word Search II
func findWords(board [][]byte, words []string) []string {
    trie := NewTrie()
    for _, word := range words {
        trie.Insert(word)
    }

    result := make(map[string]bool)

    for i := range board {
        for j := range board[0] {
            dfsBoard(board, i, j, trie.root, "", result)
        }
    }

    // Преобразовать map в slice
    ans := []string{}
    for word := range result {
        ans = append(ans, word)
    }

    return ans
}

func dfsBoard(board [][]byte, i, j int, node *TrieNode, word string, result map[string]bool) {
    if i < 0 || i >= len(board) || j < 0 || j >= len(board[0]) {
        return
    }

    ch := rune(board[i][j])
    if ch == '#' || node.children[ch] == nil {
        return
    }

    word += string(ch)
    node = node.children[ch]

    if node.isEnd {
        result[word] = true
    }

    // Пометить посещенную клетку
    board[i][j] = '#'

    // DFS в 4 направлениях
    dfsBoard(board, i+1, j, node, word, result)
    dfsBoard(board, i-1, j, node, word, result)
    dfsBoard(board, i, j+1, node, word, result)
    dfsBoard(board, i, j-1, node, word, result)

    // Восстановить клетку
    board[i][j] = byte(ch)
}
```

### 4. Longest Common Prefix

```go
func longestCommonPrefix(strs []string) string {
    if len(strs) == 0 {
        return ""
    }

    trie := NewTrie()
    for _, str := range strs {
        trie.Insert(str)
    }

    prefix := ""
    node := trie.root

    for len(node.children) == 1 && !node.isEnd {
        // Только один ребёнок → общий префикс продолжается
        for ch, child := range node.children {
            prefix += string(ch)
            node = child
        }
    }

    return prefix
}

// longestCommonPrefix(["flower","flow","flight"]) → "fl"
```

### 5. Replace Words (словарь корней)

```go
// LeetCode 648
func replaceWords(dictionary []string, sentence string) string {
    trie := NewTrie()
    for _, root := range dictionary {
        trie.Insert(root)
    }

    words := strings.Split(sentence, " ")
    for i, word := range words {
        words[i] = findRoot(trie, word)
    }

    return strings.Join(words, " ")
}

func findRoot(trie *Trie, word string) string {
    node := trie.root
    prefix := ""

    for _, ch := range word {
        if _, ok := node.children[ch]; !ok {
            return word  // Корень не найден
        }

        prefix += string(ch)
        node = node.children[ch]

        if node.isEnd {
            return prefix  // Нашли корень
        }
    }

    return word
}

// dictionary: ["cat", "bat", "rat"]
// sentence: "the cattle was rattled by the battery"
// → "the cat was rat by the bat"
```

### 6. Add and Search Word (с wildcard)

```go
type WordDictionary struct {
    root *TrieNode
}

func (wd *WordDictionary) AddWord(word string) {
    node := wd.root

    for _, ch := range word {
        if _, ok := node.children[ch]; !ok {
            node.children[ch] = &TrieNode{
                children: make(map[rune]*TrieNode),
            }
        }
        node = node.children[ch]
    }

    node.isEnd = true
}

func (wd *WordDictionary) Search(word string) bool {
    return searchHelper(wd.root, word, 0)
}

func searchHelper(node *TrieNode, word string, index int) bool {
    if node == nil {
        return false
    }

    if index == len(word) {
        return node.isEnd
    }

    ch := rune(word[index])

    if ch == '.' {
        // Wildcard: попробовать все варианты
        for _, child := range node.children {
            if searchHelper(child, word, index+1) {
                return true
            }
        }
        return false
    }

    child, ok := node.children[ch]
    if !ok {
        return false
    }

    return searchHelper(child, word, index+1)
}

// dict.AddWord("bad")
// dict.AddWord("dad")
// dict.Search("pad") → false
// dict.Search("bad") → true
// dict.Search(".ad") → true
// dict.Search("b..") → true
```

## Сложность

| Операция | Сложность |
|----------|-----------|
| Insert | O(m) |
| Search | O(m) |
| StartsWith | O(m) |
| Delete | O(m) |
| Space | O(ALPHABET_SIZE × N × M) |

где:
- m = длина слова
- N = количество слов
- M = средняя длина слова

## Trie vs Hash Table

| Свойство | Trie | Hash Table |
|----------|------|------------|
| Поиск | O(m) | O(1)* |
| Префикс | O(m) | O(n) |
| Autocomplete | ✅ | ❌ |
| Упорядоченность | ✅ | ❌ |
| Память | Больше | Меньше |
| Реализация | Сложнее | Проще |

\* Амортизированное

**Используйте Trie когда:**
- Нужен поиск по префиксу
- Autocomplete
- Spell checker
- IP routing
- T9 (словарь для телефона)

**Используйте Hash Table когда:**
- Только точный поиск
- Префиксы не нужны
- Память ограничена

## Compressed Trie (Patricia Tree)

Объединяет цепочки одиночных узлов:

```
Обычный Trie:          Compressed:

    r                      r
    |                      |
    o                    omane
    |                     / \
    m                   s    (root)
    |
    a
    |
    n
    |
    e
   / \
  s   (root)
```

**Преимущества:**
- Меньше узлов
- Экономия памяти

**Недостатки:**
- Сложнее реализация

## Best Practices

1. ✅ Используйте массив для фиксированного алфавита
2. ✅ Используйте map для Unicode
3. ✅ Compressed Trie для экономии памяти
4. ✅ Trie для autocomplete и prefix search
5. ❌ Не используйте для точного поиска (Hash Table лучше)
6. ❌ Помните о потреблении памяти

## Связанные темы

- [[Деревья - Основы]]
- [[HashMap - Реализация и особенности]]
- [[BFS и DFS]]
- [[Алгоритмы на строках]]
