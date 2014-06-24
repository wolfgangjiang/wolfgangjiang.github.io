---
layout: post
title: Clojure中的update-in
---
前两天是周末，我学了Clojure语言。它真是好用啊。以下是我写的一个例子，展示了Clojure对Scala的一个长处。

Clojure是Lisp的方言，有Lisp的各种好处，但与Common Lisp相比，增添了很多现代脚本语言风格的操作，使得好些过去在Ruby/Python里才能做的事，现在在Lisp里可以做了。而且，它天生是主推immutable的，在这方面花了大量的心思。

和Scala相比，例如hashmap这个常用类，如果是immutable的，那么在进行嵌套map的内容更新时就比较吃力了。例如对于ruby中这样的数据结构：

{% highlight ruby %}
weibo[:tweets][9507][:kind] = :retweet
weibo[:tweets][9507][:parent_id] = 9504
weibo[:tweets][9504][:retweeted_count] += 1
{% endhighlight %}

上面是mutable的hashmap，这样写起来没有什么不自然。但是有时我们需要保持历史上每一次的全局状态，就需要immutable的表示。如果是immutable的hashmap，则每一次更新都是返回一个全新的map，与旧的父map无关，我们必须显式地更新旧的父map。但是父map也是immutable的，所以它也是返回一个全新的map……光是非形式化的描述就这样纠结难懂了，写成Scala代码势必要变成这样：

{% highlight scala %}

val tweets = weibo("tweets")
val tweet_9507 = tweets(9507)
val new_tweet_9507 = tweet_9507.updated("kind" -> "retweet").
                                updated("parent_id" -> 9504)
val new_tweets = tweets.updated(9507 -> new_tweets_9507)
val new_weibo = weibo.updated("tweets" -> new_tweets)

{% endhighlight %}

合起来写就更可怕：

{% highlight scala %}

val new_weibo = weibo.updated("tweets" -> 
  tweets.updated(9507 -> 
    weibo("tweets")(9507).updated("kind" -> "retweet").
                          updated("parent_id" -> 9504)))

{% endhighlight %}

而Clojure就有专门做这个的库函数：`get-in`、`assoc-in`、`update-in`。上面的需求，写出来就是这样的：


{% highlight clojure %}
(assoc-in weibo [:tweets 9507 :kind] :retweet)
(assoc-in weibo [:tweets 9507 :parent-id] 9504)
(update-in weibo [:tweets 9504 :retweeted-count] inc)
{% endhighlight %}

每一个操作都是返回全新的修改过的map，保留旧有的immutable map。我们看到，所用的括号数比Scala版本还少得多呢。甚至我们可以用Clojure内置的高阶函数`->>`把它们连接起来：

{% highlight clojure %}
(->> weibo
     (assoc-in [:tweets 9507 :kind] :retweet)
     (assoc-in [:tweets 9507 :parent-id] 9504)
     (update-in [:tweets 9504 :retweeted-count] inc))
{% endhighlight %}

可见Clojure对于immutable的环境下的程序设计想得更加充分。

在Clojure的核心库函数代码中，可以看到`assoc-in`的写法：

{% highlight clojure %}

(defn assoc-in [obj keys value]
  (let [key (first keys)
        more-keys (rest keys)]
    (if (empty? more-keys)
      (assoc obj key value))
      (assoc obj key (assoc-in (get obj key) more-keys value))))

{% endhighlight %}

就这么简单，其中`[keys & more-keys]`相当于对列表拆分head/tail， `(assoc obj key value)`相当于`obj.updated(key -> value)`，而`(get obj key)`自然就是相当于`obj(key)`。

在Scala中，因为静态类型的缘故，要想对不同类型的key做到这个很难，除非每一次调用`assoc-in`的时候都指定整个参数序列的每一个的类型。假设最简单的情形，所有的keys都是字符串，则可以在Scala中模仿写成这样（省略使用implicit方式来做monkey patching的繁琐过程）：

{% highlight scala %}

def assoc_in[T](keys : List[String], value : T) : Map[String, T] = {
  val key = keys.head
  val more_keys = keys.tail
  if(more_keys.isEmpty) 
    this.updated(key -> value)
  else
    this.updated(key -> this(key).assoc_in(more_keys, value))
}

{% endhighlight %}

这些在Clojure里是不需要我们自己写的，它们都是标准库的一部分。读这样的标准库，令我们能够近距离地感受到牛人是怎样思考的。
