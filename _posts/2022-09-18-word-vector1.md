---
title: 'Word Embedding (I): Count-based Word Vectors'
date: 2022-09-18
permalink: /posts/2022/09/Word-vector1/
tags:
  - NLP
---
If you are interested in NLP (natural language processing), you probably have heard of word embedding. In simple words, we use word embedding **to map words to real vectors**. It is the first step for computers to understand human language.

We could find a more formal definition from Wikipedia.
> In natural language processing (NLP), word embedding is a term used for the representation of words for text analysis, typically in the form of a real-valued vector that encodes the meaning of the word such that the words that are closer in the vector space are expected to be similar in meaning.

So our main question will be how to do it? There are generally two types of method, **count-based vectors and prediction-based vectors**. 

In this first article of "Word Embedding" series, we will cover count-based (or sometimes frequency-based) word vectors and their implementation in Python.
- [One-hot vectors](#one-hot-vectors)
- [TF-IDF (term frequency–inverse document frequency)](#tf-idf-term-frequencyinverse-document-frequency)
- [Co-occurrence matrix](#co-occurrence-matrix)

Let's start with the most straight-forward one.
## One-hot vectors
On-hot vectors are frequently used to represent categorical or discrete data. In NLP, it just takes one word as one category. 

From Wikipedia
>  A one-hot vector is a 1×N matrix (vector) used to distinguish each word in a vocabulary from every other word in the vocabulary. The vector consists of 0s in all cells with the exception of a single 1 in a cell used uniquely to identify the word. 

Let's break it down into simple words with an example. Suppose we have a small dictionary of only 5 words `['the','boy','is','playing','sleep']`. To encode them in one-hot vectors, we will create a 1×5 vector for each word. In each vector, we put 1 at the position of the target word, and 0 at all other positions. 

```
  the = [1 0 0 0 0]
  boy = [0 1 0 0 0]
  is = [0 0 1 0 0]
  playing = [0 0 0 1 0]
  ball = [0 0 0 0 1]
```

You could find that the word matrix is simply a N×N identity matrix, where N equals the number of unique words in the dictionary. You could easily generate one by `np.identity(N)`, so we won't spend more time on its code.

Despite its simplicity, there are two main problems with the one-hot vector.
- **It generates a sparse matrix with very large dimension**, which could be expensive in terms of storage and computation.
- **It does not encode similarity**, as all vectors are orthogonal. If we compute cosine similarity between two vectors, we will get 0 for all pairs of different words and 1 for all pairs of same words. For instance, you get $similarity(beer,champagne) = similarity(beer,sky) = 0$. It provides no information at all! 

To address the first one, we could add up the one-hot vectors for every word in a document to get its frequency vector. For instance, the vector for "the boy is playing" will be `[1 1 1 1 0]`, while the vector for "the boy is playing the ball" will be `[2 1 1 1 1]`. If we have multiple documents and stack their vectors, we can get a document-term matrix. [TF-IDF](#tf-idf-term-frequencyinverse-document-frequency) is a popular way to weight different terms.

To address the second one, we could make use of the context of a word, namely, the set of words that appear nearby within a fixed-size window. We will introduce this one in [Co-occurrence matrix](#co-occurance-matrix).

## TF-IDF (term frequency–inverse document frequency)
TF-IDF is compusted as the product of TF and IDF. Again we follow the definition from Wikipedia, for a given term *t* and document *d*
- **TF (term frequency)**: $\frac{f_{t,d}}{\sum_{t'\in d}f_{t',d}}$, the number of times *t* occurs over the total number of terms (duplicated occurrences are counted as well). It measures the relative frequency of *t* in document *d*.
- **IDF (inverse document frequency)**: $log \frac{\|D\|}{\|d\in D : t\in d\|}$, the logarithmically scaled inverse fraction of the documents containing *t*. It measures the commonality of *t* across documents. If the term is less common, IDF will be higher, the term provides more information.

Still sounds a bit abstract? Let's move to an example with its Python implementation.

Suppose we have the following corpus.

```python
  raw_corpus = ['The dog is chasing the ball.',
                'The boy is playing with the ball.',
                'The girl is crying.']
```

One of the simplest ways to compute TF-IDF matrix is to use `TfidfVectorizer` from `sklearn`. Let's first try the default option and check what we have.

```python
  from sklearn.feature_extraction.text import TfidfVectorizer

  tv = TfidfVectorizer()
  tv_fit = tv.fit_transform(raw_corpus)
  print(tv_fit.toarray())

  [[0.36580076 0.         0.48098405 0.        0.48098405 0.        0.28407693 0.         0.56815385 0.        ]
   [0.32965117 0.43345167 0.         0.        0.         0.        0.25600354 0.43345167 0.51200708 0.43345167]
   [0.         0.         0.         0.6088451 0.         0.6088451 0.35959372 0.         0.35959372 0.        ]]
```

We get a 3×10 matrix, where 3 is the number of documents, 10 is the number of unique words. Documents are ranked by your input order and words are ranked in alphabetical order. There are several differences between the default option of `TfidfVectorizer` and the above Wiki's definition we introduced.
- TF is computed as $f_{t,d}$ without normalization, namely, the absolute frequency or word count of term *t* in document *d*.
- IDF is computed as $log \frac{\|D\|+1}{\|d\in D : t\in d\|+1}+1$ to avoid division-by-zero or negative results. Traditionally, we add 1 to the denominator in case *t* is not in the corpus. But this will lead to negative IDF when some term is contained in all documents. So when `smooth_idf=True` (default), `sklearn` add 1 to both the nominator and denominator. If you set `smooth_idf=False`, IDF will be $log \frac{\|D\|}{\|d\in D : t\in d\|}+1$.
- `norm='l2'` by default. It means L2 norm is applied to TF-IDF matrix such that the sum of squares of vector elements is 1 for each row. You could customize it to L1 norm `norm='l1'` or no normalization `norm=None` (as in Wiki's definition).

To see it more clearly, let's try to build it up by ourselves. We define the following functions.
- `preprocessing`: input `raw_corpus` (list of strings) and `start_end_token` (bool, `True` if add start token and end token), remove punctuation, return `corpus` (list of list of strings).
- `distinct_words`: input `corpus` (list of list of strings), return a sorted list of unique words in the corpus `corpus_words` (list of strings).
- `compute_tf_idf`: input `corpus` (list of list of strings), `smooth_idf` (bool, default `True`) and `normalize` (string, default 'l2', possible values `{'l1','l2','None'}`), return TF-IDF matrix `tf_idf` (numpy matrix, \|D\|×N).

```python
  import re 
  import numpy as np
  from numpy.linalg import norm

  def preprocessing(raw_corpus, start_end_token=True):
    if start_end_token == True:
        return [[start_token] + re.sub(r'[^\w\s]', '', s).lower().split() + [end_token] for s in raw_corpus]
    else:
        return [re.sub(r'[^\w\s]', '', s).lower().split() for s in raw_corpus]
  
  def distinct_words(corpus):
    corpus_words = list(set([w for s in corpus for w in s]))
    corpus_words = sorted(corpus_words)
    return corpus_words

  def compute_tf_idf(corpus, smooth_idf=True, normalize='l2'):
      words = distinct_words(corpus)
      tf = np.zeros((len(corpus),len(words)))
      for i in range(len(corpus)):
          tf[i] = np.array([corpus[i].count(w) for w in words])
      if smooth_idf == True:
          idf = [np.log((len(corpus)+1)/(sum(w in d for d in corpus)+1))+1 for w in words]
      else: 
          idf = [np.log(len(corpus)/sum(w in d for d in corpus))+1 for w in words]
      tf_idf = tf*idf
      if normalize == 'l1':
          tf_idf = tf_idf/norm(tf_idf,1,axis=1)[:,None]
      elif normalize == 'l2':
          tf_idf = tf_idf/norm(tf_idf,axis=1)[:,None]
      return tf_idf
```

Let's check out their results. When `smooth_idf=True, normalize='l2'`, we should get the exactly same TF-IDF as `sklearn`.
```python
  corpus = preprocessing(raw_corpus, start_end_token=False)
  print(corpus)
  corpus_words = distinct_words(corpus)
  print(corpus_words)
  tf_idf = compute_tf_idf(corpus)
  print(tf_idf)
  print((tf_idf==tv_fit.toarray()).all())

  [['the', 'dog', 'is', 'chasing', 'the', 'ball'], ['the', 'boy', 'is', 'playing', 'with', 'the', 'ball'], ['the', 'girl', 'is', 'crying']]
  ['ball', 'boy', 'chasing', 'crying', 'dog', 'girl', 'is', 'playing', 'the', 'with']
  [[0.36580076 0.         0.48098405 0.        0.48098405 0.        0.28407693 0.         0.56815385 0.        ]
   [0.32965117 0.43345167 0.         0.        0.         0.        0.25600354 0.43345167 0.51200708 0.43345167]
   [0.         0.         0.         0.6088451 0.         0.6088451 0.35959372 0.         0.35959372 0.        ]]
  True
```
If we do not smooth IDF or normalize TF-IDF, we should get the same matrix as `TfidfVectorizer(smooth_idf = False, norm=None)`. We could check this easily.
```python
  tv = TfidfVectorizer(smooth_idf = False, norm=None)
  tv_fit = tv.fit_transform(raw_corpus)
  sklearn_tf_idf = tv_fit.toarray()
  our_tf_idf = compute_tf_idf(corpus, smooth_idf=False, normalize=None)
  print((sklearn_tf_idf==our_tf_idf).all())

  True
```

If you want to get TF-IDF according to Wiki's definition, you need to add `tf = tf/tf.sum(axis=1)[:,None]` to get relative frequency, and to change IDF as `idf = [np.log(len(corpus)/(sum(w in d for d in corpus)+1)) for w in words]`. Try it yourself!

## Co-occurrence matrix
The main idea of co-occurrence matrix is **distributional semantics** that a word's meaning is given by the words that frequently appear close-by. For a given word, it counts how often other words appears in the context window [-w,+w] surrounding the word, where w is the fixed window size. This gives us a N×N matrix, where N is the number of unique words in the corpus. The matrix is symmetric by definition. If word i is in the window of word j, then word j must be in the window of word i as well. So it does not really matter if you read it by row or columns. I personally like to take each row as word vector, namely, the value at (i,j) represents the number of times when word j appears in word i's window.

However, this N×N matrix suffers the same problem of sparsity as one-hot vector. Usually, dimensionality reduction is the next step, where SDC (Singular Value Decomposition) or PCA (Principal Components Analysis) are commonly used to decompose the matrix and to obtain key components.

In the rest of this section, we will compute co-occurrence matrix together. Let's start with the same raw corpus in TF-IDF and use a fixed window size of 1.
```python
  raw_corpus = ['The dog is chasing the ball.',
                'The boy is playing with the ball.',
                'The girl is crying.']

  corpus = preprocessing(raw_corpus, start_end_token=True)
  print(corpus)
  corpus_words = distinct_words(corpus)
  print(corpus_words)

  [['<START>', 'the', 'dog', 'is', 'chasing', 'the', 'ball', '<END>'], ['<START>', 'the', 'boy', 'is', 'playing', 'with', 'the', 'ball', '<END>'], ['<START>', 'the', 'girl', 'is', 'crying', '<END>']]
  ['<END>', '<START>', 'ball', 'boy', 'chasing', 'crying', 'dog', 'girl', 'is', 'playing', 'the', 'with']
```
It is more common to add start and end token to each sentence since position matters here.

Before diving into the code, we could check some words manually. 
- For `'<END>'`: `'ball'` occurs twice in its context window of size 1, `'crying'` occurs once, while other words never occurs. So the vector of `'<END>'` should be `[0 0 2 0 0 1 0 0 0 0 0 0]`.
- For `'ball'`: `'the'` and `'<END>'` occur twice in its window, , while other words never occurs. Its word vector should be `[2 0 0 0 0 0 0 0 0 0 2 0]`.
- For `'is'`:  `'boy', 'chasing', 'crying', 'dog', 'girl', 'playing'` each occurs once. Its vector should be `[0 0 0 1 1 1 1 1 0 1 0 0]`.
- For `'the'`: `'<START>'` occurs three times, `'ball'` occurs twice, `'boy', 'chasing', 'dog', 'girl', 'with'` each occurs once. Its word vector should be `[0 3 2 1 1 0 1 1 0 0 0 1]`.

Now it's time to teach your computer to do this. We define a new function here.
- `compute_co_occurrence_matrix`: input `corpus` (list of list of strings) and `window_size` (positive integer), return co-occurance matrix `M` (numpy matrix, N×N) and word-index dictionary `word2ind` (dict). We usually use the alphabetical order for `word2ind`, this dictionary helps you look up word vector in co-occurrence matrix easier.
- `reduce_to_k_dim`: input co-occurrence matrix `M` (numpy matrix, N×N) and desired dimension `k` (positive integer), return matrix of k-dimensioal word embeddings `M_reduced` (numpy matrix, N×k). We just borrow `TruncatedSVD` from `sklearn`, learn more about it [here](https://scikit-learn.org/stable/modules/generated/sklearn.decomposition.TruncatedSVD.html).

```python
  from sklearn.decomposition import TruncatedSVD

  def compute_co_occurrence_matrix(corpus, window_size):
    words = distinct_words(corpus)
    M = np.zeros((len(words),len(words)))
    word2ind = {}
    for i in range(len(words)):
        co_occur = []
        for s in corpus:
            if words[i] in s: 
                # get the index list of words[i] (multiple occurences are possible)
                for j in [index for index, x in enumerate(s) if x == words[i]]:
                    co_occur.extend(s[max(0,j-window_size):min(len(s)-1,j+window_size)+1])
        M[i] = np.array([co_occur.count(w) for w in words])
        word2ind[words[i]] = i
    np.fill_diagonal(M, 0)
    return M, word2ind

  def reduce_to_k_dim(M, k):
    svd = TruncatedSVD(n_components=k, n_iter=10)
    M_reduced = svd.fit_transform(M)
    return M_reduced
```

Let's check its outcomes with our manual calculation.
```python
  M, word2ind = compute_co_occurrence_matrix(corpus,1)
  M_reduced = reduce_to_k_dim(M, k=2)
  print(word2ind)
  print(M)
  print(M_reduced)
  
  {'<END>': 0, '<START>': 1, 'ball': 2, 'boy': 3, 'chasing': 4, 'crying': 5, 'dog': 6, 'girl': 7, 'is': 8, 'playing': 9, 'the': 10, 'with': 11}
  [[0. 0. 2. 0. 0. 1. 0. 0. 0. 0. 0. 0.]
   [0. 0. 0. 0. 0. 0. 0. 0. 0. 0. 3. 0.]
   [2. 0. 0. 0. 0. 0. 0. 0. 0. 0. 2. 0.]
   [0. 0. 0. 0. 0. 0. 0. 0. 1. 0. 1. 0.]
   [0. 0. 0. 0. 0. 0. 0. 0. 1. 0. 1. 0.]
   [1. 0. 0. 0. 0. 0. 0. 0. 1. 0. 0. 0.]
   [0. 0. 0. 0. 0. 0. 0. 0. 1. 0. 1. 0.]
   [0. 0. 0. 0. 0. 0. 0. 0. 1. 0. 1. 0.]
   [0. 0. 0. 1. 1. 1. 1. 1. 0. 1. 0. 0.]
   [0. 0. 0. 0. 0. 0. 0. 0. 1. 0. 0. 1.]
   [0. 3. 2. 1. 1. 0. 1. 1. 0. 0. 0. 1.]
   [0. 0. 0. 0. 0. 0. 0. 0. 0. 1. 1. 0.]]
  [[ 0.81973139  0.83034113]
   [ 1.95121418 -1.97001638]
   [ 1.66301653 -1.68238885]
   [ 0.8536293  -0.8438658 ]
   [ 0.8536293  -0.8438658 ]
   [ 0.38432811 -0.37171597]
   [ 0.8536293  -0.8438658 ]
   [ 0.8536293  -0.8438658 ]
   [ 0.91985811  0.84236217]
   [ 0.36472082 -0.04340859]
   [ 2.94393564  2.9549918 ]
   [ 0.73098262 -0.64702567]]
```
These word vectors are no longer orthogonal to each other. You could measure their similarity by cosine similarity. 
```python 
  def cos_sim(M, word2ind, word_i, word_j):
      a =  M_reduced[word2ind[word_i]]
      b =  M_reduced[word2ind[word_j]]
      return np.dot(a, b)/(norm(a)*norm(b))

  print('sim(boy,ball):',cos_sim(M_reduced, word2ind, 'boy', 'ball'))
  print('sim(boy,playing):',cos_sim(M_reduced, word2ind, 'boy', 'playing'))

  sim(boy,ball): 0.9999333883164274
  sim(boy,playing): 0.7892650840360069
```

From above results, we may conclude that `'boy'` is more similar to `'ball'` than to `'playing'`. The results will make more sense when you have larger corpus.

That's all we have for this post. If you find it helpful and want to learn more about word embedding, stay tuned to this series, see you next time!
<br>

**Reference**
- [Stanford CS224N \| Winter 2021](https://web.stanford.edu/class/archive/cs/cs224n/cs224n.1214/index.html#schedule), recordings can be found on [Youtube](https://www.youtube.com/playlist?list=PLoROMvodv4rOSH4v6133s9LFPRHjEmbmJ), highly recommended!
- [Wikipedia](https://en.wikipedia.org/wiki/Word_embedding), Word embedding.
- [Wikipedia](https://en.wikipedia.org/wiki/One-hot), One-hot.
- [Wikipedia](https://en.wikipedia.org/wiki/Bag-of-words_model), Bag-of-words model.
- [Wikipedia](https://en.wikipedia.org/wiki/Tf%E2%80%93idf), tf-idf.
- [sklearn](https://scikit-learn.org/stable/modules/generated/sklearn.feature_extraction.text.TfidfVectorizer.html), sklearn.feature_extraction.text.TfidfVectorizer.
- [sklearn](https://scikit-learn.org/stable/modules/generated/sklearn.decomposition.TruncatedSVD.html), sklearn.decomposition.TruncatedSVD.
- [Manjeet Singh](https://medium.com/data-science-group-iitr/word-embedding-2d05d270b285), Word embedding.