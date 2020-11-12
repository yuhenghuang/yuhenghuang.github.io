---
title: Post-Order Tree Traversal by Iteration
top: false
cover: false
toc: true
mathjax: false
date: 2020-11-12 20:58:05
password:
summary:
tags: 
  - Tree
categories:
  - Algorithm
---


The implementations of three types of tree traversal are all trivial by recursion, however, the problem becomes much more difficult if the traversal is done by an iterative way. Among three traversals, post-order traversal is even harder to implemented as it traverses from left, right to the root itself at last.

This post introduces a practical example by solving a problem on [Leetcode](https://leetcode.com/) using iterative post-order traversal.

[Binary Tree Tilt](https://leetcode.com/problems/binary-tree-tilt/)

The algorithm of the solution is quite straight forward.

1. Sum up the values from deeper to left node and right node respectively

2. Compute the absolute difference of sums between left and right node, to see whether update answer or not

3. Sum up sums of left, right and the node itself to ancestors.


### Recursion

The return value of `dfs(TreeNode*)` is the sum of all the deeper node values.

```cpp
class Solution {
  private:
    int res;
    int dfs(TreeNode* root) {
      if (!root) return 0;

      int l = dfs(root->left);
      int r = dfs(root->right);
      res += abs(l - r);

      return l + r + root->val;
    }
  public:
    int findTilt(TreeNode* root) {
      res = 0;
      dfs(root);
      return res;
    }
};
```



### Iteration

```cpp
#include <stack>
#include <unordered_map>

using namespace std;

class Solution {
  public:
    int findTiltIter(TreeNode* root) {
      stack<TreeNode*> s;
      unordered_map<TreeNode*, int> m;
      TreeNode* super;

      int res = 0;
      while (!s.empty() || root) {

        // push root and its left non-null child
        while (root) {
          s.push(root);
          root = root->left;
        }

        while (!s.empty() && !s.top()->right) {

          // if there is node in stack, and its right child is null
          // this means it either has no right child, or right child is detached (see next block)
          // save the pointer to root, pop it
          root = s.top();
          s.pop();

          if (!s.empty()) {

            // if stack is not empty
            // the one at the stack top is the parent of root
            // update the summed value
            super = s.top();
            super->val += root->val;

            if (super->right)
              // if super has right child, and not detached yet
              // at this point root must be super's left child
              // save the value to the hash map
              m[super] = root->val;
            else
              // if super has no right child (m[super] = 0), or root is actually its detached right child
              res += abs(m[super] - root->val);
          }
        }

        if (!s.empty()) {
          // assign the right child of the node at stack top to root
          root = s.top()->right;
          // detach the right child so that previous block can work smoothly
          s.top()->right = nullptr;
        }
        else
          // empty stack indicates that root is the original root of the whole tree
          // and the tree traversal is completed
          // assign null pointer to root to end the while loop
          root = nullptr;
      }

      return res;
    }
};
```


To achieve the same thing by iteration, we must resort to `stack`. And as there is no way we can save summed value from deeper nodes in iteration, I choose to update the values of the tree node. 

Given the feature of the problem, there should be another data structure that saves the summed values of children of a node if neither of two children is null. This is because in iteration we cannot access two children of a node at the same time.

For other details, please follow the inline comments.
