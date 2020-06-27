---
title: 借 Git 的第一个 commit 探索 Git 原理
date: 2020-06-27 20:44:07
tags: [Git]
---

最近想了解一下 Git 的实现，首先是看了 pro git 的 [git 原理部分](https://git-scm.com/book/en/v2/Git-Internals-Plumbing-and-Porcelain)。看完之后对 git 的实现有了个大概的了解，但还是不过瘾。于是把 git 的源码 clone 下来看源码是如何实现的，但是 git 的源码实在太多了，要耗费太多精力了。

我尝试把跳转到 Git 的第一个 commit，代码量就少了很多，只有差不多 1000 行左右。修改了一些代码把程序编译起来了。看了下初始版本的 git，发现很多概念还是可以和现在的 git 的相通，有阅读的价值。

之后还看到 jacob Stopak 的解析，对 git 的代码做了很多注释。看到的时候很是惊喜，因为网上对 git 这样的解析很少，看日期还是今年的 5 月 22 号发布的，新鲜的 >_<。

[What Can We Learn from the Code in Git’s Initial Commit?](https://bitbucket.org/blog/what-can-we-learn-from-the-code-in-gits-initial-commit)

## 获取源代码并编译

在这里使用 Jacob Stopak 的项目，它对 Git 的第一个 commit 里面的代码添加了大量注释，还修改了一些代码方便我们在现代操作系统上编译。

获取代码

```
git clone https://bitbucket.org/jacobstopak/baby-git.git
```

编译直接 make 就好了，不过这里还是会出现编译错误，需要在 `cache.h` 文件修改下面的变量，加上 extern

```
extern const char *sha1_file_directory;
extern struct cache_entry **active_cache;
extern unsigned int active_nr, active_alloc;
```

直接 make

```
make
```

如果你想要尝试使用 github 上 git 的源码来编译的话，可以进行下面操作

```
# 先 clone 源码
git clone https://github.com/git/git.git
# 根据 log 找到第一个 commit
git log --reverse
# 检出第一个 commit
git checkout e83c5163316f89bfbde7d9ab23ca2e25604af290
```

这里的源码直接 make 的话会编译失败，还需要作出下面的修改

首先是 Makefile 文件

```
# 把 LIBS 这行修改成这样
LIBS= -lcrypto -lz
```

然后在 cache.h 中

```
#添加头文件
#include <string.h>
```

和前面同样的操作，把下面几个变量加上 extern

```
extern const char *sha1_file_directory;
extern struct cache_entry **active_cache;
extern unsigned int active_nr, active_alloc;
```

之后就可以直接 make 了。或者你嫌麻烦可以用我 [z.diff](z.diff) 文件，直接 apply 一下。

```
git apply z.diff
```

效果一样。

## 源码结构

可以看到 git 的开始的时候只有 8 个 .c 文件和 1 个 .h 文件。

可以看到一下包含代码和注释行数只有 1037 行

```shell
cat *.c *.h | wc -l
1037
```

其中 7 个 .c 文件可以对应现在的 7 个 git 命令

| 文件         | 当前 Git       | 作用                                                   |
| ------------ | -------------- | ------------------------------------------------------ |
| init-db      | git init       | 初始化 git 仓库                                        |
| update-cache | git add        | 添加文件到暂存区                                       |
| write-tree   | git write-tree | 将暂存区的内容写入到一个 tree 对象到 git 的仓库        |
| commit-tree  | git commit     | 基于指定的 tree 对象创建一个 commit 对象到 git 的 仓库 |
| read-tree    | git read-tree  | 显示 git 仓库的树对象内容                              |
| show-diff    | diff           | 显示暂存的文件和工作目录的文件差异                     |
| cat-file     | git cat-file   | 显示存储在 Git 仓库中的对象内容                        |

至于 read-cache.c 文件则定义了一些程序一些公用的函数和几个全局变量。

## 概念分析

Linus Torvalds 中 readme 中对 git 的实现原理作出了一些解释。

首先它给出了为什么要使用 git 这个名字的原因

- 随机的三个字母组合，没有和其他的 unix 命令冲突
- 简单
- 当它好使的时候叫 global information tracker
- 不好使的时候叫 goddam idiotic truckload of sh*t

> - random three-letter combination that is pronounceable, and not
>   actually used by any common UNIX command.  The fact that it is a
>   mispronounciation of "get" may or may not be relevant.
> - stupid. contemptible and despicable. simple. Take your pick from the
>   dictionary of slang.
> - "global information tracker": you're in a good mood, and it actually
>   works for you. Angels sing, and a light suddenly fills the room. 
> - "goddamn idiotic truckload of sh*t": when it breaks

它有两个核心的概念

- objects databases
- current directory cache （相当于现在的暂存区）

### Object Databases

object database 相当于一个基于文件系统的键值对数据库，用文件的 sha-1 值作为键，在 `.dircache/objects` 中存放各种类型的数据。

在这里介绍三种类型：

- blob
- tree
- commit

blob 对象用来存储完整的文件内容，用于 git 追踪文件内容。blob 的结构

```
blob size\0blobdata(file content)
```

git 除了要追踪文件内容外还要追踪文件名、文件存储路径和权限属性之类的信息。这个时候我们引入 tree 对象来存储这些信息。tree 只会保留 blob 对象的 sha-1 值，不会追踪文件内容。（现在的 tree 对象还会可以 tree 对象嵌套 tree 对象，现在这里还没有）

大致的结构是这样的（省略了头）

```
100644 a.txt (6e666502660a7e810b276afd62523c56b34c1671)
100644 b.txt (b 的 sha-1 值)
100644 c.txt (c 的 sha-1 值)
```

commit 对象，可以理解为我们的一个 commit，它记录了特定时间的目录树（tree 对象）、父 commit 的 sha-id（可以有多个或 0 个）、作者和提交者信息和对应的 commit message。git 通过 commit 对象来追踪存储库的完整历史开发记录。

同样是大致结构：

```
tree 1c93ac491de01f734fedbe70f31c47ca965c93b6
parent 4ae1f8aae02d7178aa4fc52f45f068cd750aa3de
author  <zzk@archlinux> Sat Jun 27 14:41:43 2020
committer  <zzk@archlinux> Sat Jun 27 14:41:43 2020

second commit
```

现在的 git 同样还是一直在使用上面这些概念，通过管理这些对象来实现我们的版本控制。

Git 把每个文件的版本都完全保存下来，会不会占用很多空间？是不是太暴力了？

git 会使用 zlib 对所有对象进行压缩，代码这类文本数据可以做到很高的压缩率（因为代码中很多都是重复的，比如关键字、函数调用）。在现在的 git 还引入了 packfiles 机制，会查找找命名及大小相近的文件，只保存文件不同版本之间的差异，也是可以只保存 diff 的。关于暴力的话，就是空间换时间方式了，这样做在版本跳转的时候很快，不用一个一个地应用 diff，直接拿出来就好了。

Git 如何存储这些对象？

它们都被存放在 `.dircache/objects` 文件夹中，通过内容的 sha-1 值来索引对用的对象。在 `dircache/objects` 下还会生成 256 个子目录（00～ff），用来索引前两位的 sha-1 值。

### Current directory cache

current directory cache 就是我们使用的暂存区，它存储在 `.dircache/index` 文件中，可以看作是一个临时的 tree 对象。当执行 update-cache 时，就会创建对应文件的 blob 对象，并将树信息加到 `.dircache/index` 文件中。

## 如何使用

前面说的那些可能没有实际使用会不太好理解，下面我们可以实战一下，使用一些初始版本的 git。

在这里我们会利用这些现有的命令

- 把暂存区的文件取出来（相当于 git checkout -- file）
- 提交两个 commit 并在这两个 commit 的之间切换

先使用 init-db 创建存储库

```
$ ./init-db
```

可以看到创建了 .dircache 文件夹，和现在的 .git 文件是一样的。

创建一个文件写入到暂存区，执行 update-cache 会在 `.dircache/objects` 文件夹中生成一个 blob 对象并把它加入到 `.dircache/index` 暂存区中。

```
$ echo "123456" > a.txt
$ ./update-cache a.txt
```

使用 find 查看创建的 blob 对象

```
$ find .dircache/objects -type f
.dircache/objects/6e/666502660a7e810b276afd62523c56b34c1671
```

查看一下

```
$ cat .dircache/objects/6e/666502660a7e810b276afd62523c56b34c1671
xKOR0g0426156%
```

由于是用 zlib 压缩过的没解压看不出来啥，我们可以使用 cat-file 查看

```
$ ./cat-file 6e666502660a7e810b276afd62523c56b34c1671
temp_git_file_O5J3ZL: blob
```

他会生成一个临时文件，里面是解压好的文件内容，可以看到我们保存到存储库里 a.txt 的内容

```
$ cat temp_git_file_O5J3ZL
123456
```

现在我们把当前的暂存区的内容保存为 tree 对象到 `.dircache/objects` 中

```
$ ./write-tree
433aef473a665a9efe1cf21fbc617fbf833c71b5
```

写入成功会打印对象的 SHA-1 值，我们可以用 read-tree 查看这个 tree 对象有什么

```
$ ./read-tree 433aef473a665a9efe1cf21fbc617fbf833c71b5
100644 a.txt (6e666502660a7e810b276afd62523c56b34c1671)
```

现在可以提交我们的第一个 commit 了，填写完 commit message 后按 crtrl+d 退出

```
$ ./commit-tree 433aef473a665a9efe1cf21fbc617fbf833c71b5
Committing initial tree 433aef473a665a9efe1cf21fbc617fbf833c71b5
first commit 
4ae1f8aae02d7178aa4fc52f45f068cd750aa3de
```

可以查看一下 commit 对象有什么东西

```
$ ./cat-file 4ae1f8aae02d7178aa4fc52f45f068cd750aa3de
temp_git_file_pNLmni: commit
$ cat temp_git_file_pNLmni
tree 433aef473a665a9efe1cf21fbc617fbf833c71b5
author  <zzk@archlinux> Sat Jun 27 14:20:55 2020
committer  <zzk@archlinux> Sat Jun 27 14:20:55 2020

first commit
```

我们完成了第一个提交，现在我们对 a.txt 做一下修改

```
$ echo "version2" > a.txt
```

用 show-diff 可以查看和暂存区的版本的差异

```
$ ./show-diff
a.txt:  6e666502660a7e810b276afd62523c56b34c1671
--- -   2020-06-27 14:27:19.975168003 +0800
+++ a.txt       2020-06-27 14:27:18.048129094 +0800
@@ -1 +1 @@
-123456
+version2
```

如果想取出暂存区的文件，有两个方法

- 使用生成的 diff，修改文件
- 用 cat-file 取出暂存区文件内容（第二行有文件的 SHA-1 值）

先用 diff 试试

```
$ ./show-diff > a.diff
$ patch --reverse a.txt a.diff
patching file a.txt
$ cat a.txt
123456
```

可以看到文件又回到暂存区的版本了。现在把文件再修改回去，使用 cat-file 来试试

```
$ echo "version2" > a.txt
$ ./show-diff
a.txt:  6e666502660a7e810b276afd62523c56b34c1671
--- -   2020-06-27 14:33:53.795263485 +0800
+++ a.txt       2020-06-27 14:33:50.726893274 +0800
@@ -1 +1 @@
-123456
+version2
$ ./cat-file 6e666502660a7e810b276afd62523c56b34c1671
temp_git_file_d4cTZ7: blob
$ mv temp_git_file_d4cTZ7 a.txt
$ cat a.txt
123456
```

现在我们准备第二个 commit ，再执行一遍之前的操作

```
$ echo "verison2" > a.txt
$ ./update-cache a.txt
$ ./write-tree
1c93ac491de01f734fedbe70f31c47ca965c93b6
```

执行 commit-tree 时候要注意，由于这是第二个提交，需要用 -p 指定一下父提交的 SHA-id 也就是第一个 commit

```
$ ./commit-tree 1c93ac491de01f734fedbe70f31c47ca965c93b6 -p 4ae1f8aae02d7178aa4fc52f45f068cd750aa3de
second commit
3d4340867cb1dd857987fc2db9b1b4bc14af7051
```

看一下这个 commit

```
$ ./cat-file 3d4340867cb1dd857987fc2db9b1b4bc14af7051
temp_git_file_8Cguru: commit
$ cat temp_git_file_8Cguru 
tree 1c93ac491de01f734fedbe70f31c47ca965c93b6
parent 4ae1f8aae02d7178aa4fc52f45f068cd750aa3de
author  <zzk@archlinux> Sat Jun 27 14:41:43 2020
committer  <zzk@archlinux> Sat Jun 27 14:41:43 2020

second commit
```

现在我们要把 a.txt 的版本切换到上一个 commit，从上面的输出可以找出 parent 提交的 SHA-id

我们利用这个父提交的 sha-id 找到对应的 tree object 的 sha-id，再从 tree object 找到对应 a.txt 的 blob object 的 sha-id。最后利用 cat-file 就可以还原出来原始 a.txt 的内容了。

```
$ ./cat-file 4ae1f8aae02d7178aa4fc52f45f068cd750aa3de
temp_git_file_MssRdY: commit
$ cat temp_git_file_MssRdY
tree 433aef473a665a9efe1cf21fbc617fbf833c71b5
author  <zzk@archlinux> Sat Jun 27 14:20:55 2020
committer  <zzk@archlinux> Sat Jun 27 14:20:55 2020

first commit
$ ./read-tree 433aef473a665a9efe1cf21fbc617fbf833c71b5
100644 a.txt (6e666502660a7e810b276afd62523c56b34c1671)
$ ./cat-file 6e666502660a7e810b276afd62523c56b34c1671
temp_git_file_2RlDAI: blob
$ mv temp_git_file_2RlDAI a.txt
$ cat a.txt
123456
```

现在 a.txt 就被还原到了 第一个 commit 时的状态。

总结：原始的 git 很难用。经过不断地打磨才成了今天这样子好用，现在的 git 很多人只要会 add、commit、push 就完事了，也不用了解 git 原理。

## 代码分析

这里对 git 的部分源码进行分析，由于前面已经写了很多篇幅了，这里就不会对太多的代码进行分析，里面的代码也是超简单，可以说是 linus 代码写得漂亮。你自己看估计也花不了一会儿，而且 Jacob Stopak 对代码做超多的注释，真的没啥好说的。 甚至你可以根据前面的知识脑补出大概源码了>_<。

在这里推荐一个阅读源码的工具 [sourcetrail](https://www.sourcetrail.com)。看源码时很方便。

init-db.c

```
int main(int argc, char **argv)
{
	char *sha1_dir = getenv(DB_ENVIRONMENT), *path;
	int len, i, fd;
	
	// 创建 .dircache 的目录
	if (mkdir(".dircache", 0700) < 0) {
		perror("unable to create .dircache");
		exit(1);
	}

	/*
	 * If you want to, you can share the DB area with any number of branches.
	 * That has advantages: you can save space by sharing all the SHA1 objects.
	 * On the other hand, it might just make lookup slower and messier. You
	 * be the judge.
	 */
	sha1_dir = getenv(DB_ENVIRONMENT);
	if (sha1_dir) {
		struct stat st;
		if (!stat(sha1_dir, &st) < 0 && S_ISDIR(st.st_mode))
			return 1;
		fprintf(stderr, "DB_ENVIRONMENT set to bad directory %s: ", sha1_dir);
	}

	/*
	 * The default case is to have a DB per managed directory. 
	 */
	sha1_dir = DEFAULT_DB_ENVIRONMENT;
	fprintf(stderr, "defaulting to private storage area\n");
	len = strlen(sha1_dir);
	if (mkdir(sha1_dir, 0700) < 0) {
		if (errno != EEXIST) {
			perror(sha1_dir);
			exit(1);
		}
	}
	path = malloc(len + 40);
	memcpy(path, sha1_dir, len);
	// 创建 .dircache/objects 下的 256 个子目录
	for (i = 0; i < 256; i++) {
		sprintf(path+len, "/%02x", i);
		if (mkdir(path, 0700) < 0) {
			if (errno != EEXIST) {
				perror(path);
				exit(1);
			}
		}
	}
	return 0;
}
```

init-db 就只是单纯创建好这些目录而已，和我们前面看到的效果也一样。

这里介绍 update-cache.c 文件，主要说一下怎么创建 blob 和加入 `.dircache/index` 文件。

首先看 main

```c
int main(int argc, char **argv)
{
	int i, newfd, entries;
	
    // 将 .dircache/index 中的内容读入到 active_cache 这个全局变量中
	entries = read_cache();
	if (entries < 0) {
		perror("cache corrupted");
		return -1;
	}
	
    // 这里创建 lock 文件是为了不让多个 update-cahce 同时运行的锁文件
	newfd = open(".dircache/index.lock", O_RDWR | O_CREAT | O_EXCL, 0600);
	if (newfd < 0) {
		perror("unable to create new cachefile");
		return -1;
	}
	for (i = 1 ; i < argc; i++) {
		char *path = argv[i];
        // 验证路径
		if (!verify_path(path)) {
			fprintf(stderr, "Ignoring path %s\n", argv[i]);
			continue;
		}
        // 把文件添加到 object 数据库中
		if (add_file_to_cache(path)) {
			fprintf(stderr, "Unable to add %s to database\n", path);
			goto out;
		}
	}
    // 更新 index 文件
	if (!write_cache(newfd, active_cache, active_nr) && !rename(".dircache/index.lock", ".dircache/index"))
		return 0;
out:
	unlink(".dircache/index.lock");
}
```

通过 main 函数可以看到生成 blob 应该是在 add_file_to_cache 中做的、而更新 index 则是在 write_cache 中做的。后面的可以自己追踪一下。我说一下大概做了啥

add_file_to_cache 中读取文件然后构造一个 blob 对象进行压缩，最后计算 sha-1 值，把 压缩好的 blob 对象存到 sha-1 值对应的位置。（现在看文档好像是先计算 sha-1 值再进行压缩了）之后通过文件名二分查找放入 active_cache 中（也就是 .dircahce/index 暂存区，这个变量是个全局变量开始时将 .dircache/index 中的内容读入到里面），因为是暂存区是通过文件名进行排序的，所以确认一个文件是否在暂存区很快，二分只要 logn。

write_cache 中就是把 active_cache 再写回 `.dircache/index.lock` 文件，最后把锁文件重命名为  `.dircache/index` 暂存区文件。