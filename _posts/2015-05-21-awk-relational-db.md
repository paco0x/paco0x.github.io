---
layout:     post
title:      用 awk 实现一个关系型数据库
subtitle:   "AWK & relational DB"
date:       2015-05-21 23:00:00
author:     Liao
catalog:    true
header-img: img/post-bg-2015.jpg
permalink:  /awk-relational-db/
tags:
    - Basic
---

awk 的祖师爷 Brian W. Kernighan，写过一本 《[The AWK Programming Language](http://book.douban.com/subject/1876898/)》，这本书一如 Brian W. Kernighan 的其他书，简明扼要却不乏深入。更厉害的是这本书在淘宝上的售价居然高达 1000 多。

书中用 awk + 纯文本数据模拟了一个微型的关系型数据库外加一个数据库查询语言编译器，看完真让我感觉脑洞大开。

本文将整个过程整理下来，感兴趣的可以去找原书，应该有 PDF 版本的可以下载到。

### 小型的关系型数据库系统

提到关系型数据库就少不了查询语言。可以用 awk 实现一个类 awk 的查询语言，称之为 `q`，使用一个称之为 `qawk` 的解释器程序将 `q` 查询语句转换为 awk 程序。

`q` 语言使用字段名来引用关系型数据中表的字段名，例如使用 $area 查询 area 字段的值。数据库的文件应该可以支持存放在多个文件中（例如每个表一个文件），那么我们还需要记录数据文件的原信息，我们将这个数据字典文件称为 "relfile"。

我们使用下面的文件创建一个数据库，countries 文件的每行有四个字段，分别是 `country`, `area`, `population` 和 `continent`。capitals 文件中每行有两个字段，分别是 `country` 和 `capital`。

countries 文件：

	USSR    8649    275     Asia
	Canada  3852    25      North America
	China   3705    1032    Asia
	USA     3615    237     North America
	Brazil  3286    134     South America
	India   1267    746     Asia
	Mexico  762     78      North America
	France  211     55      Europe
	Japan   144     120     Asia
	Germany 96      61      Europe
	England 94      56      Europe

capitals 文件：

	USSR    Moscow
	Canada  Ottawa
	China   Beijing
	USA     Washington
	Brazil  Brasilia
	India   New Delhi
	Mexico  Mexico City
	France  Paris
	Japan   Tokyo
	Germany Bonn
	England London

文件的字段间使用 `\t` 分隔。

使用这两个文件，如果我们想查询所有亚洲的国家以及每个国家的人口和首都，我们需要查询这两个文件，并将结果合并。例如，可以使用下面的程序：

	awk ' BEGIN  { FS = "\t" }
		  FILENAME == "capitals" {
		      cap[$1] = $2
		  }
		  FILENAME == "countries" && $4 == "Asia" {
		      print $1, $3, cap[$1]
		  }
	' capitals countries

这个脚本显得有点复杂，如果我们可以直接使用字段名来取字段的值会简单的多，例如下面这样：

	$continent ~ /Asia/ { print $country, $population, $capital }

这就是 `q` 需要实现的功能。

### 自然连接

在关系型数据库中，一个文件被称为「表」，表中的列被称为「属性」。例如，`capitals` 有 `country` 和 `capital` 两个属性。

连接查询就是根据两个表的共同属性，将查询结果合并成一个结果集。结果集中两个表中用于连接的属性还需要进行去重处理。如果我们将 `countries` 和 `capitals` 表连接，叫做 `cc` ，那么 `cc` 有这些属性：

	country, area, population, continent, capital

`cc` 表中的每行记录都在两个表中存在对应的值，下面是 `cc` 表的值：

	Brazil	3286	134	South America	Brasilia
	Canada	3852	25	North America	Ottawa
	China	3705	1032	Asia	Beijing
	England	94	56	Europe	London
	France	211	55	Europe	Paris
	Germany	96	61	Europe	Bonn
	India	1267	746	Asia	New Delhi
	Japan	144	120	Asia	Tokyo
	Mexico	762	78	North America	Mexico City
	USA	3615	237	North America	Washington
	USSR	8649	275	Asia	Moscow

实现连接的方式是对每个表根据其连接属性字段进行排序，然后将两表中具有相同属性的行合并为一个新行，就组成了上面的表。如果一个查询需要跨越两个表进行连接，那么我们就可以先将两个表进行简单的初步连接，即上面的表。然后再对这个结果集应用查询语句。因此，这个表的作用就是查询的临时表。

那么，首先，我们要用 awk 实现一个连接两个文件的程序。它能自动根据每个文件的第一个字段属性进行连接。例如：

表1：

<table>
    <tbody>
        <tr>
            <td>ATT1</td>
            <td>ATT2</td>
            <td>ATT3</td>
        </tr>
        <tr>
            <td>A</td>
            <td>w</td>
            <td>p</td>
        </tr>
        <tr>
            <td>B</td>
            <td>x</td>
            <td>q</td>
        </tr>
        <tr>
            <td>B</td>
            <td>y</td>
            <td>r</td>
        </tr>
        <tr>
            <td>C</td>
            <td>z</td>
            <td>s</td>
        </tr>
    </tbody>
</table>

和表2：

<table>
    <tbody>
        <tr>
            <td>ATT1</td>
            <td>ATT4</td>
        </tr>
        <tr>
            <td>A</td>
            <td>1</td>
        </tr>
        <tr>
            <td>A</td>
            <td>2</td>
        </tr>
        <tr>
            <td>B</td>
            <td>3</td>
        </tr>
    </tbody>
</table>

连接的结果应为：

<table>
    <tbody>
        <tr>
            <td>ATT1</td>
            <td>ATT2</td>
            <td>ATT3</td>
            <td>ATT4</td>
        </tr>
        <tr>
            <td>A</td>
            <td>w</td>
            <td>p</td>
            <td>1</td>
        </tr>
        <tr>
            <td>A</td>
            <td>w</td>
            <td>p</td>
            <td>2</td>
        </tr>
        <tr>
            <td>B</td>
            <td>x</td>
            <td>q</td>
            <td>3</td>
        </tr>
        <tr>
            <td>B</td>
            <td>y</td>
            <td>y</td>
            <td>3</td>
        </tr>
    </tbody>
</table>

下面是用于这种连接的 awk 程序 `join`：

    #!/bin/awk -f
    # join - join file1 file2 on first field
    # input: two sorted files, tab-separated fields
    # output: natural join of lines with common first field

    BEGIN {
        OFS = sep = "\t"
        file2 = ARGV[2]
        ARGV[2] = ""        # 这样的目的是将 file2 放在后面处理，而不由默认的 awk 流程处理
        eofstat = 1         # file2 的读取状态
        if ((ng = getgroup()) <= 0)
            exit            # file2 为空则退出
    }

    { while (prefix($0) > prefix(gp[1]))
          if ((ng = getgroup()) <= 0)
              exit
      if (prefix($0) == prefix(gp[1])) # 第一个字段相同
          for (i = 1; i <= ng; i++)
              print $0, suffix(gp[i])  # 将结果合并 
    }

    function getgroup() {   # 将有相同前缀的条目放入到 gp[1..ng] 数组中 
        if (getone(file2, gp, 1) <= 0)  # 读取结束 
            return 0
        for (ng = 2; getone(file2, gp, ng) > 0; ng++) # ng 的值需要每次初始化为 2 
            if (prefix(gp[ng]) != prefix(gp[1])) {
                unget(gp[ng]) # 多读了，将这一行返回给下次读取
                return ng-1
            }
        return ng-1
    }

    function getone(f, gp, n) { # 读取下一行到 gp[n] 中
        if (eofstat <= 0)
            return 0
        if (ungot) {         # 如果为真则将上一次读取的结果返回而不读取新行
            gp[n] = ungotline
            ungot = 0
            return 1
        }
        return eofstat = (getline gp[n] < f)
    }

    function unget(s)  { ungotline = s; ungot = 1 }
    function prefix(s) { return substr(s, 1, index(s, sep) - 1) } # 前缀，即第一个字段
    function suffix(s) { return substr(s, index(s, sep) + 1) } # 后缀，即剩余的字段值

这个程序结构两个文件作为参数输入。以它们的第一个字段作为共同值进行合并。

`getgroup` 函数将 `file2` 中拥有相同第一字段的行放入到 `gp` 数组中，它调用 `getone` 来读取文件的每一行，如果第一字段不相同就停止读取，并调用 `unget` 将读取的行放入到一个临时变量 `ungotline` 中，供下次读取时重新读取。

### relfile

为了能够让数据库组织为多个文件，并支持跨文件查询，我们需要保存所有数据库表的描述信息。我们将这些信息存放在 `relfile` 文件中。`relfile` 包含表名，表的所有属性（字段名），如果一个表是临时表（如之前的 `cc` 表），则保存用于构建临时表的命令。`relfile` 的格式如下：

	tablename:
		attribute1
		attribute2
		...
		!command
		...

表名和属性都是字符串值。表名后面紧跟它的属性列表，以 `tab` 为前缀。属性过后是用于构建此表的命令，如果这个表不是临时表，那么就不需要构建命令，这样的表我们称为 `基表`。数据都应该保存在基表中。使用命令生成的表称为 `派生表`。派生表可以
根据查询的需求临时创建。

我们使用下面的 `relfile` 来保存我们数据库的元数据：

	countries:
	        country
	        area
	        population
	        continent
	capitals:
	        country
	        capital
	cc:
	        country
	        area
	        population
	        continent
	        capital
	        !sort countries > temp.countries
	        !sort capitals > temp.capitals
	        !./join temp.countries temp.capitals > cc

`relfile` 文件的总是包含一个全局关系，即一个包含所有属性的表，通常是 `relfile` 的最后一个表。这样就能保证总有一个表能够包含任意的属性的组合。`cc` 表就是 `countries-capitals` 数据库的全局关系表。

对于一个复杂的数据库来说，还应该能够处理各表的属性间的依赖关系，但是 "q" 只是一个简单的小型数据库查询语言，因此这里并没有考虑这么多。

### q, 一个类 awk 的数据库查询语言

我们的查询语言 `q` 由一个单行 awk 程序组成，但是它使用字段名，还不是类似 $1, $2 的位置参数。查询语言的解释器器 `qawk` 负责解析 `q` 查询，它应该完成这些功能：

1. 判断查询中的属性
2. 从 `relfile` 的起始处开始，寻找第一个能够包含所有查询语句中所涉及的所有属性的表。如果这个表是个 `基表`，那么就直接使用这个表所有查询的输入文件。如果这个表是个 `派生表`，那么就先使用响应的命令生成这个派生表，然后将其作为查询的输入。（这也意味着所有查询中涉及的属性的组合必须能够被 `relfile` 中的某个基表或派生表所完全包含。）
3. 它将 `q` 查询语句翻译成 awk 程序，即将所有的字段名称转换为相应的数字形式的位置参数。然后将这个程序处理第 2 步中的表，得到结果。

例如，下面的 `q` 查询：

	$continent ~ /Asia/ { print $country, $population }

涉及了 `continent`，`country` 和 `population` 三个属性，这些属性都在第一个表 `countries` 中存在，这个查询将被翻译成下面的 awk 语句：

	$4 ~ /Asia/ { print $1, $3 }

这个语句将作用于 `countries` 文件。

下面这个 `q` 语句：

	{ print $country, $population, $capital }

包含了 `country`，`population` 和 `capital` 属性，这些属性只被派生表 `cc` 完全包含。因此，查询处理器将使用 `relfile` 中的命令构建派生表 `cc` 并将 `q` 查询翻译成：

	{ print $1, $3, $5 }

这个语句将作用于刚创建的 `cc` 文件。

我们称 `q` 为查询语言，但是它也可以使用 `qawk` 进行计算操作（这本来就是 awk 支持的，因此 `q` 当然也能够使用），例如计算所有国家的平均面积：

	{ area += $area }; END { print area/NR }

### qawk, 将 q  翻译为 awk 程序的解释器

首先，`qawk` 读取 `relfile`，将所有的表明存放到 `relname` 数组中。它将所有构建派生表的语句存放在数组 `cmd` 中，`cmd` 从 `cmd[i, 1]` 开始计数。它还将每个表的所有属性放到二位数组 `attr` 中，`attr[i, a]` 保存了第 `i` 个表的属性 `a` 的字段号。

然后，`qawk` 读取查询语句，判断它需要哪些属性，这些属性是查询语句中的 `$name` 形式的字符串。通过调用 `subset` 函数，它将第一个包含所有查询所需属性的表保存在 `Ti` 中。它将这些属性名替换为对应的字段号，生成一个 awk 程序，执行对应的命令创建表 `Ti`，并将其作为输入，执行新生成的 awk 程序。

下面的图展示了 `qawk` 的工作过程：

![](/img/in-post/awk-relational-db/qawk.png)

`qawk` 程序的代码如下：

    #!/bin/awk -f
    # qawk - awk relaitonal database query processor

    BEGIN { readrel("relfile") }
    /./   { doquery($0) }

    function readrel(f) {
        while ((getline < f) > 0)
            if ($0 ~ /^[A-Za-z]+ *:/) {     # 匹配表名
                gsub(/[^A-Za-z]+/, "", $0)  # 移除其他内容，仅保留表名
                relname[++nrel] = $0
            } else if ($0 ~ /^[ \t]*!/)     # !command...
                cmd[nrel, ++ncmd[nrel]] = substr($0, index($0, "!")+1)
            else if ($0 ~ /^[ \t]*[A-Za-z]+[ \t]*$/)     # 匹配表的属性
                attr[nrel, $1] = ++nattr[nrel]
            else if ($0 !~ /^[ \t]*$/)      # 不符合格式的行 
                print "bad line in relfile:", $0
    }
    function doquery(s,    i, j) {
        for (i in qattr)                    # 先清空数组
            delete qattr[i]
        query = s                           # 将查询中的 $name 去除"$"后存入 qattr 数组
        while (match(s, /\$[A-Za-z]+/)) {
            qattr[substr(s, RSTART+1, RLENGTH-1)] = 1
            s = substr(s, RSTART+RLENGTH+1)
        }
        for (i = 1; i <= nrel && !subset(qattr, attr, i); )
            i++
        if (i > nrel)                       # 没有找到包含所有属性的表
            missing(qattr)
        else {                              # 第 i 个表能够包含查询中的所有属性
            for (j in qattr)                # 开始构造 awk 程序
                gsub( "\\$" j, "$" attr[i, j], query)
            for (j = 1; j <= ncmd[i]; j++)  # 创建第 i 个表（如果需要的话）
                if (system(cmd[i, j]) != 0) {
                    print "command failed, query skipped\n", cmd[i, j]
                    return
                }
            awkcmd = sprintf("awk -F'\\t' -v OFS='\\t' '%s' %s", query, relname[i])
            printf("query: %s\n", awkcmd)
            system(awkcmd)
        }
    }
    function subset(q, a, r,    i) {
        for (i in q)
            if (!((r, i) in a))
                return 0
        return 1
    }
    function missing(x,    i) {
        print "no table contains all of the following attributes:"
        for (i in x)
            print i
    }

我们用 `q` 写一个查询语句传递给 `qawk` 解释执行：

	[root@node1 awk]# echo '{ print $country, $population, $population/$area, $capital }' | ./qawk 
	query: awk -F'\t' -v OFS='\t' '{ print $1, $3, $3/$2, $5 }' cc
	Brazil	134	0.0407791	Brasilia
	Canada	25	0.00649013	Ottawa
	China	1032	0.278543	Beijing
	England	56	0.595745	London
	France	55	0.260664	Paris
	Germany	61	0.635417	Bonn
	India	746	0.588792	New Delhi
	Japan	120	0.833333	Tokyo
	Mexico	78	0.102362	Mexico City
	USA	237	0.0655602	Washington
	USSR	275	0.0317956	Moscow

大功告成！！可以看到目录下 `cc` 文件已经被自动生成了

	[root@node1 awk]# ls
	capitals  cc  countries  join  qawk  relfile

最后，我们的 `qawk` 程序还可以进一步优化，例如将创建派生表的语句合并为一条，使得只需要调用一次 `system()` 就能完成；在使用派生表之前先判断派生表是否已经创建，以避免重复创建派生表等等。












 
