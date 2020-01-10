---
title: Strong parameters in rails4
date: "2013-11-10"
description: rails learning notes
---

在 rails 中有很多安全相关的特性保证我们的 web 应用足够安全，今天在学习 rails 时候碰到了这个，当我试图用一个 create 方法创建一个新话题的时候，我遇到了麻烦，我的 controller 如下：

```ruby
def new
  @topic = Topic.new
end

def create
  Topic.create(params[:topic])
  redirect_to root_path
end
```

在用户提交新的话题后，rails 会报错提示

`ActiveModel::ForbiddenAttributesError`

在查阅了 railsguides 后才知道，这里不能直接使用 create(params[:topic])，因为如果这样写，在 web 应用中这样可以随便让任何参数都传进来从而导致不安全的结果。在这里我们要用的是 strong_parameters，我们需要指定 controller 只允许接受哪几个参数

```ruby
def create
  Topic.create(params[:topic].permit(:title, :content, :node_id))
end
```

这样再刷新页面应该就 ok 了。不过在这个过程中我还遇到了一个问题，我当时只传入了 title 和 content 2 个参数，忘了把 node_id 传过去了，结果刷新页面就报错，怎么也找不到问题，后来尝试把默认传过去的参数打印出来，像这样

```ruby
def create
  render text: params[:topic].inspect
end
```

在提交之后就能在浏览器中看到传过去的参数以及取值，再来考虑 create 方法中应该怎么写了。

但其实上面那个还不是最好的写法，你应该这样写

```ruby
def create
  Topic.create(topic_params)
end

private
  def topic_params
    params.require(:topic).permit(:title, :content, :node_id)
  end
```

在这里我们定义一个私有方法`topic_params`，然后在 create 里调用这个方法来实现 topic 所带的参数的传递，这样就避免了通过修改 hash 传递给 model 来设置 model 属性的风险。更多的可以参考[这里](http://weblog.rubyonrails.org/2012/3/21/strong-parameters/)。
