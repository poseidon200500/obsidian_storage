---
tags:
  - Шаблоны
  - Программирование
---
	# Основы BST

*Фундаментальные свойства и операции бинарных деревьев поиска*

## Определение и свойства BST

**Бинарное дерево поиска (BST)** — это бинарное дерево, обладающее следующими свойствами:

1. **Упорядоченность**: Для каждого узла X:
   - Все элементы в левом поддереве ≤ X
   - Все элементы в правом поддереве > X

2. **Рекурсивная структура**: Каждое поддерево также является BST

3. **Уникальность ключей** (в базовой версии): Все ключи уникальны

### Основные свойства:
- **Минимум**: Самый левый узел
- **Максимум**: Самый правый узел  
- **Высота**: O(log n) в сбалансированном дереве, O(n) в вырожденном
- **Обход in-order**: Возвращает элементы в отсортированном порядке

## Базовая структура узла

```go
// Базовая структура узла BST
type TreeNode struct {
    Val   int
    Left  *TreeNode
    Right *TreeNode
}

// Альтернативная структура с дополнительными полями
type BSTNode struct {
    Key    int
    Value  interface{} // Для ассоциативного массива
    Left   *BSTNode
    Right  *BSTNode
    Parent *BSTNode    // Опционально, для некоторых операций
    Height int         // Для балансировки
}

// Конструктор узла
func NewTreeNode(val int) *TreeNode {
    return &TreeNode{
        Val:   val,
        Left:  nil,
        Right: nil,
    }
}
```

### Пример построения BST:
```go
// Ручное построение BST
func createSampleBST() *TreeNode {
    root := &TreeNode{Val: 8}
    
    root.Left = &TreeNode{Val: 3}
    root.Right = &TreeNode{Val: 10}
    
    root.Left.Left = &TreeNode{Val: 1}
    root.Left.Right = &TreeNode{Val: 6}
    
    root.Left.Right.Left = &TreeNode{Val: 4}
    root.Left.Right.Right = &TreeNode{Val: 7}
    
    root.Right.Right = &TreeNode{Val: 14}
    root.Right.Right.Left = &TreeNode{Val: 13}
    
    return root
}

//        8
//      /   \
//     3     10
//    / \      \
//   1   6      14
//      / \    /
//     4   7  13
```

## Проверка валидности BST

### Рекурсивная проверка с границами
```go
func isValidBST(root *TreeNode) bool {
    return validate(root, nil, nil)
}

func validate(node *TreeNode, min *int, max *int) bool {
    if node == nil {
        return true
    }
    
    // Проверка текущего узла
    if min != nil && node.Val <= *min {
        return false
    }
    if max != nil && node.Val >= *max {
        return false
    }
    
    // Рекурсивная проверка поддеревьев
    return validate(node.Left, min, &node.Val) && 
           validate(node.Right, &node.Val, max)
}

// Альтернативная версия с использованием диапазонов
func isValidBST2(root *TreeNode) bool {
    var validateRange func(node *TreeNode, min, max int) bool
    validateRange = func(node *TreeNode, min, max int) bool {
        if node == nil {
            return true
        }
        
        if node.Val <= min || node.Val >= max {
            return false
        }
        
        return validateRange(node.Left, min, node.Val) && 
               validateRange(node.Right, node.Val, max)
    }
    
    return validateRange(root, -1<<31, 1<<31-1) // Используем MinInt32 и MaxInt32
}
```

### In-order проверка
```go
func isValidBSTInOrder(root *TreeNode) bool {
    var prev *int
    var inOrder func(node *TreeNode) bool
    
    inOrder = func(node *TreeNode) bool {
        if node == nil {
            return true
        }
        
        if !inOrder(node.Left) {
            return false
        }
        
        if prev != nil && node.Val <= *prev {
            return false
        }
        prev = &node.Val
        
        return inOrder(node.Right)
    }
    
    return inOrder(root)
}
```

## Высота и балансировка

### Вычисление высоты дерева
```go
func height(root *TreeNode) int {
    if root == nil {
        return -1 // Или 0, в зависимости от определения
    }
    
    leftHeight := height(root.Left)
    rightHeight := height(root.Right)
    
    return 1 + max(leftHeight, rightHeight)
}

func max(a, b int) int {
    if a > b {
        return a
    }
    return b
}
```

