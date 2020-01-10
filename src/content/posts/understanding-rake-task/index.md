---
title: Understanding rake task
date: "2014-05-26"
description: notes of learing rake task in ruby
---

在新项目中接触 ruby 的 Rake 任务很多，自己还是不太了解 Rake，闲了找了一些资料学习一下，做一些笔记。
其实 Rake 就是 ruby 中的 make，一个用 ruby 开发的构建工具。Rake 真的是一个功能强大，又很贴心的工具，正如它的英文意思*耙子*--给力的劳动工具。

### Rake 的优点

- 通过 Rake 我们可以以任务的方式创建和运行脚本。对于大型的应用项目，我们总是会编写很多脚本用来实现自动化，比如数据迁移，清空缓存，清理数据库等。Rake 就是一个非常好的任务脚本管理工具，会让你的任务脚本管理非常轻松。

- 同时 Rake 还轻松提供了任务执行之间的依赖关系管理，加入任务 1 依赖于任务 2，那么在执行任务 1 的时候任务 2 会自动被调用执行，非常方便。

### 安装 Rake

非常简单, gem 安装搞定！

```sh
$ gem install rake
```

### 学写 Rake 脚本

- 简单 rake 任务

创建一个 Rakefile，写一个简单的 rake 任务，从 hello world 开始吧。

```ruby
desc "say hello"
task :hello do
  puts "hello, world!"
end
```

然后在文件目录下执行 rake -T 就能看到这个任务, rake hello 就可以执行 rake 任务了。

```sh
➜  rake -T
rake hello  # say hello
➜  rake hello
hello world!
```

- 指定默认任务

在 Rakefile 中加入默认任务, 然后执行 rake 不指定任务名，就执行默认任务。

```sh
task :default => :hello
```

```sh
➜  rake
hello world!
```

- 创建具有依赖关系的任务

一般我们在执行一些脚本时候，会有一些预制条件，比如跑测试时候，需要提前在数据库中插入一些测试数据。在 rake 中，可以把预制条件写为一个任务，其他任务依赖于这个时候，可以把它加到依赖关系中

```ruby
desc "purchase vegetables"
task :purchase_vegetables do
	puts "Buy some potatoes from Walmart!"
end

desc "cook"
task :cook => :purchase_vegetables do
	puts "I am cooking..."
end
```

在这个例子中，做饭依赖于买菜，我们把给做饭加一个依赖任务， 这样在执行 cook 任务时候，purchase_vegetables 任务会被自动执行。

- 命名空间

如果在一个项目中有多个模块，由很多 rake 任务，那么我们可以用命名空间来防止 rake 任务名重复

```ruby
namespace "main" do
	desc "build the main programe"
	task :build do
		#build the main programe.
	end
end

namespace "module1" do
	desc "build the module1"
	task :build do
		#build the module1.
	end
end
```

这样即使同名的任务在一个 Rakefile 里，由于它们属于不同的 namespace，任务之间胡不相干。在调用的时候加上 namespace 名字即可。namespace 还支持嵌套。

- 带参数的 rake 任务

很多情况我们需要 rake 任务能够接受参数来扩展可用性，在 rake 0.8.0 版本后可以支持直接传入参数

```ruby
desc "build the programe"
task :build, [:programename, :programeversion] do |t, args|
	puts "building programe #{args.programename} with version #{args.programeversion} successful!"
end
```

查看 rake 任务时候可以看到 build 任务可以以数组的形式接受 2 个参数

```sh
➜ rake -T
rake build[programename,programeversion]  # build the programe
➜ rake build[mysite, 0.1.0]
building programe mysite with version 0.1.0 successful!
```

_Notes_
如果你用的是 zsh，那么可能会碰到 zsh 返回如下:

```sh
zsh: no matches found: build[mysite,0.1.0]
```

在 zsh 下必须用单引号把传递的参数引起来:

```sh
rake build'[mysite, 0.1.0]'
```

如果不想每次都这么麻烦，可以在~/.zshrc 中把 glob 对 rake 的扩展关掉:

```sh
alias rake="noglob rake"
```

这些都是 rake 的最基本用法，更多更高级的特性可以从 rake[官网](http://rake.rubyforge.org/)学习
