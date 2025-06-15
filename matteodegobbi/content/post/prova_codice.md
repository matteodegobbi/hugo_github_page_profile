+++
title = "Prova_codice"
date = "2025-06-13T23:34:39+02:00"
#dateFormat = "2006-01-02" # This value can be configured for per-post date formatting
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["", ""]
keywords = ["", ""]
description = ""
showFullContent = false
readingTime = false
hideComments = false
categories = ["Code", "Deep Learning"]
+++
```cpp

#pragma once

#include <algorithm>
#include <iostream>
#include <stdexcept>
#include <utility>
#include <vector>

template <typename Key, typename Val, int M> class BTree {
  // M is the minimum degree of the B-tree.
  // Max keys in a node: 2*M - 1
  // Min keys in a non-root node: M - 1
private:
  struct Node {
    bool is_leaf;
    std::vector<Key> keys;
    std::vector<Val> vals;
    std::vector<Node *> children;

    Node(bool leaf = false) : is_leaf(leaf) {}

    // NOTE: destructor called recursively
    ~Node() {
      for (Node *child : children) {
        delete child;
      }
    }
  };

  Node *root = nullptr;
  int _size = 0;
  int _height = 0;
  int split_counter = 0;

  void _clear(Node *node) {
    if (node != nullptr) {
      delete node; // The destructor of sruct Node deletes children recursively
    }
  }

  // NOTE: Functions for search in this section

  Val *_search(Node *node, const Key &k) {
    if (node == nullptr) {
      return nullptr;
    }

    // STL binary search to get first element >= to k
    auto it = std::lower_bound(node->keys.begin(), node->keys.end(), k);

    // get index from iterator
    int i = std::distance(node->keys.begin(), it);

    // found k
    if (it != node->keys.end() && *it == k) {
      return &node->vals[i];
    }

    if (node->is_leaf) {
      return nullptr;
    }

    return _search(node->children[i], k);
  }

  // NOTE: Functions for insertion in this section
  //
  // splits the child at `parent->children[child_index]`
  void _split_child(Node *parent, int child_index) {
    split_counter++;
    Node *child_to_split = parent->children[child_index];

    // create new sibling for splitting the node in half
    Node *new_child = new Node(child_to_split->is_leaf);

    // Promote the median key and value to the parent node
    parent->keys.insert(parent->keys.begin() + child_index,
                        child_to_split->keys[M - 1]);
    parent->vals.insert(parent->vals.begin() + child_index,
                        child_to_split->vals[M - 1]);

    // Put the sibling as a child of the parent
    // It will be the sibling  to the right of the split node
    parent->children.insert(parent->children.begin() + child_index + 1,
                            new_child);

    // move half of keys and values to the split node's sibling
    new_child->keys.assign(child_to_split->keys.begin() + M,
                           child_to_split->keys.end());
    new_child->vals.assign(child_to_split->vals.begin() + M,
                           child_to_split->vals.end());

    child_to_split->keys.resize(M - 1);
    child_to_split->vals.resize(M - 1);

    // if the split node was not a leaf, move its children into its new sibling
    if (!child_to_split->is_leaf) {
      new_child->children.assign(child_to_split->children.begin() + M,
                                 child_to_split->children.end());
      child_to_split->children.resize(M);
    }
  }

  // NOTE: IMPORTANT INVARIANT node should not be already full
  // otherwise it will overflow, only call this from public method
  void _insert_rec(Node *node, const Key &k, const Val &v) {

    // STL binary search to get index of first key >= k
    auto it = std::lower_bound(node->keys.begin(), node->keys.end(), k);
    int i = std::distance(node->keys.begin(),
                          it); // get index in vector of iterator it

    if (node->is_leaf) {
      // case where we found already existing key, update the val
      if (it != node->keys.end() && *it == k) {
        node->vals[i] = v;
        return;
      }

      // regular case just insert the key in the leaf
      node->keys.insert(it, k);
      node->vals.insert(node->vals.begin() + i, v);
      _size++;
    } else {
      // case where we found already existing key, update the val, same as in
      // leaf
      if (it != node->keys.end() && *it == k) {
        node->vals[i] = v;
        return;
      }

      // NOTE: important, guarantees that no node will overflow
      // splits the child we will visit recursively in case
      // it's already full
      if (node->children[i]->keys.size() == 2 * M - 1) {
        _split_child(node, i);
        // increase variabile i in case i need to go to the new sibling after
        // the split
        if (k > node->keys[i]) {
          i++;
        }
      }
      _insert_rec(node->children[i], k, v);
    }
  }

  // NOTE: Functions for deletion in this section


  std::pair<Key, Val> _get_pred(Node *node, int i) {
    Node *cur = node->children[i];
    while (!cur->is_leaf) {
      cur = cur->children.back();
    }
    return {cur->keys.back(), cur->vals.back()};
  }

  std::pair<Key, Val> _get_succ(Node *node, int i) {
    Node *cur = node->children[i + 1];
    while (!cur->is_leaf) {
      cur = cur->children.front();
    }
    return {cur->keys.front(), cur->vals.front()};
  }

  void _borrow_from_prev(Node *node, int i) {
    Node *child = node->children[i];
    Node *sibling = node->children[i - 1];

    // take key from parent
    child->keys.insert(child->keys.begin(), node->keys[i - 1]);
    child->vals.insert(child->vals.begin(), node->vals[i - 1]);

    // eventually add children inherited from key taken from parent
    if (!child->is_leaf) {
      child->children.insert(child->children.begin(), sibling->children.back());
      sibling->children.pop_back();
    }

    node->keys[i - 1] = sibling->keys.back();
    node->vals[i - 1] = sibling->vals.back();

    sibling->keys.pop_back();
    sibling->vals.pop_back();
  }

  void _borrow_from_next(Node *node, int i) {
    Node *child = node->children[i];
    Node *sibling = node->children[i + 1];

    child->keys.push_back(node->keys[i]);
    child->vals.push_back(node->vals[i]);

    if (!child->is_leaf) {
      child->children.push_back(sibling->children.front());
      sibling->children.erase(sibling->children.begin());
    }

    node->keys[i] = sibling->keys.front();
    node->vals[i] = sibling->vals.front();

    sibling->keys.erase(sibling->keys.begin());
    sibling->vals.erase(sibling->vals.begin());
  }

  void _merge(Node *node, int i) {
    Node *child = node->children[i];
    Node *sibling = node->children[i + 1];

    child->keys.push_back(node->keys[i]);
    child->vals.push_back(node->vals[i]);

    child->keys.insert(child->keys.end(), sibling->keys.begin(),
                       sibling->keys.end());
    child->vals.insert(child->vals.end(), sibling->vals.begin(),
                       sibling->vals.end());

    if (!child->is_leaf) {
      child->children.insert(child->children.end(), sibling->children.begin(),
                             sibling->children.end());
      sibling->children.clear(); // Important: prevent double deletion
    }

    node->keys.erase(node->keys.begin() + i);
    node->vals.erase(node->vals.begin() + i);
    node->children.erase(node->children.begin() + i + 1);

    delete sibling;
  }

  void _fill(Node *node, int i) {
    // if i can borrow from left sibling
    if (i != 0 && node->children[i - 1]->keys.size() >= M) {
      _borrow_from_prev(node, i);

    // if i can borrow from right sibling
    } else if (i != node->keys.size() &&
               node->children[i + 1]->keys.size() >= M) {
      _borrow_from_next(node, i);

    // if i cannot borrow i need to merge the siblings into a node
    } else {
      if (i != node->keys.size()) {
        _merge(node, i);
      } else {
        _merge(node, i - 1);
      }
    }
  }

  void _remove_in_leaf(Node *node, int i) {
    node->keys.erase(node->keys.begin() + i);
    node->vals.erase(node->vals.begin() + i);
    _size--;
  }

  void _remove_in_internal(Node *node, int i) {
    Key k = node->keys[i];

    if (node->children[i]->keys.size() >= M) {
      auto [pred_key, pred_val] = _get_pred(node, i);
      node->keys[i] = pred_key;
      node->vals[i] = pred_val;
      _remove(node->children[i], pred_key);
    } else if (node->children[i + 1]->keys.size() >= M) {
      auto [succ_key, succ_val] = _get_succ(node, i);
      node->keys[i] = succ_key;
      node->vals[i] = succ_val;
      _remove(node->children[i + 1], succ_key);
    } else {
      _merge(node, i);
      _remove(node->children[i], k);
    }
  }

  void _remove(Node *node, const Key &k) {
    // STL binary search to get index of first key >= k
    auto it = std::lower_bound(node->keys.begin(), node->keys.end(), k);
    int i = std::distance(node->keys.begin(), it);

    if (i < node->keys.size() && node->keys[i] == k) {
      if (node->is_leaf) {
        _remove_in_leaf(node, i);
      } else {
        _remove_in_internal(node, i);
      }
    } else {
      if (node->is_leaf) {
        return; // key not found nothing is removed
      }

      bool is_last_child = (i == node->keys.size());

      // to keep invariant about minimum number of keys
      // in a node is >= M-1
      if (node->children[i]->keys.size() < M) {
        _fill(node, i);
      }

      if (is_last_child && i > node->keys.size()) {
        _remove(node->children[i - 1], k);
      } else {
        _remove(node->children[i], k);
      }
    }
  }

  // NOTE: other functions not related to search, insertio and removal in this
  // section

  void _inorder(Node *node, std::vector<std::pair<Key, Val>> &v) const {
    if (node == nullptr)
      return;

    int i;
    for (i = 0; i < node->keys.size(); i++) {
      if (!node->is_leaf) {
        _inorder(node->children[i], v);
      }
      v.push_back(std::make_pair(node->keys[i], node->vals[i]));
    }

    if (!node->is_leaf) {
      _inorder(node->children[i], v);
    }
  }

  // NOTE: only used for testing the actual height
  // computation dont use this, it's super naive
  int _height_naive(Node *node) const {
    if (node == nullptr) {
      return -1;
    }

    if (node->is_leaf) {
      return 0;
    }

    int max_child_height = -1;
    for (Node *child : node->children) {
      int child_height = _height_naive(child);
      max_child_height = std::max(max_child_height, child_height);
    }

    return 1 + max_child_height;
  }

public:
  BTree() { root = new Node(true); }

  ~BTree() { _clear(root); }

  int size() const { return _size; }

  bool empty() const { return _size == 0; }

  void insert(const Key &k, const Val &v) {
    split_counter = 0;
    Node *r = root;
    // if root is full make a new one with old root as child
    if (r->keys.size() == 2 * M - 1) {
      Node *s = new Node();
      root = s;
      s->children.push_back(r);
      _split_child(s, 0);
      ++_height;
      _insert_rec(s, k, v);
    } else {
      _insert_rec(r, k, v);
    }
  }

  void remove(const Key &k) {
    if (!root) {
      return;
    }

    _remove(root, k);

    // NOTE: case where we need to shrink tree height by 1
    if (root->keys.empty() && !root->is_leaf) {
      Node *old_root = root;
      root = root->children[0];
      old_root->children
          .clear(); // Prevent destructor from deleting the new root
      delete old_root;
      --_height;
    }
  }

  Val &search(const Key &k) {
    Val *result = _search(root, k);
    if (result == nullptr) {
      throw std::runtime_error("Key not found");
    }
    return *result;
  }

  const Val &search(const Key &k) const {
    Val *result = _search(root, k);
    if (result == nullptr) {
      throw std::runtime_error("Key not found");
    }
    return *result;
  }

  bool contains(const Key &k) { return _search(root, k) != nullptr; }

  void inorder_vector(std::vector<std::pair<Key, Val>> &v) const {
    v.clear();
    _inorder(root, v);
  }

  int height() const { return _height; }
  int last_insert_splits() const { return split_counter; }

  void clear() {
    _clear(root);
    root = new Node(true);
    _size = 0;
  }

  // NOTE: only used for testing the actual height
  // computation dont use this, it's super naive
  int _height_naive() { return _height_naive(root); }
};
```