### Проверка сбалансированности
```go
func isBalanced(root *TreeNode) bool {
    var checkBalance func(node *TreeNode) (bool, int)
    
    checkBalance = func(node *TreeNode) (bool, int) {
        if node == nil {
            return true, -1
        }
        
        leftBalanced, leftHeight := checkBalance(node.Left)
        if !leftBalanced {
            return false, 0
        }
        
        rightBalanced, rightHeight := checkBalance(node.Right)
        if !rightBalanced {
            return false, 0
        }
        
        // Дерево сбалансировано, если разница высот <= 1
        balanced := abs(leftHeight - rightHeight) <= 1
        height := 1 + max(leftHeight, rightHeight)
        
        return balanced, height
    }
    
    balanced, _ := checkBalance(root)
    return balanced
}

func abs(x int) int {
    if x < 0 {
        return -x
    }
    return x
}
```

### Фактор балансировки
```go
func balanceFactor(node *TreeNode) int {
    if node == nil {
        return 0
    }
    return height(node.Left) - height(node.Right)
}

// Проверка необходимости балансировки
func needsBalancing(node *TreeNode) bool {
    factor := balanceFactor(node)
    return factor < -1 || factor > 1
}
```

## Основные инварианты BST

### 1. Инвариант упорядоченности
```go
// Проверка, что все элементы в поддереве находятся в диапазоне [min, max]
func checkOrderInvariant(node *TreeNode, min, max int) bool {
    if node == nil {
        return true
    }
    
    if node.Val < min || node.Val > max {
        return false
    }
    
    return checkOrderInvariant(node.Left, min, node.Val-1) && 
           checkOrderInvariant(node.Right, node.Val+1, max)
}
```

### 2. Инвариант структуры
```go
// Проверка, что дерево является корректным бинарным деревом
func checkStructureInvariant(root *TreeNode) bool {
    visited := make(map[*TreeNode]bool)
    return checkStructureHelper(root, visited)
}

func checkStructureHelper(node *TreeNode, visited map[*TreeNode]bool) bool {
    if node == nil {
        return true
    }
    
    // Проверка на циклы
    if visited[node] {
        return false
    }
    visited[node] = true
    
    // Рекурсивная проверка детей
    return checkStructureHelper(node.Left, visited) && 
           checkStructureHelper(node.Right, visited)
}
```

### 3. Инвариант уникальности
```go
func checkUniquenessInvariant(root *TreeNode) bool {
    elements := make(map[int]bool)
    return checkUniquenessHelper(root, elements)
}

func checkUniquenessHelper(node *TreeNode, elements map[int]bool) bool {
    if node == nil {
        return true
    }
    
    if elements[node.Val] {
        return false
    }
    elements[node.Val] = true
    
    return checkUniquenessHelper(node.Left, elements) && 
           checkUniquenessHelper(node.Right, elements)
}
```

## Вспомогательные функции

### Поиск минимума и максимума
```go
func findMin(root *TreeNode) *TreeNode {
    if root == nil {
        return nil
    }
    current := root
    for current.Left != nil {
        current = current.Left
    }
    return current
}

func findMax(root *TreeNode) *TreeNode {
    if root == nil {
        return nil
    }
    current := root
    for current.Right != nil {
        current = current.Right
    }
    return current
}
```

### Подсчет количества узлов
```go
func countNodes(root *TreeNode) int {
    if root == nil {
        return 0
    }
    return 1 + countNodes(root.Left) + countNodes(root.Right)
}
```

### Проверка существования значения
```go
func contains(root *TreeNode, target int) bool {
    if root == nil {
        return false
    }
    
    if root.Val == target {
        return true
    } else if target < root.Val {
        return contains(root.Left, target)
    } else {
        return contains(root.Right, target)
    }
}
```

## Пример полной проверки BST
```go
func validateFullBST(root *TreeNode) bool {
    return isValidBST(root) && 
           isBalanced(root) && 
           checkStructureInvariant(root) && 
           checkUniquenessInvariant(root)
}

// Комплексная диагностика
func diagnoseBST(root *TreeNode) {
    fmt.Printf("Is valid BST: %t\n", isValidBST(root))
    fmt.Printf("Is balanced: %t\n", isBalanced(root))
    fmt.Printf("Height: %d\n", height(root))
    fmt.Printf("Node count: %d\n", countNodes(root))
    
    minNode := findMin(root)
    maxNode := findMax(root)
    if minNode != nil {
        fmt.Printf("Min value: %d\n", minNode.Val)
    }
    if maxNode != nil {
        fmt.Printf("Max value: %d\n", maxNode.Val)
    }
}
```

## Заключение

Основные принципы BST обеспечивают:

- **Эффективный поиск**: O(h) время, где h - высота дерева
- **Динамическую структуру**: Легкое добавление и удаление
- **Автоматическую сортировку**: In-order обход дает отсортированную последовательность

Понимание этих фундаментальных концепций необходимо для работы с более сложными вариантами BST (AVL, Красно-черные деревья) и для решения алгоритмических задач.