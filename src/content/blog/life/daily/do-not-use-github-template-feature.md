---
title: 记一次基于 Template 创建仓库的爆炸经历
description: 不要轻易使用 GitHub 的 Template 功能，除非...你真的只是需要一个 template
date: 2026-01-20 22:47:03
tags:
  - Git
  - GitHub
  - VSCode
categories:
  - [随笔, 日常]
---

## 误用 Template 小故事

GitHub 的 Template 功能不会保留来源仓库的任何 history，也就是所有内容看起来都出自你自己，通过一个叫 `Initial commit` 的提交。

起因是今天上午给一个用了 template 的仓库（没错就是 blog repo）同步上游，寻思它本来就没有 history 对吧，干脆 squash merge 进去得了。

```sh
git remote add upstream https://github.com/cosZone/astro-koharu.git
git fetch upstream
git --squash --allow-unrelated-histories upstream/main
```

然后不出意外，就出意外了，这个 git 操作造成了 68 个 conflicts。对，虽然很不合理，但是它就是有 68 个，所有上游更改过和我更改过的文件全 conflict 了！

然后查了查才发现，原因就是没有 history 导致 Git 不知道这个更改是基于什么版本的，也不知道当前的版本跟更改有什么关系，只好把它标成 conflict 了——因为它没办法找 diff。

然后的然后，我下午又以同样的方式同步了一次，结果它又 conflict 了！这次倒不是因为没 history，哦，也可以说是因为没 history，因为 squash 并不会保留 history，而是直接把所有 commits 合并为一个 commit 交上去——因此仍然没办法找 diff。

因此请不要轻易使用 GitHub 的 Template 功能！否则你也会被 conflicts 炸掉的！

于是我就忆起很多 repo 明明有需要同步的需求但是还是把自己作为 template 提供，比如我的上游 [cosZone/astro-koharu](https://github.com/cosZone/astro-koharu)。遂在此呼吁此类仓库的作者不再将仓库作为 template 提供，而是建议使用者直接 fork，这样解决了更新的冲突问题，~~还能刷 fork 数量，一举两得~~。

## 适合 Template 的情况

也有的仓库是适合作为 template 提供的，那么，什么仓库适合呢？

答案是确实符合**模板**这个概念的仓库，比如一个 .NET 的 hello world 仓库，一个基于某个 SDK 的参考应用程序，一段调用 API 的示例代码，等等。这类仓库一定不会产生需要**后续同步更改**的需求，因为使用者只是参考仓库的设计思路、使用方式或是文件结构，实际内容全是使用者自己产生的。

因此有一个简单的判断方法，就是思考这个仓库的用户需不需要频繁同步更改。如果需要同步，就不适合 template，反之则通常是适合的。

## VSCode 小技巧

最后放一个我在解决冲突的时候发现的小技巧：VSCode 的 Merge Editor 是可以直接打开的，而不需要进文件编辑页面再手动点按钮。

方法很简单，去 VSCode 设置里搜索 `git.mergeEditor` 然后勾上就好了。
