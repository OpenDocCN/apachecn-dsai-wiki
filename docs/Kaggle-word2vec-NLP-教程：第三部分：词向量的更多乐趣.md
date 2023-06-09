<!--yml
category: Kaggle
date: 2022-07-01 00:00:00
-->

# Kaggle word2vec NLP 教程：第三部分：词向量的更多乐趣

### 代码

第三部分的代码在[这里](https://github.com/wendykan/DeepLearningMovies/blob/master/Word2Vec_BagOfCentroids.py)。

### 单词的数值表示

现在我们有了训练好的模型，对单词有一些语义理解，我们应该如何使用它？ 如果你看它的背后，第 2 部分训练的 Word2Vec 模型由词汇表中每个单词的特征向量组成，存储在一个名为`syn0`的[`numpy`](http://www.numpy.org/)数组中：

```py
>>> # Load the model that we created in Part 2
>>> from gensim.models import Word2Vec
>>> model = Word2Vec.load("300features_40minwords_10context")
2014-08-03 14:50:15,126 : INFO : loading Word2Vec object from 300features_40min_word_count_10context
2014-08-03 14:50:15,777 : INFO : setting ignored attribute syn0norm to None

>>> type(model.syn0)
<type 'numpy.ndarray'>

>>> model.syn0.shape
(16492, 300)
```

`syn0`中的行数是模型词汇表中的单词数，列数对应于我们在第 2 部分中设置的特征向量的大小。将最小单词计数设置为 40 ，总词汇量为 16,492 个单词，每个词有 300 个特征。 可以通过以下方式访问单个单词向量：

```py
>>> model["flower"]
```

...返回一个 1x300 的`numpy`数组。

### 从单词到段落，尝试 1：向量平均

IMDB 数据集的一个挑战是可变长度评论。 我们需要找到一种方法来获取单个单词向量并将它们转换为每个评论的长度相同的特征集。

由于每个单词都是 300 维空间中的向量，我们可以使用向量运算来组合每个评论中的单词。 我们尝试的一种方法是简单地平均给定的评论中的单词向量（为此，我们删除了停止词，这只会增加噪音）。

以下代码基于第 2 部分的代码构建了特征向量的平均值。

```py
import numpy as np  # Make sure that numpy is imported

def makeFeatureVec(words, model, num_features):
    # 用于平均给定段落中的所有单词向量的函数
    #
    # 预初始化一个空的 numpy 数组（为了速度）
    featureVec = np.zeros((num_features,),dtype="float32")
    #
    nwords = 0.
    # 
    # Index2word 是一个列表，包含模型词汇表中的单词名称。
    # 为了获得速度，将其转换为集合。 
    index2word_set = set(model.index2word)
    #
    # 遍历评论中的每个单词，如果它在模型的词汇表中，
    # 则将其特征向量加到 total
    for word in words:
        if word in index2word_set: 
            nwords = nwords + 1.
            featureVec = np.add(featureVec,model[word])
    # 
    # 将结果除以单词数来获得平均值
    featureVec = np.divide(featureVec,nwords)
    return featureVec


def getAvgFeatureVecs(reviews, model, num_features):
    # 给定一组评论（每个评论都是单词列表），计算每个评论的平均特征向量并返回2D numpy数组
    # 
    # 初始化计数器
    counter = 0.
    # 
    # 为了速度，预分配 2D numpy 数组
    reviewFeatureVecs = np.zeros((len(reviews),num_features),dtype="float32")
    # 
    # 遍历评论
    for review in reviews:
       #
       # 每 1000 个评论打印一次状态消息
       if counter%1000. == 0.:
           print "Review %d of %d" % (counter, len(reviews))
       # 
       # 调用生成平均特征向量的函数（定义如上）
       reviewFeatureVecs[counter] = makeFeatureVec(review, model, \
           num_features)
       #
       # 增加计数器
       counter = counter + 1.
    return reviewFeatureVecs
```

现在，我们可以调用这些函数来为每个段落创建平均向量。 以下操作将需要几分钟：

```py
# ****************************************************************
# 使用我们在上面定义的函数，
# 计算训练和测试集的平均特征向量。
# 请注意，我们现在删除停止词。

clean_train_reviews = []
for review in train["review"]:
    clean_train_reviews.append( review_to_wordlist( review, \
        remove_stopwords=True ))

trainDataVecs = getAvgFeatureVecs( clean_train_reviews, model, num_features )

print "Creating average feature vecs for test reviews"
clean_test_reviews = []
for review in test["review"]:
    clean_test_reviews.append( review_to_wordlist( review, \
        remove_stopwords=True ))

testDataVecs = getAvgFeatureVecs( clean_test_reviews, model, num_features )
```

接下来，使用平均段落向量来训练随机森林。 请注意，与第 1 部分一样，我们只能使用标记的训练评论来训练模型。

```py
# 使用 100 棵树让随机森林拟合训练数据
from sklearn.ensemble import RandomForestClassifier
forest = RandomForestClassifier( n_estimators = 100 )

print "Fitting a random forest to labeled training data..."
forest = forest.fit( trainDataVecs, train["sentiment"] )

# 测试和提取结果
result = forest.predict( testDataVecs )

# 写出测试结果
output = pd.DataFrame( data={"id":test["id"], "sentiment":result} )
output.to_csv( "Word2Vec_AverageVectors.csv", index=False, quoting=3 )
```

我们发现这产生了比偶然更好的结果，但是表现比词袋低了几个百分点。

由于向量的元素平均值没有产生惊人的结果，或许我们可以以更聪明的方式实现？ 加权单词向量的标准方法是应用[“tf-idf”](http://en.wikipedia.org/wiki/Tf%E2%80%93idf)权重，它衡量给定单词在给定文档集中的重要程度。 在 Python 中提取 tf-idf 权重的一种方法，是使用 scikit-learn 的[`TfidfVectorizer`](http://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.TfidfVectorizer.html)，它具有类似于我们在第 1 部分中使用的`CountVectorizer`的接口。但是，当我们尝试以这种方式加权我们的单词向量时，我们发现没有实质的性能改善。

### 从单词到段落，尝试 2：聚类

Word2Vec 创建语义相关单词的簇，因此另一种可能的方法是利用簇中单词的相似性。 以这种方式来分组向量称为“向量量化”。 为了实现它，我们首先需要找到单词簇的中心，我们可以通过使用[聚类算法](http://scikit-learn.org/stable/modules/clustering.html)（如 [K-Means](http://en.wikipedia.org/wiki/K-means_clustering)）来完成。

在 K-Means 中，我们需要设置的一个参数是“K”，或者是簇的数量。 我们应该如何决定要创建多少个簇？ 试错法表明，每个簇平均只有5个单词左右的小簇，比具有多个词的大簇产生更好的结果。 聚类代码如下。 我们使用 [scikit-learn 来执行我们的 K-Means](http://scikit-learn.org/stable/modules/generated/sklearn.cluster.KMeans.html)。

具有较大 K 的 K-Means 聚类可能非常慢；以下代码在我的计算机上花了 40 多分钟。 下面，我们给 K-Means 函数设置一个计时器，看看它需要多长时间。

```py
from sklearn.cluster import KMeans
import time

start = time.time() # Start time

# 将“k”（num_clusters）设置为词汇量大小的 1/5，或每个簇平均 5 个单词
word_vectors = model.syn0
num_clusters = word_vectors.shape[0] / 5

# 初始化 k-means 对象并使用它来提取质心
kmeans_clustering = KMeans( n_clusters = num_clusters )
idx = kmeans_clustering.fit_predict( word_vectors )

# 获取结束时间并打印该过程所需的时间
end = time.time()
elapsed = end - start
print "Time taken for K Means clustering: ", elapsed, "seconds."
```

现在，每个单词的聚类分布都存储在`idx`中，而原始 Word2Vec 模型中的词汇表仍存储在`model.index2word`中。 为方便起见，我们将它们压缩成一个字典，如下所示：

```py
# 创建单词/下标字典，将每个词汇表单词映射为簇编号
word_centroid_map = dict(zip( model.index2word, idx ))
```

这有点抽象，所以让我们仔细看看我们的簇包含什么。 你的簇可能会有所不同，因为 Word2Vec 依赖于随机数种子。 这是一个循环，打印出簇 0 到 9 的单词：

```py
# 对于前 10 个簇
for cluster in xrange(0,10):
    #
    # 打印簇编号
    print "\nCluster %d" % cluster
    #
    # 找到该簇编号的所有单词，然后将其打印出来
    words = []
    for i in xrange(0,len(word_centroid_map.values())):
        if( word_centroid_map.values()[i] == cluster ):
            words.append(word_centroid_map.keys()[i])
    print words
```

结果很有意思：

```py
Cluster 0
[u'passport', u'penthouse', u'suite', u'seattle', u'apple']

Cluster 1
[u'unnoticed']

Cluster 2
[u'midst', u'forming', u'forefront', u'feud', u'bonds', u'merge', u'collide', u'dispute', u'rivalry', u'hostile', u'torn', u'advancing', u'aftermath', u'clans', u'ongoing', u'paths', u'opposing', u'sexes', u'factions', u'journeys']

Cluster 3
[u'lori', u'denholm', u'sheffer', u'howell', u'elton', u'gladys', u'menjou', u'caroline', u'polly', u'isabella', u'rossi', u'nora', u'bailey', u'mackenzie', u'bobbie', u'kathleen', u'bianca', u'jacqueline', u'reid', u'joyce', u'bennett', u'fay', u'alexis', u'jayne', u'roland', u'davenport', u'linden', u'trevor', u'seymour', u'craig', u'windsor', u'fletcher', u'barrie', u'deborah', u'hayward', u'samantha', u'debra', u'frances', u'hildy', u'rhonda', u'archer', u'lesley', u'dolores', u'elsie', u'harper', u'carlson', u'ella', u'preston', u'allison', u'sutton', u'yvonne', u'jo', u'bellamy', u'conte', u'stella', u'edmund', u'cuthbert', u'maude', u'ellen', u'hilary', u'phyllis', u'wray', u'darren', u'morton', u'withers', u'bain', u'keller', u'martha', u'henderson', u'madeline', u'kay', u'lacey', u'topper', u'wilding', u'jessie', u'theresa', u'auteuil', u'dane', u'jeanne', u'kathryn', u'bentley', u'valerie', u'suzanne', u'abigail']

Cluster 4
[u'fest', u'flick']

Cluster 5
[u'lobster', u'deer']

Cluster 6
[u'humorless', u'dopey', u'limp']

Cluster 7
[u'enlightening', u'truthful']

Cluster 8
[u'dominates', u'showcases', u'electrifying', u'powerhouse', u'standout', u'versatility', u'astounding']

Cluster 9
[u'succumbs', u'comatose', u'humiliating', u'temper', u'looses', u'leans']
```

我们可以看到这些簇的质量各不相同。 有些是有道理的 - 簇 3 主要包含名称，而簇 6- 8包含相关的形容词（簇 6 是我最喜欢的）。 另一方面，簇 5 有点神秘：龙虾和鹿有什么共同之处（除了是两只动物）？ 簇 0 更糟糕：阁楼和套房似乎属于一个东西，但它们似乎不属于苹果和护照。 簇 2 包含......可能与战争有关的词？ 也许我们的算法在形容词上效果最好。

无论如何，现在我们为每个单词分配了一个簇（或“质心”），我们可以定义一个函数将评论转换为质心袋。 这就像词袋一样，但使用语义相关的簇而不是单个单词：

```py
def create_bag_of_centroids( wordlist, word_centroid_map ):
    #
    # 簇的数量等于单词/质心映射中的最大的簇索引
    num_centroids = max( word_centroid_map.values() ) + 1
    #
    # 预分配质心向量袋（为了速度）
    bag_of_centroids = np.zeros( num_centroids, dtype="float32" )
    #
    # 遍历评论中的单词。如果单词在词汇表中，
    # 找到它所属的簇，并将该簇的计数增加 1
    for word in wordlist:
        if word in word_centroid_map:
            index = word_centroid_map[word]
            bag_of_centroids[index] += 1
    #
    # 返回“质心袋”
    return bag_of_centroids
```

上面的函数将为每个评论提供一个`numpy`数组，每个数组的特征都与簇数相等。 最后，我们为训练和测试集创建了质心袋，然后训练随机森林并提取结果：

```py
# 为训练集质心预分配一个数组（为了速度）
train_centroids = np.zeros( (train["review"].size, num_clusters), \
    dtype="float32" )

# 将训练集评论转换为质心袋
counter = 0
for review in clean_train_reviews:
    train_centroids[counter] = create_bag_of_centroids( review, \
        word_centroid_map )
    counter += 1

# 对测试评论重复
test_centroids = np.zeros(( test["review"].size, num_clusters), \
    dtype="float32" )

counter = 0
for review in clean_test_reviews:
    test_centroids[counter] = create_bag_of_centroids( review, \
        word_centroid_map )
    counter += 1
```

```py
# 拟合随机森林并提取预测
forest = RandomForestClassifier(n_estimators = 100)

# 拟合可能需要几分钟
print "Fitting a random forest to labeled training data..."
forest = forest.fit(train_centroids,train["sentiment"])
result = forest.predict(test_centroids)

# 写出测试结果
output = pd.DataFrame(data={"id":test["id"], "sentiment":result})
output.to_csv( "BagOfCentroids.csv", index=False, quoting=3 )
```

我们发现与第 1 部分中的词袋相比，上面的代码给出了相同（或略差）的结果。

### 深度和非深度学习方法的比较

你可能会问：为什么词袋更好？

最大的原因是，在我们的教程中，平均向量和使用质心会失去单词的顺序，这使得它与词袋的概念非常相似。性能相似（在标准误差范围内）的事实使得所有三种方法实际上相同。

一些要尝试的事情：

首先，在更多文本上训练 Word2Vec 应该会大大提高性能。谷歌的结果基于从超过十亿字的语料库中学到的单词向量；我们标记和未标记的训练集合在一起只有 1800 万字左右。方便的是，Word2Vec 提供了加载由谷歌原始 C 工具输出的任何预训练模型的函数，因此也可以用 C 训练模型然后将其导入 Python。

其次，在已发表的文献中，分布式单词向量技术已被证明优于词袋模型。在本文中，在 IMDB 数据集上使用了一种名为段落向量的算法，来生成迄今为止最先进的一些结果。在某种程度上，它比我们在这里尝试的方法更好，因为向量平均和聚类会丢失单词顺序，而段落向量会保留单词顺序信息。
