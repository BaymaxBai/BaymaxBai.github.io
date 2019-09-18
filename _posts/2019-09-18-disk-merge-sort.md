---
layout: post
title: "磁盘多路归并排序"
subtitle: 'Golang实现磁盘多路归并排序'
author: "BaymaxBai"
header-style: text
tags:
  - Go
  - Algorithm
---

### 算法说明
基于磁盘对大文件进行多路归并排序，通过对多个有序子文件进行合并来得到一个完全有序的文件

### 实现思路
将一个大文件切割为n个小文件，每个小文件都是有序的，然后在内存中准备n个优先队列，通过对n个优先队列中的数据进行最小值弹出操作来最终得到一个合并后的完全有序的大文件

### 图示说明

![](https://tva1.sinaimg.cn/large/006y8mN6gy1g6wsayzvl3j30u016taf1.jpg)

### 代码实现

```
package main

import (
	"bufio"
	"golang-sort-algorithm/diskmerge"
	"math/rand"
	"os"
	"sort"
	"strconv"
)

func main() {
	// generateFiles
	files := generateFiles()
	// sort file
	diskmerge.DiskMergeSort(files)
	// remove files and move outfile to tmp
	for _, file := range files {
		os.Remove(file)
	}
	os.Rename("output.txt", "/tmp/output.txt")
}

func generateFiles() []string {
	files := []string{}

	for i := 0; i < 10; i++ {
		filename := "input-" + strconv.Itoa(i) + ".txt"
		files = append(files, filename)
		// sort

		numbers := []int{}
		for k := 0; k < 100; k++ {
			numbers = append(numbers, rand.Intn(1000))
		}

		sort.Ints(numbers)

		// write random numbers to input files
		file, _ := os.Create(filename)
		wr := bufio.NewWriter(file)
		for _, number := range numbers {
			wr.WriteString(strconv.Itoa(number) + "\n")
			wr.Flush()
		}
		file.Close()
	}

	return files
}
```


```
package diskmerge
// 多路归并实现
import (
	"bufio"
	"os"
	"sort"
	"strconv"
	"strings"
)

// Source 排序输入源
type Source struct {
	file   *os.File
	reader *bufio.Reader
	line   string
}

// 是否还有下一个元素
func (s *Source) hasNext() bool {
	line, _, err := s.reader.ReadLine()
	s.line = strings.TrimSpace(string(line))
	if err != nil {
		return false
	}
	return true
}

// 获取下一个元素
func (s *Source) next() int {
	if s.line == "" {
		if !s.hasNext() {
			panic(s)
		}
	}
	i, _ := strconv.Atoi(s.line)
	return i
}

// Out 排序输出
type Out struct {
	file   *os.File
	writer *bufio.Writer
}

func (o *Out) write(item MemoryItem) {
	o.writer.WriteString(strconv.Itoa(item.Item) + "\n")
}

func (o *Out) close() {
	if o.writer != nil && o.file != nil {
		o.writer.Flush()
		o.file.Close()
	}
}

// MemoryItem 内存元素
type MemoryItem struct {
	Item   int
	Source Source
}

// MemoryItems 内存排序对象
type MemoryItems []*MemoryItem

func (m MemoryItems) Len() int      { return len(m) }
func (m MemoryItems) Swap(i, j int) { m[i], m[j] = m[j], m[i] }

type ByItem struct {
	MemoryItems
}

func (b ByItem) Less(i, j int) bool { return b.MemoryItems[i].Item < b.MemoryItems[j].Item }

// DiskMergeSort 硬盘多路归并排序
func DiskMergeSort(files []string) {

	items := []*MemoryItem{}

	for _, filename := range files {
		file, _ := os.Open(filename)
		reader := bufio.NewReader(file)
		source := Source{file: file, reader: reader}
		item := MemoryItem{Source: source, Item: source.next()}
		items = append(items, &item)
	}

	// sort items
	sort.Sort(ByItem{items})

	filename := "output.txt"
	file, _ := os.Create(filename)
	wr := bufio.NewWriter(file)
	out := Out{file, wr}
	defer file.Close()
	defer out.close()

	for {
		current := items[0]
		out.write(*current)
		if current.Source.hasNext() {
			current.Item = current.Source.next()
		} else {
			// remove first
			items = items[1:]
			if len(items) == 0 {
				break
			}
		}
		sort.Sort(ByItem{items})
	}
}
```
