---
title: 关于book重命名的一些事儿
date: 2025-01-29 19:25:03
banner_img: /img/bg/archive.jpg
tags: 代码
excerpt: 书籍需要归档，如何重新定义书籍名字
categories: [实用代码, go语言]
---

# 需求
我从网上下了一批万本的英文电子书，需要按照特定的方式进行归档。约定规则如下:

* 源文件命名方式：Title - author.mobi
* Book需要按照作者A-Z进行分组
* 同一个作者，超过2本生成一个以作者名作为文件夹的名字
* 单本的mobi文件命名方式：【author】Title.mobi

例如源文件为： Lightnin' Hopkins_ His Life and Blues - Alan Govenar.mobi  
新生成的文件为：【Alan Govenar】Lightnin' Hopkins_ His Life and Blues

# 核心代码
定义来源文件夹、目标文件夹路径：
```go
	sourceDir := ""
	targetDir := ""
```
遍历来源目录下，统计每个作者的文件数量：
```go
	// 用于存储每个作者的文件计数
	authorCount := make(map[string]int)

	// 首先遍历目录以统计每个作者的文件数量
	err := filepath.Walk(sourceDir, func(path string, info os.FileInfo, err error) error {
		if err != nil {
			return err
		}
		// 检查文件是否是.mobi文件
		if !info.IsDir() && filepath.Ext(path) == ".mobi" {
			fileName := info.Name()
			parts := strings.SplitN(fileName, " - ", 2)
			if len(parts) == 2 {
				author := strings.TrimSpace(strings.TrimSuffix(parts[1], ".mobi"))
				authorCount[author]++
			}
		}
		return nil
	})
```
遍历目录，并复制文件：
```go
	// 遍历目录并复制文件
	err = filepath.Walk(sourceDir, func(path string, info os.FileInfo, err error) error {
		if err != nil {
			return err
		}
		// 检查文件是否是.mobi文件
		if !info.IsDir() && filepath.Ext(path) == ".mobi" {
			fileName := info.Name()
			parts := strings.SplitN(fileName, " - ", 2)
			if len(parts) == 2 {
				author := strings.TrimSpace(strings.TrimSuffix(parts[1], ".mobi"))
				newFileName := fmt.Sprintf("【%s】%s.mobi", author, strings.TrimSpace(parts[0]))

				// 仅在作者的文件数量大于1时创建文件夹
				if authorCount[author] > 1 {
					authorDir := filepath.Join(targetDir, author)
					err := os.MkdirAll(authorDir, os.ModePerm)
					if err != nil {
						return err
					}

					// 复制文件到作者文件夹并重命名，去掉【到】之间的内容
					newFilePath := filepath.Join(authorDir, removeBrackets(newFileName))
					err = copyFile(path, newFilePath)
					if err != nil {
						return err
					}
					fmt.Printf("已复制并重命名: %s -> %s\n", fileName, newFilePath)
				} else {
					// 如果文件数量小于或等于1，直接复制到目标目录，保持原样
					newFilePath := filepath.Join(targetDir, newFileName)
					err = copyFile(path, newFilePath)
					if err != nil {
						return err
					}
					fmt.Printf("已复制: %s -> %s\n", fileName, newFilePath)
				}
			}
		}
		return nil
	})
```
文件名替换规则逻辑：
```go
// 去掉文件名中[到]之间的内容的辅助函数
func removeBrackets(fileName string) string {
	// 替换【为[，】为]
	modifiedFileName := strings.ReplaceAll(fileName, "【", "[")
	modifiedFileName = strings.ReplaceAll(modifiedFileName, "】", "]")

	start := strings.Index(modifiedFileName, "[")
	end := strings.Index(modifiedFileName, "]")
	if start != -1 && end != -1 && end > start {
		// 返回去掉[到]之间内容后的字符串，并去掉前后的空格
		str := strings.TrimSpace(modifiedFileName[end+1:]) // 从]后开始
		fmt.Printf("遍历文件名: %s => %s\n", fileName, str)     // 使用 %s 格式化输出
		return str
	}

	return modifiedFileName // 如果没有找到，则返回修改后的文件名
}
```
# 执行
1. 填入sourceDir和targetDir的值
2. 执行main函数:  go run ./bookrename.go
3. 显示执行结果： 
```
已复制并重命名: Doctor Who_ Remembrance of the Daleks - Ben Aaronovitch.mobi -> {{mydir}}/dist/Ben Aaronovitch/Doctor Who_ Remembrance of the Daleks.mobi
```