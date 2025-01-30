---
title: 'LeetCode Daily Question: #1382'
date: 2024-06-27T07:56:53+08:00
draft: false
type: leetcode
tags: ['bst']
---

## Problem

Given the root of a BST, return a balanced BST with the same node values. If there is more than one answer, return any of them.

## Key Takeaways

* An inorder traversal of a BST returns a list of sorted values. 
* Whenever recursing using indices, make sure to check if the start index ever exceeds the end index as a base case. 
* Calculate the midpoint between two indices as `(start + (end - start) / 2)`.

## Algorithm

1. Do inorder traversal on the original BST and get a vector of sorted values in the BST.
2. Recurse on the vector to construct the balanced BST, keeping track of the start and index node at each step.
    * For each step in the recursion, set the midpoint as the root of that subtree, and then construct the left and right subtrees out of the two halves of the vector.

## Implementation Details

### Inorder Traversal of BST

```cpp
void sortedElements(TreeNode* root, vector<int>& sol) {
    if (!root) {
        return {};
    }
    sortedElements(root->left, sol);
    sol.push_back(root->val);
    sortedElements(root->right, sol);
}
```

### Constructing Balanced BST

```cpp
TreeNode* constructBBST(TreeNode* root, vector<int> vals, int start, int end) {
    if (start > end) {
        return nullptr;
    }
    if (start == end) {
        return new TreeNode(vals[start]);
    }
    int mid = start + (end - start) / 2;
    root = new TreeNode(vals[mid]);
    root->left = constructBBST(root->left, vals, start, mid - 1);
    root->right = constructBBST(root->right, vals, mid + 1, end);
    return root;
}
```

### Final Solution

```cpp
TreeNode* solution(TreeNode* root) {
    vector<int> vals;
    sortedElements(root, vals);
    TreeNode * fin = constructBBST(nullptr, vals, 0, vals.size() - 1);
}
```
