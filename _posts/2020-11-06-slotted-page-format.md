---
layout: post
title: Slotted page format
---

## Introduction

`Slotted page structure` is a very widely used data format in many database systems, especially MySQL and SQLite. One of its biggest strengths is its ability to store variable-length efficiently without wasting too much space.

![Alt Visualization](https://static.javatpoint.com/dbms/images/file-organization-storage.png)

This post will be a discussion about the `Slotted page structure`. There will be a simple implementation of `Slotted page structure` at the end of this post

## Data format

A `slotted page` is usually divided into two components: `Header` and `Records`. Between these two is a `Free space` area, which allows the `Header` to grow rightward, and the `Records` to grow leftward. Because of this characteristic, it is very easy for the engineers to fill in pages of this type a variable number of records without wasting lots of space.

### Header

The header of a `slotted page` should hold the below information:
- The number of record entries in the header
- Amount of remaining free space in the page
- Information on the location and size of the records

### Records

Records will be placed based on the inserting order of the records, which means that whatever record inserted earlier will be placed on the right of the page.

One problem with this mechanism is that, whenever a record is deleted, space previously occupied by the record will be empty, which leads to internal fragmentation. Periodically garbage collecting those fragments may help, but it requires extra computing overhead. However, this is acceptable, comparing to the benefits `slotted page` bring.

## Simple implementation

Full implementation can be found here: https://github.com/duynguyen-ori75/playground/tree/master/blockds

### Class definition

`std::vector` is used mostly because its provided standard library is exceptional. Its behavior is also very similar to that of the traditional array - except for the ability to auto-resize. However, in this implementation, the auto-resize ability won't be used.

```cpp
class SlottedPage {
  std::vector<int> data_;
  int maxSize_, currentSize_, payloadOffset_;
}
```

### Insert operator

```cpp
bool Insert(int key, int value) {
  // look up for possible index of the (key, value) pair - aka `record` - in the `slotted page`
  auto index = this->lookUp(key);
  // if a similar key already exists, update its value
  if (index < this->currentSize_ &&
      cellPayload(this->data_[index]).first == key) {
    this->data_[this->data_[index] + 1] = value;
    return true;
  }
  // if the page is full, return
  if (this->currentSize_ >= this->maxSize_) return false;
  // inserting the (key, value) pair
  currentSize_++;
  // determine the correct offset location in the `Records` space, and insert the record
  this->payloadOffset_ -= 2;
  this->data_[this->payloadOffset_] = key;
  this->data_[this->payloadOffset_ + 1] = value;
  // shift-right all offsets of records whose key is higher than the key of new record
  for (int idx = currentSize_; idx > index; idx--)
    this->data_[idx] = this->data_[idx - 1];
  // specify the offset of the new record
  this->data_[index] = this->payloadOffset_;
  return true;
}
```

### Remove operator

```cpp
bool Remove(int key) {
  // look up for the index of the key
  auto index = this->lookUp(key);
  // if the key not exists, return false
  if (index >= this->currentSize_ ||
      cellPayload(this->data_[index]).first != key)
    return false;
  // deleting the key by shift-left all bigger records
  int deletedOffset = this->data_[index];
  this->currentSize_--;
  for (int idx = index; idx < this->currentSize_; idx++)
    this->data_[idx] = this->data_[idx + 1];
  this->data_[currentSize_] = 0;
  // empty payload cell
  this->data_[deletedOffset] = this->data_[deletedOffset + 1] = 0;
  return true;
}
```