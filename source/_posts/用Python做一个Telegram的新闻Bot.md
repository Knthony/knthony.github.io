---
title:  用Python做一个Telegram的新闻Bot
date: 2020/6/10 12:04:23
categories:
- Tech
tags:
- python
- telegram
- bot
---



> 在这篇文章中，我们构建一个高可用的Telegram机器人，用来在成千上万的新闻源当中搜索内容。



## 介绍



Telegram 现在是一个全球流行的实时消息应用，它的特点是安全性和高效性，除了可以互相发送消息以外，还可以在它上面创建bot用来自动执行一些任务。



在这个教程中，我们将用Python和Telegram's bot API 创建一个基于Datanews的新闻bot。



## Telegram API 简单介绍



我们将使用官方的`python-telegram-bot`Api，这个库大大简化了开发Bot的工作，而且通过一些官方给出的例子，很容易学习和掌握它，下面是一个例子：

```python
from telegram.ext import Updater, CommandHandler


USAGE = '/greet <name> - Greet me!'


def start(update, context):
  update.message.reply_text(USAGE)


def greet_command(update, context):
  update.message.reply_text(f'Hello {context.args[0]}!')


def main():
  updater = Updater("TOKEN", use_context=True)
  dp = updater.dispatcher 

  # on different commands - answer in Telegram 
  dp.add_handler(CommandHandler("start", start)) 
  dp.add_handler(CommandHandler("greet", greet_command)) 

  # Start the Bot 
  updater.start_polling() 
  updater.idle() 


if __name__ == '__main__': 
  main()
```

这个小代码片段创建了一个bot用来识别两个命令：

1. `/start` - Bot会根据这个命令响应出帮助页面
2. `/greet`- 这个命令会接受一个参数，例如`Datanews`,然后返回`Hello Datanews!`

来看看每一行代码的含义：

`main`方法

```python
def main():
  updater = Updater("TOKEN", use_context=True)
  dp = updater.dispatcher

  dp.add_handler(CommandHandler("start", start))
  dp.add_handler(CommandHandler("greet", greet_command))

  updater.start_polling()
  updater.idle()
```

