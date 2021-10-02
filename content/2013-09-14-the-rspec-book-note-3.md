+++
date = 2013-09-14
description = "rspec learning notes"
title = "The rspec book note 3"
+++

BDD行为驱动开发，一般是指从软件的外在行为出发进行功能描述，从而推动代码开发并最终实现软件交付的过程。首先我们需要从客户的角度去理解需求，去搞清楚他们面临的问题，从他们对期望的软件描述中来获取有价值的信息。其次我们不要只关注主要客户的需求，任何对于我们将要做的项目有浓厚兴趣并提出一些看法的人我们都需要理解他们的意见。

当我们向客户交付有价值的软件的时候，我们需要意识到，我们交付的东西要能够解决客户的问题，但是也不能过度设计。BDD有一系列自己的原则需要我们遵守： 

## Enough is enough  
从前期的需求分析，设计开始我们需要多做获取一个好的开始，但是也不能做过了，因为那样是一种浪费，刚刚好就是最好。 

## Deliver stakeholder value  
如果你做的东西既不能给客户带来价值也不能锻炼你交付有价值软件的能力，那么还是歇歇吧，干点别的啥。 

## It's all behavior 
不管是从代码，应用程序，还是更高的层次来讲，我们都可以用相同的思维方式和语言结构从不同的颗粒度去描述我们的软件的外在行为。

BDD中定义任何关心软件的人价值的人都可以作为利益相关者，当他们与BA讨论了他遇到的问题和面临的困境后，会与测试人员来决定故事场景，我理解的故事场景其实越简单越好，就是一个很简单的软件的对外呈现的行为，然后选择出优先级最高的故事，开发人员只需要实现故事中最简单的场景即可，不要做过多的开发工作。在开始写代码之前最重要的的事情就是我们要让这些故事场景全部自动化，我们需要Cucumber。然后我们就可以开始编码，使用rspec进行TDD直到场景实现，然后重构，然后再次迭代直到其他场景也都实现，最终整个故事全部完成。

# 一个故事应该是什么样的？  

### A title   
它应该有个标题，来说明我们做的是什么。 

### A narrative  
一般的描述是作为一个怎样的角色，我需要一个什么样的东西，这样我就能够干什么。

### Acceptance criteria
有了验收标准我们才知道什么样的行为就算是ok了。 

在软件交付的节奏中我们把需求分解为特性然后再变成故事和一堆场景，我们把它们全都写成自动化测试用例，就可以作为验收用例来保证我们做的软件是我们期望的。