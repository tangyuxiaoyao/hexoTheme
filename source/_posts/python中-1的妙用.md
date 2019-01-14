---
title: python中-1的妙用
date: 2018-11-12 10:19:55
tags: [python,strip]
---

## 使用背景

>在浏览HadoopStreaming官网时，有看到如下的用法也可以实现截取一行的数据。

<!--more-->

``` python
#!/usr/bin/python

import sys;

def generateLongCountToken(id):
    return "LongValueSum:" + id + "\t" + "1"

def main(argv):
    line = sys.stdin.readline();
    try:
        while line:
            line = line[:-1];
            fields = line.split("\t");
            print generateLongCountToken(fields[0]);
            line = sys.stdin.readline();
    except "end of file":
        return None
if __name__ == "__main__":
     main(sys.argv)
```
## 调研过程

>很好奇这样的用法作用与字符串会有什么效果，就有了如下的调研。

``` python
import sys

def main():
	for line in sys.stdin:
		print line
		print line[:-1]
		print line.strip()

	a = '   '
	print len(a)
	print len(a[:-1])
	print len(a.strip())

if __name__ == '__main__':
	main()
```

+ 输出如下：

``` shell
1,2,3,4

1,2,3,4
1,2,3,4
3
2
0

```
## 总结

>从上可以得出如下的结论：

 1. -1可以去除数据行后面的换行符或者最后的空格或者tab等单个字符
 2. 但是不可以去除多个空格或者多个无用字符
 3. 此类的场景还是推荐使用strip()来保证万无一失,当然在获得完整的一行的数据的场景,该用法完全够用。