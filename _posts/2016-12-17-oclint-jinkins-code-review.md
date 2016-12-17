---
layout: post
date: 2016-12-17
title: 使用 oclint + Jenkins 进行自动化 Code Review
disqus: y
---

## 前言

Code Review 是保证代码质量的一个十分有效的手段，笔者公司由于开发人员人数增加，代码质量良莠不齐，所以打算进行 Code Review。在项目迭代比较频繁的时候，就没有太多的时间去 Review 其他人的代码。所以打算使用 oclint + Jenkins 进行代码 Review。还有一个前人造的坑就是，公司的项目十分糟糕，代码十分不规范，为了让进行迭代的同学不受历史代码的影响，所以只对当前分支修改的文件进行 Review。

## 查找改变的文件

我们公司目前使用的是 `gitflow` 进行项目的管理，所以，要找到改变的文件只需要把当前开发的分支与`develop` 分支进行对比即可找到改变的文件。

```bash
git diff --name-only dev $(git merge-base dev master) | grep '^[^(Pods/)].*\.m$'
```

## OCLint

oclint 是一个静态代码分析器，oclint 本身已经有很多 Rules，当然你也可以根据接口开发自己的 Rules。

通过 oclint 对代码进行分析，生成报告，命令如下:

```bash
report_file_o="./report_result.$type"
oclint -report-type html -R ./rules -o report.html -rc LONG_METHOD=30 -- -x objective-c -std=gnu99 -fobjc-arc
```
* report-type: report 文件类型，可以是 `html`、`pmd` 等
* `-R` 后面传入 rules 的未知
* 通过 `-rc` 可以配置某些 rules 的阈值，例如：`LONG_METHOD` 表示方法的代码行限制，具体可以参考 [OCLint Documents](http://docs.oclint.org/en/stable/howto/thresholds.html)

## Jenkins
Jenkins 是一个开源的持续集成(CI)工具，主要用于持续、自动的构建/测试软件项目等。

这里我们主要是使用了 Jenkins 的  `PMD Plugin`，用来浏览 OCLint 分析出 pmd 格式的报告。

具体的配置方法，直接 [Google](https://google.com) 就行。

## 本地 Review

由于 xcode8 的到来，很多插件都不能使用了，oclint 的插件貌似也没能幸免，就不能通过点击 xcode 的 build 进行本地的 code review 了，这里为了方便使用，笔者就学了一会儿 `shell` ，写了一个脚本，进行本地的 code review。代码如下：

```shell

## 检查 Git 是否存在
git=$(which git)
[[ -x "$git" ]] || { printf "\e[1;31mGit Not Found,  \"brew install git\"...\e[0m\n";exit 1; }

## 检查 OCLint 是否存在
lint=$(which oclint) 
[[ -x "$lint" ]] || { printf "\e[1;31mOCLint Not Found, use \"brew tap oclint/formulae && brew install oclint\" to install\e[0m\n"; exit 1; }

## 获取当前分支

branch=$(git rev-parse --abbrev-ref HEAD)
echo "Current Branch: $branch"

## 在本分支修改的文件
# files=$(git diff --name-only dev $(git merge-base dev master) | grep '^[^(Pods/)].*\.m$')

## 在本分支新增的文件(Pods 除外)
files=$(git diff --name-only --diff-filter=A master $branch | grep '^[^(Pods/)].*\.[mh]$')
[[ -n "$files" ]] || { printf "\e[1;36m没有新增文件\e[0m\n"; exit 0; }
commnadFiles=""
echo "\nAnalized Files:\n--------------------------------"
for file in $files; do
    if [[ -f $file ]]; then
        echo $file
        commnadFiles=" $commnadFiles $file"
    fi
done
echo "--------------------------------\n"

## 输出类型，默认 html
type=$1
[[ -n $1 ]] || type='html'
echo "Report Type: $type"

report_file_o="./report_result.$type"
oclint $commnadFiles -report-type $type -R ./rules -o $report_file_o \
-rc LONG_METHOD=50 \
-rc TOO_MANY_PARAMETERS=8 \
-- -x objective-c -std=gnu99 -fobjc-arc

if [ $? -eq 0 ]; then
    printf "\e[1;36m报告生成成功！\e[0m\n"
else
    printf "\e[1;31m报告生成失败\e[0m\n"
    exit 1
fi
```

只需要进入项目目录，在 terminal 中输入：

```bash
./codereview.sh
```

## Jenkins Code Review

如何在 Jenkins 上进行代码 Review 生成报告呢，笔者依然使用的是本地 review 的 shell 脚本进行代码分析，使用 PMD 插件对结果进行预览。配置如下：

![config](http://obb77efas.bkt.clouddn.com/oclint-jinkins/jinkins-config.png)

构建成功后 PMD 插件预览界面是这样的：
![](http://obb77efas.bkt.clouddn.com/oclint-jinkins/pmd-plugin.png)

点击任意一个 issue，可以查看代码：
![](http://obb77efas.bkt.clouddn.com/oclint-jinkins/pmd-report-code.png)

## 参考

- [OCLint 官方文档](http://docs.oclint.org/en/stable/index.html)
- [Jenkins 配置](https://ruby-china.org/topics/30632)
