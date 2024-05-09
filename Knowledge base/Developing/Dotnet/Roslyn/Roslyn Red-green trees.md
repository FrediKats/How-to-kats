---
title: Roslyn Red-Green trees
---

Red-green trees - это концепция синтаксических деревьев в [[Roslyn]], которая предполагает представление синтаксического дерева двумя деревьями:
- Зелёное иммутабельное дерево, которое знает про своё содежимое, длинну, но ничего не знает про parent-child, позицию в дереве
- Красное мутабельнео дерево, которое является фасадом вокруг зелёного

## Sources
- https://learn.microsoft.com/en-us/archive/blogs/ericlippert/persistence-facades-and-roslyns-red-green-trees.