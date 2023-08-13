Part-of-Speech Tagging for Classical Greek 🏺
==============================
The goal of this project is to implement and train a [part-of-speech (POS) tagger](https://en.wikipedia.org/wiki/Part-of-speech_tagging) for Ancient Classical Greek, a [morphologically rich](https://arxiv.org/pdf/2005.01330.pdf) and [low-resource](https://arxiv.org/pdf/2006.07264.pdf) language, using texts from [The Ancient Greek and Latin Dependency Treebank](https://perseusdl.github.io/treebank_data/) and modeling techniques inspired by Jurafsky and Martin's [*Speech and Language Processing*](https://web.stanford.edu/~jurafsky/slp3/).

## Implementation details

A [hidden Markov model](https://en.wikipedia.org/wiki/Hidden_Markov_model) is implemented to estimate the transition and emission probabilities from the training data. The [Viterbi algorithm](https://en.wikipedia.org/wiki/Viterbi_algorithm) is used for decoding, i.e. finding the most likely sequence of hidden states (POS tags) for previously unseen observations (sentences).

The HMM is trained on bigram distributions (distributions of _pairs_ of adjacent tokens). The first pass over the training data generates a fixed list of vocabulary tokens. Any token occurring less than twice in the training data is assigned a [special unknown word token](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/data/unk_toks.txt) based on a few selected [morphological idiosyncrasies](https://wac.colostate.edu/books/sound/chapter5.pdf) of common English word classes (e.g. most tokens with the suffix "-ism" are nouns). The second pass uses the transformed training data to collect the bigram transition and emission counts and saves them to a model file.

To decode the development and test splits, the input sequence is first transformed according to the unknown word rules mentioned above. The transition and emission counts are then converted to proper probability distributions, using [additive smoothing](https://en.wikipedia.org/wiki/Additive_smoothing) to estimate probabilities for transitions/emissions that have not been observed in the training data. A pseudo count `alpha > 0` is used as the smoothing parameter, with `alpha = 0.001` giving best results on the development split (see results below).

For training and decoding, the input sequences are treated as a continuous sequence of tokens. Sentence boundaries are marked by introducing an artificial "start-of-sentence" state (`"--s--"`) occuring with "newline" tokens (`"--n--"`). It takes about 60 seconds to train the model and decode the development split.

## Run the tagger

The HMM is implemented in [`scripts/hmm.py`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/scripts/hmm.py). The trained model with transition, emission, and state counts is stored in [`data/hmm_model.txt`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/data/hmm_model.txt). A sorted list of vocabulary tokens is stored in [`data/hmm_vocab.txt`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/data/hmm_vocab.txt).

The Viterbi algorithm is implemented in [`scripts/viterbi.py`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/scripts/viterbi.py). Output files containing the predicted POS tags are written to the [`output/`](https://github.com/melanietosik/viterbi-pos-tagger/tree/master/output) directory. All settings can be adjusted by editing the paths specified in [`scripts/settings.py`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/scripts/settings.py).

To **(re-)run the tagger on the development and test set**, run:

```
[viterbi-pos-tagger]$ python3.6 scripts/hmm.py dev
[viterbi-pos-tagger]$ python3.6 scripts/hmm.py test
```

You should expect similar output pretty much immediately:

```
[viterbi-pos-tagger]$ python3.6 scripts/hmm.py dev
Generating vocabulary...
Training model...
Decoding dev split...
Words processed:     5000
Words processed:    10000
Words processed:    15000
Words processed:    20000
Words processed:    25000
Words processed:    30000
Done
python scripts/hmm.py dev  64.14s user 0.75s system 95% cpu 1:07.72 total
```

Please note that unless you run `rm -rf data/hmm*` to delete the old model files, they will _not_ be regenerated during the next run.

## Evaluation

The evaluation script is implemented in [`scripts/eval.py`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/scripts/eval.py). It prints a [text report](http://scikit-learn.org/stable/modules/generated/sklearn.metrics.classification_report.html) showing the main classification metrics, as well as the overall [accuracy classification score](http://scikit-learn.org/stable/modules/generated/sklearn.metrics.accuracy_score.html). It also writes a [confusion matrix](https://pandas.pydata.org/pandas-docs/stable/generated/pandas.crosstab.html) to [`docs/confusion_matrix.csv`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/data/confusion_matrix.csv).

### Run the evaluation script

First create a [virtual environment](https://virtualenv.pypa.io/en/stable/) and `pip install` all the requirements:

```
[viterbi-pos-tagger]$ virtualenv -p python3.6 env/
[viterbi-pos-tagger]$ source env/bin/activate
[viterbi-pos-tagger]$ pip install -r requirements.txt
```

Then run the evaluation script as follows:

```
[viterbi-pos-tagger]$ python scripts/eval.py <TRUE .pos> <PREDICTED .pos>
```

To evaluate the results on the development and test set, run:

```
[viterbi-pos-tagger]$ python scripts/eval.py WSJ/WSJ_24.pos output/wsj_24.pos  # dev
[viterbi-pos-tagger]$ #python scripts/eval.py WSJ/WSJ_23.pos output/wsj_23.pos  # test
```

### Results on the development set

As usual, section 24 of the WSJ corpus is used as the development set. The tagged output file for the development set is [`output/wsj_24.pos`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/output/wsj_24.pos). The original corpus files are [`WSJ/WSJ_24.words`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/WSJ/WSJ_24.words) and [`WSJ/WSJ_24.pos`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/WSJ/WSJ_24.pos).

Initially, Viterbi decoding with a uniform probability for unknown words and add-one smoothing gave a tagging accuracy of 92.88% on the development set. Adding morphological features to improve the handling of unknown words increased accuracy to a score of 93.34%. Finally, tuning the additive smoothing parameter resulted in a **tagging accuracy score of 95.31% on the development set**.

For more details, please see [`docs/accuracy.md`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/docs/accuracy.md).

| alpha | accuracy score     |
|------:|-------------------:|
|   1.0 | 0.9334307369190028 |
|   0.5 | 0.9419839892856056 |
|   0.2 | 0.9474020637384714 |
|   0.1 | 0.9498980306212522 |
| 0.001 | 0.953063647155511 |

Below is the classification report for the tagging accuracy on the development set.

```
             precision    recall  f1-score   support

          #       1.00      1.00      1.00         3
          $       1.00      1.00      1.00       216
         ''       1.00      1.00      1.00       247
          (       1.00      1.00      1.00        54
          )       1.00      1.00      1.00        53
          ,       1.00      1.00      1.00      1671
          .       1.00      1.00      1.00      1337
          :       1.00      1.00      1.00       221
         CC       1.00      0.99      1.00       877
         CD       0.98      0.98      0.98      1054
         DT       0.99      0.99      0.99      2856
         EX       0.97      1.00      0.99        37
         FW       0.29      0.50      0.36         8
         IN       0.99      0.95      0.97      3612
         JJ       0.86      0.94      0.90      2036
        JJR       0.86      0.87      0.87        93
        JJS       0.96      0.94      0.95        53
         LS       1.00      0.60      0.75         5
         MD       1.00      0.98      0.99       339
         NN       0.96      0.94      0.95      4541
        NNP       0.94      0.97      0.95      3216
       NNPS       0.77      0.51      0.62       127
        NNS       0.93      0.96      0.94      2050
        PDT       0.88      0.95      0.91        22
        POS       0.99      0.99      0.99       299
        PRP       0.99      0.99      0.99       538
       PRP$       0.99      1.00      0.99       271
         RB       0.87      0.91      0.89      1044
        RBR       0.73      0.76      0.75        54
        RBS       0.95      0.95      0.95        20
         RP       0.54      0.89      0.67        87
        SYM       1.00      0.80      0.89        10
         TO       1.00      1.00      1.00       805
         UH       0.33      0.25      0.29         4
         VB       0.95      0.93      0.94      1010
        VBD       0.94      0.90      0.92      1020
        VBG       0.93      0.82      0.87       528
        VBN       0.85      0.82      0.83       758
        VBP       0.92      0.89      0.90       422
        VBZ       0.94      0.95      0.94       701
        WDT       0.91      0.95      0.93       123
         WP       0.97      0.99      0.98        90
        WP$       1.00      1.00      1.00         7
        WRB       1.00      0.99      0.99        83
         ``       1.00      1.00      1.00       251

avg / total       0.95      0.95      0.95     32853
```

### Results on the test set

Section 23 of the WSJ corpus is usually reserved for testing. The tagged output file for the test set is [`output/wsj_23.pos`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/output/wsj_23.pos). The original corpus file is [`WSJ/WSJ_23.words`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/WSJ/WSJ_23.words). Note that the original `.pos` file for the test set has not yet been released.

To achieve optimal results on the test split, the additive smoothing alpha parameter is set to `alpha = 0.001`. The training file is set to [`WSJ/WSJ_02-21.pos`](https://github.com/melanietosik/viterbi-pos-tagger/blob/master/WSJ/WSJ_02-21.pos).




==============================

I built an NLP model that does Part-of-Speech Tagging for Classical Greek.

Greek is a hard, low-resource language.

Developing a great POS Tagger will require a lot of attention to detail, smart feature engineering, and a thoughtful approach to model selection. 

Let us motivate the task by providing background on the difficulties.

##  English Part-of-Speech Tagging is kind of easy 

English makes it easy to figure out part-of-speech. 

First off, sentences typically follow a strict Subject-Verb-Object order:

![img/SVO.png](img/SVO.png)

Also, word endings are pretty simple:

![img/conj.png](img/conj.png)

And lastly, we have *a lot* of it:

![img.png](img/eng_tokens.png)

## Greek is hard

Greek, on the other hand, is not so simple.

Firstly, the words can appear in any order (and they do). 

Secondly, part-of-speech is determined by an extremely complex system of word endings:
![img/greek.png](img/greek.png)
*Present tense conjugation for one verb φύω, to appear. There are 5 other tenses*

Lastly, we do not have a lot of Greek, and we do have is largely the most difficult and abstruse subset of the language: namely, poetry, philosophical dialogue, and religious texts.

## POS Classical Greek: a true test of NLP 

All of this makes Classical Greek a perfect language to study NLP -- and vice-versa.

# Steps
## Previous work shows 

## Treebanks 

## 

## Tools used in this project
* [Poetry](https://towardsdatascience.com/how-to-effortlessly-publish-your-python-package-to-pypi-using-poetry-44b305362f9f): Dependency management - [article](https://mathdatasimplified.com/2023/06/12/poetry-a-better-way-to-manage-python-dependencies/)
* [hydra](https://hydra.cc/): Manage configuration files - [article](https://mathdatasimplified.com/2023/05/25/stop-hard-coding-in-a-data-science-project-use-configuration-files-instead/)
* [pre-commit plugins](https://pre-commit.com/): Automate code reviewing formatting
* [DVC](https://dvc.org/): Data version control - [article](https://mathdatasimplified.com/2023/02/20/introduction-to-dvc-data-version-control-tool-for-machine-learning-projects-2/)
* [pdoc](https://github.com/pdoc3/pdoc): Automatically create an API documentation for your project

## Set up the environment
1. Install [Poetry](https://python-poetry.org/docs/#installation)
2. Set up the environment:
```bash
make env 
```

## Install dependencies
To install all dependencies for this project, run:
```bash
poetry install
```

To install a new package, run:
```bash
poetry add <package-name>
```

## Version your data
To track changes to the "data" directory, type:
```bash
dvc add data
```

This command will create the "data.dvc" file, which contains a unique identifier and the location of the data directory in the file system.

To keep track of the data associated with a particular version, commit the "data.dvc" file to Git:
```bash
git add data.dvc
git commit -m "add data"
```

To push the data to remote storage, type:
```bash
dvc push 
```

## Auto-generate API documentation

To auto-generate API document for your project, run:

```bash
make docs
```
