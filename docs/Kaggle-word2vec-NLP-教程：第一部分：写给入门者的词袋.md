<!--yml
category: Kaggle
date: 2022-07-01 00:00:00
-->

# Kaggle word2vec NLP 教程：第一部分：写给入门者的词袋

### 什么是 NLP

NLP（自然语言处理）是一组用于处理文本问题的技术。这个页面将帮助你从加载和清理IMDB电影评论来起步，然后应用一个简单的[词袋](http://en.wikipedia.org/wiki/Bag-of-words_model)模型，来获得令人惊讶的准确预测，评论是点赞还是点踩。

### 在你开始之前

本教程使用 Python。如果你之前没有使用过 Python，我们建议你前往[泰坦尼克号竞赛 Python 教程](http://www.kaggle.com/c/titanic-gettingStarted)，熟悉一下（查看随机森林介绍）。

如果你已熟悉 Python 并使用基本的 NLP 技术，则可能需要跳到第 2 部分。

本教程的这一部分不依赖于平台。在本教程中，我们将使用各种 Python 模块进行文本处理，深度学习，随机森林和其他应用。详细信息请参阅“配置你的系统”页面。

有很多很好的教程，以及实际上用 Python 写的关于 NLP 和文本处理的[整本书](http://www.nltk.org/book/)。本教程绝不是详尽无遗的 - 只是为了帮助你以电影评论起步。

### 代码

第 1 部分的教程代码就在[这里](https://github.com/wendykan/DeepLearningMovies/blob/master/BagOfWords.py)。

### 读取数据

可以从“数据”页面下载必要的文件。你需要的第一个文件是`unlabeledTrainData`，其中包含 25,000 个 IMDB 电影评论，每个评论都带有正面或负面情感标签。

接下来，将制表符分隔文件读入 Python。为此，我们可以使用泰坦尼克号教程中介绍的`pandas`包，它提供了`read_csv`函数，用于轻松读取和写入数据文件。如果你之前没有使用过`pandas`，则可能需要安装它。

```py
# 导入 pandas 包，然后使用 "read_csv" 函数读取标记的训练数据
import pandas as pd       
train = pd.read_csv("labeledTrainData.tsv", header=0, \
                    delimiter="\t", quoting=3)
```

这里，`header=0`表示文件的第一行包含列名，`delimiter=\t`表示字段由制表符分隔，`quoting=3`让 Python 忽略双引号，否则试图读取文件时，可能会遇到错误。

我们可以确保读取 25,000 行和 3 列，如下所示：

```py
>>> train.shape
(25000, 3)

>>> train.columns.values
array([id, sentiment, review], dtype=object)
```

这三列被称为`"id"`，`"sentiment"`和`"array"`。 现在你已经读取了培训集，请查看几条评论：

```py
print train["review"][0]
```

提醒一下，这将显示名为`"review"`的列中的第一个电影评论。 你应该看到一个像这样开头的评论：

```py
"With all this stuff going down at the moment with MJ i've started listening to his music, watching the odd documentary here and there, watched The Wiz and watched Moonwalker again. Maybe i just want to get a certain insight into this guy who i thought was really cool in the eighties just to maybe make up my mind whether he is guilty or innocent. Moonwalker is part biography, part feature film which i remember going to see at the cinema when it was originally released. Some of it has subtle messages about MJ's feeling towards the press and also the obvious message of drugs are bad m'kay. <br/><br/>..."
```

有 HTML 标签，如`"<br/>"`，缩写，标点符号 - 处理在线文本时的所有常见问题。 花一些时间来查看训练集中的其他评论 - 下一节将讨论如何为机器学习整理文本。

### 数据清理和文本预处理

删除 HTML 标记：`BeautifulSoup`包

首先，我们将删除 HTML 标记。 为此，我们将使用[`BeautifulSoup`库](http://www.crummy.com/software/BeautifulSoup/bs4/doc/)。 如果你没有安装，请从命令行（不是从 Python 内部）执行以下操作：

```
$ sudo pip install BeautifulSoup4
```

然后，从 Python 中加载包并使用它从评论中提取文本：

```py
# Import BeautifulSoup into your workspace
from bs4 import BeautifulSoup             

# Initialize the BeautifulSoup object on a single movie review     
example1 = BeautifulSoup(train["review"][0])  

# Print the raw review and then the output of get_text(), for 
# comparison
print train["review"][0]
print example1.get_text()
```

调用`get_text()`会为你提供不带标签的评论文本。如果你浏览`BeautifulSoup`文档，你会发现它是一个非常强大的库 - 比我们对此数据集所需的功能更强大。但是，使用正则表达式删除标记并不是一种可靠的做法，因此即使对于像这样简单的应用程序，通常最好使用像`BeautifulSoup`这样的包。

处理标点符号，数字和停止词：NLTK 和正则表达式

在考虑如何清理文本时，我们应该考虑我们试图解决的数据问题。对于许多问题，删除标点符号是有意义的。另一方面，在这种情况下，我们正在解决情感分析问题，并且有可能`"!!!"`或者`":-("`可以带有情感，应该被视为单词。在本教程中，为简单起见，我们完全删除了标点符号，但这是你可以自己玩的东西。

与之相似，在本教程中我们将删除数字，但还有其他方法可以处理它们，这些方法同样有意义。例如，我们可以将它们视为单词，或者使用占位符字符串（例如`"NUM"`）替换它们。

要删除标点符号和数字，我们将使用一个包来处理正则表达式，称为`re`。Python 内置了该软件包；无需安装任何东西。对于正则表达式如何工作的详细说明，请参阅[包文档](https://docs.python.org/2/library/re.html#)。现在，尝试以下方法：

```py
import re
# 使用正则表达式执行查找和替换
letters_only = re.sub("[^a-zA-Z]",           # 要查找的模式串
                      " ",                   # 要替换成的模式串
                      example1.get_text() )  # 要从中查找的字符串
print letters_only
```

正则表达式的完整概述超出了本教程的范围，但是现在知道`[]`表示分组成员而`^`表示“不”就足够了。 换句话说，上面的`re.sub()`语句说：“查找任何不是小写字母（`a-z`）或大写字母（`A-Z`）的内容，并用空格替换它。”

我们还将我们的评论转换为小写并将它们分成单个单词（在 NLP 术语中称为“[分词](http://en.wikipedia.org/wiki/Tokenization)”）：

```py
lower_case = letters_only.lower()        # 转换为小写
words = lower_case.split()               # 分割为单词
```

最后，我们需要决定如何处理那些没有多大意义的经常出现的单词。 这样的词被称为“停止词”；在英语中，它们包括诸如“a”，“and”，“is”和“the”之类的单词。方便的是，Python 包中内置了停止词列表。让我们从 Python 自然语言工具包（NLTK）导入停止词列表。 如果你的计算机上还没有该库，则需要安装该库；你还需要安装附带的数据包，如下所示：

```py
import nltk
nltk.download()  # 下载文本数据集，包含停止词
```

现在我们可以使用`nltk`来获取停止词列表：

```py
from nltk.corpus import stopwords # 导入停止词列表
print stopwords.words("english") 
```

这将允许你查看英语停止词列表。 要从我们的电影评论中删除停止词，请执行：

```py
# 从 "words" 中移除停止词
words = [w for w in words if not w in stopwords.words("english")]
print words
```

这会查看`words`列表中的每个单词，并丢弃在停止词列表中找到的任何内容。 完成所有这些步骤后，你的评论现在应该是这样的：

```py
[u'stuff', u'going', u'moment', u'mj', u've', u'started', u'listening', u'music', u'watching', u'odd', u'documentary', u'watched', u'wiz', u'watched', u'moonwalker', u'maybe', u'want', u'get', u'certain', u'insight', u'guy', u'thought', u'really', u'cool', u'eighties', u'maybe', u'make', u'mind', u'whether', u'guilty', u'innocent', u'moonwalker', u'part', u'biography', u'part', u'feature', u'film', u'remember', u'going', u'see', u'cinema', u'originally', u'released', u'subtle', u'messages', u'mj', u'feeling', u'towards', u'press', u'also', u'obvious', u'message', u'drugs', u'bad', u'm', u'kay',.....]
```

不要担心在每个单词之前的`u`；它只是表明 Python 在内部将每个单词表示为 [unicode 字符串](https://docs.python.org/2/howto/unicode.html#python-2-x-s-unicode-support)。

我们可以对数据做很多其他的事情 - 例如，Porter Stemming（词干提取）和 Lemmatizing（词形还原）（都在 NLTK 中提供）将允许我们将`"messages"`，`"message"`和`"messaging"`视为同一个词，这当然可能很有用。 但是，为简单起见，本教程将就此打住。

### 把它们放在一起

现在我们有了清理评论的代码 - 但我们需要清理 25,000 个训练评论！ 为了使我们的代码可重用，让我们创建一个可以多次调用的函数：

```py
def review_to_words( raw_review ):
    # 将原始评论转换为单词字符串的函数
    # 输入是单个字符串（原始电影评论），
    # 输出是单个字符串（预处理过的电影评论）
    # 1. 移除 HTML
    review_text = BeautifulSoup(raw_review).get_text() 
    #
    # 2. 移除非字母        
    letters_only = re.sub("[^a-zA-Z]", " ", review_text) 
    #
    # 3. 转换为小写，分成单个单词
    words = letters_only.lower().split()                             
    #
    # 4. 在Python中，搜索集合比搜索列表快得多，
    #    所以将停止词转换为一个集合
    stops = set(stopwords.words("english"))                  
    # 
    # 5. 删除停止词
    meaningful_words = [w for w in words if not w in stops]   
    #
    # 6. 将单词连接成由空格分隔的字符串，
    #    并返回结果。
    return( " ".join( meaningful_words ))   
```

这里有两个新元素：首先，我们将停止词列表转换为不同的数据类型，即集合。 这是为了速度；因为我们将调用这个函数数万次，所以它需要很快，而 Python 中的搜索集合比搜索列表要快得多。

其次，我们将这些单词合并为一段。 这是为了使输出更容易在我们的词袋中使用，在下面。 定义上述函数后，如果你为单个评论调用该函数：

```py
clean_review = review_to_words( train["review"][0] )
print clean_review
```

它应该为你提供与前面教程部分中所做的所有单独步骤完全相同的输出。 现在让我们遍历并立即清理所有训练集（这可能需要几分钟，具体取决于你的计算机）：

```py
# 根据 dataframe 列大小获取评论数
num_reviews = train["review"].size

# 初始化空列表来保存清理后的评论
clean_train_reviews = []

# 遍历每个评论；创建索引 i
# 范围是 0 到电影评论列表长度
for i in xrange( 0, num_reviews ):
    # 为每个评论调用我们的函数，
    # 并将结果添加到清理后评论列表中
    clean_train_reviews.append( review_to_words( train["review"][i] ) )
```

有时等待冗长的代码的运行会很烦人。 编写提供状态更新的代码会很有帮助。 要让 Python 在其处理每 1000 个评论后打印状态更新，请尝试在上面的代码中添加一两行：

```py
print "Cleaning and parsing the training set movie reviews...\n"
clean_train_reviews = []
for i in xrange( 0, num_reviews ):
    # 如果索引被 1000 整除，打印消息
    if( (i+1)%1000 == 0 ):
        print "Review %d of %d\n" % ( i+1, num_reviews )                                                                    
    clean_train_reviews.append( review_to_words( train["review"][i] ))
```

### 从词袋创建特征（使用`sklearn`）

现在我们已经整理了我们的训练评论，我们如何将它们转换为机器学习的某种数字表示？一种常见的方法叫做词袋。词袋模型从所有文档中学习词汇表，然后通过计算每个单词出现的次数对每个文档进行建模。例如，考虑以下两句话：

句子1：`"The cat sat on the hat"`

句子2：`"The dog ate the cat and the hat"`

从这两个句子中，我们的词汇如下：

`{ the, cat, sat, on, hat, dog, ate, and }`

为了得到我们的词袋，我们计算每个单词出现在每个句子中的次数。在句子 1 中，“the”出现两次，“cat”，“sat”，“on”和“hat”每次出现一次，因此句子 1 的特征向量是：

`{ the, cat, sat, on, hat, dog, ate, and }`

句子 1：`{ 2, 1, 1, 1, 1, 0, 0, 0 }`

同样，句子 2 的特征是：`{ 3, 1, 0, 0, 1, 1, 1, 1}`

在 IMDB 数据中，我们有大量的评论，这将为我们提供大量的词汇。要限制特征向量的大小，我们应该选择最大词汇量。下面，我们使用 5000 个最常用的单词（记住已经删除了停止词）。

我们将使用 scikit-learn 中的`feature_extraction`模块来创建词袋特征。如果你学习了泰坦尼克号竞赛中的随机森林教程，那么你应该已经安装了 scikit-learn；否则你需要[安装它](http://scikit-learn.org/stable/install.html)。

```py
print "Creating the bag of words...\n"
from sklearn.feature_extraction.text import CountVectorizer

# 初始化 "CountVectorizer" 对象，
# 这是 scikit-learn 的一个词袋工具。
vectorizer = CountVectorizer(analyzer = "word",   \
                             tokenizer = None,    \
                             preprocessor = None, \
                             stop_words = None,   \
                             max_features = 5000) 

# fit_transform() 有两个功能：
# 首先，它拟合模型并学习词汇；
# 第二，它将我们的训练数据转换为特征向量。
# fit_transform 的输入应该是字符串列表。
train_data_features = vectorizer.fit_transform(clean_train_reviews)

# Numpy 数组很容易使用，因此将结果转换为数组
train_data_features = train_data_features.toarray()
```

要查看训练数据数组现在的样子，请执行以下操作：

```py
>>> print train_data_features.shape
(25000, 5000)
```

它有 25,000 行和 5,000 个特征（每个词汇一个）。

请注意，`CountVectorizer`有自己的选项来自动执行预处理，标记化和停止词删除 - 对于其中的每一个，我们不指定`None`，可以使用内置方法或指定我们自己的函数来使用。 详细信息请参阅[函数文档](http://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.CountVectorizer.html)。 但是，我们想在本教程中编写我们自己的数据清理函数，来向你展示如何逐步完成它。

现在词袋模型已经训练好了，让我们来看看词汇表：

```py
# 看看词汇表中的单词
vocab = vectorizer.get_feature_names()
print vocab
```

如果你有兴趣，还可以打印词汇表中每个单词的计数：

```py
import numpy as np

# 求和词汇表中每个单词的计数
dist = np.sum(train_data_features, axis=0)

# 对于每个词，打印它和它在训练集中的出现次数
for tag, count in zip(vocab, dist):
    print count, tag
```

### 随机森林

到了这里，我们有词袋的数字训练特征和每个特征向量的原始情感标签，所以让我们做一些监督学习！ 在这里，我们将使用我们在泰坦尼克号教程中介绍的随机森林分类器。 随机森林算法包含在 scikit-learn 中（[随机森林](http://scikit-learn.org/stable/modules/generated/sklearn.ensemble.RandomForestClassifier.html)使用许多基于树的分类器来进行预测，因此是“森林”）。 下面，我们将树的数量设置为 100 作为合理的默认值。 更多树可能（或可能不）表现更好，但肯定需要更长时间来运行。 同样，每个评论所包含的特征越多，所需的时间就越长。

```py
print "Training the random forest..."
from sklearn.ensemble import RandomForestClassifier

# 使用 100 棵树初始化随机森林分类器
forest = RandomForestClassifier(n_estimators = 100) 

# 使用词袋作为特征并将情感标签作为响应变量，使森林拟合训练集
# 这可能需要几分钟来运行
forest = forest.fit( train_data_features, train["sentiment"] )
```

### 创建提交

剩下的就是在我们的测试集上运行训练好的随机森林并创建一个提交文件。 如果你还没有这样做，请从“数据”页面下载`testData.tsv`。 此文件包含另外 25,000 条评论和标签；我们的任务是预测情感标签。

请注意，当我们使用词袋作为测试集时，我们只调用`transform`，而不是像训练集那样调用`fit_transform`。 在机器学习中，你不应该使用测试集来拟合你的模型，否则你将面临[过拟合](http://blog.kaggle.com/2012/07/06/the-dangers-of-overfitting-psychopathy-post-mortem/)的风险。 出于这个原因，我们将测试集保持在禁止状态，直到我们准备好进行预测。

```py
# 读取测试数据
test = pd.read_csv("testData.tsv", header=0, delimiter="\t", \
                   quoting=3 )

# 验证有 25,000 行和 2 列
print test.shape

# 创建一个空列表并逐个附加干净的评论
num_reviews = len(test["review"])
clean_test_reviews = [] 

print "Cleaning and parsing the test set movie reviews...\n"
for i in xrange(0,num_reviews):
    if( (i+1) % 1000 == 0 ):
        print "Review %d of %d\n" % (i+1, num_reviews)
    clean_review = review_to_words( test["review"][i] )
    clean_test_reviews.append( clean_review )

# 获取测试集的词袋，并转换为 numpy 数组
test_data_features = vectorizer.transform(clean_test_reviews)
test_data_features = test_data_features.toarray()

# 使用随机森林进行情感标签预测
result = forest.predict(test_data_features)

# 将结果复制到带有 "id" 列和 "sentiment" 列的 pandas dataframe
output = pd.DataFrame( data={"id":test["id"], "sentiment":result} )

# 使用 pandas 编写逗号分隔的输出文件
output.to_csv( "Bag_of_Words_model.csv", index=False, quoting=3 )
```

恭喜，你已准备好第一次提交！ 尝试不同的事情，看看你的结果如何变化。 你可以以不同方式清理评论，为词袋表示选择不同数量的词汇表单词，尝试 Porter Stemming，不同的分类器或任何其他的东西。 要在不同的数据集上试用你的 NLP 招式，你还可以参加我们的[烂番茄比赛](https://www.kaggle.com/c/sentiment-analysis-on-movie-reviews)。 或者，如果你为完全不同的东西做好了准备，请访问深度学习和词向量页面。