这个方法配置了一些Bot工作必要的参数，特别是`Update`类的实例，需要注意的是，你需要一个Telegram的`token`才能使用Telegram Bot 的API，你可以查看这个创建Bot的官方指南[点这里](https://core.telegram.org/bots#3-how-do-i-create-a-bot)。

回到代码，`Updater`的目的是为了将更新传递给`Dispatcher`，当后者收到一个更新，它将处理用户指定的一些回调请求，每一个回调都由一个程序管理。

当收到的消息满足某些条件时，虽然这些条件取决于程序，也可以由开发者自己定义，就我们而言我们有两个任务处理实例。

```python
dp.add_handler(CommandHandler("start", start))
dp.add_handler(CommandHandler("greet", greet_command))
```

上面的每一条处理一个命令，用来支持我们Bot的`/start`和`/greet`

然后我们调用`start_polling`命令。

`updater.start_polling()`

这个命令会让我们的bot周期性的处理更新，这个参数会在内部创建两个进程，一个用来从Telegram服务器轮询更新，另一个将由调度程序处理这些更新。



下面这一行确保我们的bot可以正确的处理各种中断信号

`updater.idle`

现在我们讨论两个处理bot命令的回调函数：

```python
def start(update, context):
  update.message.reply_text(USAGE)


def greet_command(update, context):
  update.message.reply_text(f'Hello {context.args[0]}!')
```

上面每一个方法都需要两个参数：

1. `update` 从Telegram服务器收到一个更新
2. `context`包含一些有用的参数和信息，举个例子，它有一个用来储存用户相关信息的`user_data`字典

除此以外，每一个参数都会给用户返回一条消息。



## Datanews API 介绍

Datanews 是一个用来从成千上万个新闻源中取回和监控新闻的API，新闻聚合器和其他网站，每天收集并处理了数十万的新闻数据，当然，它也提供了灵活和简单可用的API用来检索这些新闻文章。

对于我们这个小项目，我们只需要API中的一小部分，特别是我们想让bot能做什么：

1. 根据用户输入的关键字返回新闻内容
2. 根据特定的新闻源返回新闻内容

这些用例需求可以用一个入口处理- `/headlines`, 你可以在后面链接中了解到更多相关的API[official documentation](https://datanews.io/docs).

## 程序实施

首先，我们要定义一个处理`/start`命令的回调函数

```python
def get_usage():
  return '''This bot allows you to query news articles from Datanews API.

Available commands:
/help, /start - show this help message.
/search <query> - retrieve news articles containing <query>.
  Example: "/search covid"
/publisher <domain> - retrieve newest articles by publisher.
  Example: "/publisher techcrunch.com"'''

def help_command(update, context):
  update.message.reply_markdown(get_usage())
```

就像你看到的，这个实现方式与我们上面演示的很相似，我们简单的返回了一个help信息给用户，你可以注意到我们的bot支持四个命令，下面我们讨论其他两个：

```python
def search_command(update, context):
  def fetcher(query):
    return datanews.headlines(query, size=10, sortBy='date', page=0, language='en')
  _fetch_data(update, context, fetcher)


def publisher_command(update, context):
  def fetcher(query):
    return datanews.headlines(source=query, size=10, sortBy='date', page=0, language='en')
  _fetch_data(update, context, fetcher)
```

这些方法看起来都非常简单，它们都用了我们前面讨论过的`/headlines`API，唯一的区别是我们传递给Datanews API的参数：search_command检索与特定查询匹配的文章，而Publisher_command提取所有由特定来源发布的文章，这两个方法中我们都只活去10条最近的文章。

```python
def _fetch_data(update, context, fetcher):
  if not context.args:
    help_command(update, context)
    return

  query = '"' + ' '.join(context.args) + '"'
  result = fetcher(query)

  if not result['hits']:
    update.message.reply_text('No news is good news')
    return

  last_message = update.message
  for article in reversed(result['hits']):
    text = article['title'] + ': ' + article['url']
    last_message = last_message.reply_text(text)
```

这个方法简单的检查了用户输入的特定的参数，从Datanews API中取出数据并排序返回给用户，

1. 我们确保用“包围查询”，以便Datanews返回所有包含完整查询的文章，而不仅仅是其中的一个单词。
2. 我们还要确保处理无法查询到文章的情况，如果这种情况下bot无反应是扯淡的。
3. 我们发送给用户的内容必须确保排序后的内容最后一条是最新的内容。

有了这些功能，我们看一下主程序：

```python
def main():
  updater = Updater(token='TOKEN')

  updater.dispatcher.add_handler(CommandHandler('start', help_command))
  updater.dispatcher.add_handler(CommandHandler('help', help_command))
  updater.dispatcher.add_handler(CommandHandler('search', search_command))
  updater.dispatcher.add_handler(CommandHandler('publisher', publisher_command))

  updater.dispatcher.add_handler(
    MessageHandler(
      Filters.text & Filters.regex(pattern=re.compile('help', re.IGNORECASE)),
      help_command
    )
  )

  updater.start_polling()
  updater.idle()
```

这个程序看起来和前面的例子非常相似，只有一个主要区别是下面：

```python
updater.dispatcher.add_handler(
  MessageHandler(
    Filters.text & Filters.regex(pattern=re.compile('help', re.IGNORECASE)),
    help_command
  )
)

```

`MessageHandler`用来获得用户的输入，你可以认为它是`CommandHandler`的类似功能，它可以处理任何满足指定过滤器的消息，在我们的六种，我们想在用户输入信息中包含help的时候将帮助信息返回给用户。

因此，我们有了一个完全程序化的新闻机器人。

![bot-demo](https://datanews.io/static/assets/img/screenshots/bot-demo.gif)



> 此文章翻译自https://datanews.io/blog/building-telegram-news-bot-2020

关注公众号获得更多最新的精彩内容。

![二维码](/Users/michael/Desktop/二维码.jpg)