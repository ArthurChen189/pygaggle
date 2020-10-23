# PyGaggle: Neural Ranking Baselines on [MS MARCO Passage Retrieval](https://github.com/microsoft/MSMARCO-Passage-Ranking) - Dev Subset 

This page contains instructions for running various neural reranking baselines on the MS MARCO *passage* ranking task. 
Note that there is also a separate [MS MARCO *document* ranking task](https://github.com/castorini/anserini/blob/master/docs/experiments-msmarco-doc.md).

Prior to running this, we suggest looking at our first-stage [BM25 ranking instructions](https://github.com/castorini/anserini/blob/master/docs/experiments-msmarco-passage.md).
We rerank the BM25 run files that contain ~1000 passages per query using both monoBERT and monoT5.
monoBERT and monoT5 are pointwise rerankers. This means that each document is scored independently using either BERT or T5 respectively.

Since it can take many hours to run these models on all of the 6980 queries from the MS MARCO dev set, we will instead use a subset of 105 queries randomly sampled from the dev set. 
Running these instructions with the entire MS MARCO dev set should give about the same results as that in the corresponding paper. 

Note 1: Run the following instructions at root of this repo.
Note 2: Make sure that you have access to a GPU
Note 3: Installation must have been done from source and make sure the [anserini-eval](https://github.com/castorini/anserini-eval) submodule is pulled. 
To do this, first clone the repository recursively.

```
git clone --recursive https://github.com/castorini/pygaggle.git
```

Then install PyGaggle using:

```
pip install pygaggle/
```

## Models

+ monoBERT-Large: Passage Re-ranking with BERT [(Nogueira et al., 2019)](https://arxiv.org/pdf/1901.04085.pdf)
+ monoT5-base: Document Ranking with a Pretrained Sequence-to-Sequence Model [(Nogueira et al., 2020)](https://arxiv.org/pdf/2003.06713.pdf)

## Data Prep

We're first going to download the queries, qrels and run files corresponding to the MS MARCO set considered. The run file is generated by following the BM25 ranking instructions. We'll store all these files in the `data` directory.

```
wget https://www.dropbox.com/s/5xa5vjbjle0c8jv/msmarco_ans_small.zip -P data
```

To confirm, `msmarco_ans_small.zip` should have MD5 checksum of `65d8007bfb2c72b5fc384738e5572f74`.

Next, we extract the contents into `data`. 

```
unzip data/msmarco_ans_small.zip -d data
```

As a sanity check, we can evaluate the first-stage retrieved documents using the official MS MARCO evaluation script.

```
python tools/eval/msmarco_eval.py data/msmarco_ans_small/qrels.dev.small.tsv data/msmarco_ans_small/run.dev.small.tsv
```

The output should be:

```
#####################
MRR @10: 0.15906651549508694
QueriesRanked: 105
#####################
```

Let's download and extract the pre-built MS MARCO index into `indexes`:

```
wget https://git.uwaterloo.ca/jimmylin/anserini-indexes/raw/master/index-msmarco-passage-20191117-0ed488.tar.gz -P indexes
tar xvfz indexes/index-msmarco-passage-20191117-0ed488.tar.gz -C indexes
```

Now, we can begin with re-ranking the set.

## Re-Ranking with monoBERT

First, lets evaluate using monoBERT!

```
python -um pygaggle.run.evaluate_passage_ranker --split dev \
                                                --method seq_class_transformer \
                                                --model castorini/monobert-large-msmarco \
                                                --dataset data/msmarco_ans_small/ \
                                                --index-dir indexes/index-msmarco-passage-20191117-0ed488 \
                                                --task msmarco \
                                                --output-file runs/run.monobert.ans_small.dev.tsv
```

Upon completion, the following output will be visible:

```
precision@1     0.2761904761904762
recall@3        0.42698412698412697
recall@50       0.8174603174603176
recall@1000     0.8476190476190476
mrr     0.41089693612003686
mrr@10  0.4026795162509449
```

It takes about ~52 minutes to re-rank this subset on MS MARCO using a P100. 
The type of GPU will directly influence your inference time. 
It is possible that the default batch results in a GPU OOM error.
In this case, assigning a batch size (using option `--batch-size`) which is smaller than the default (96) should help!

The re-ranked run file `run.monobert.ans_small.dev.tsv` will also be available in the `runs` directory upon completion.

We can use the official MS MARCO evaluation script to verify the MRR@10:

```
python tools/eval/msmarco_eval.py data/msmarco_ans_small/qrels.dev.small.tsv runs/run.monobert.ans_small.dev.tsv
```

You should see the same result. Great, let's move on to monoT5!

## Re-Ranking with monoT5

We use the monoT5-base variant as it is the easiest to run without access to larger GPUs/TPUs. Let us now re-rank the set:

```
python -um pygaggle.run.evaluate_passage_ranker --split dev \
                                                --method t5 \
                                                --model castorini/monot5-base-msmarco \
                                                --dataset data/msmarco_ans_small \
                                                --model-type t5-base \
                                                --task msmarco \
                                                --index-dir indexes/index-msmarco-passage-20191117-0ed488 \
                                                --batch-size 32 \
                                                --output-file runs/run.monot5.ans_small.dev.tsv
```

The following output will be visible after it has finished:

```
precision@1     0.26666666666666666
recall@3        0.4603174603174603
recall@50       0.8063492063492063
recall@1000     0.8476190476190476
mrr     0.3973368360121561
mrr@10  0.39044217687074834
```

It takes about ~13 minutes to re-rank this subset on MS MARCO using a P100. 
It is worth noting again that you might need to modify the batch size to best fit the GPU at hand.

Upon completion, the re-ranked run file `run.monot5.ans_small.dev.tsv` will be available in the `runs` directory.

We can use the official MS MARCO evaluation script to verify the MRR@10:

```
python tools/eval/msmarco_eval.py data/msmarco_ans_small/qrels.dev.small.tsv runs/run.monot5.ans_small.dev.tsv
```

You should see the same result.

If you were able to replicate these results, please submit a PR adding to the replication log!


## Replication Log

+ Results replicated by [@MXueguang](https://github.com/MXueguang) on 2020-05-22 (commit [`69de7db`](https://github.com/castorini/pygaggle/commit/69de7db843bbe9201113c4d94c9e90be36094350)) (Tesla P4)
+ Results replicated by [@richard3983](https://github.com/richard3983) on 2020-05-22 (commit [`6e9dfc6`](https://github.com/richard3983/pygaggle/commit/6e9dfc62083c15233600c41737110c9989043b98)) (Tesla P100)
+ Results replicated by [@HangCui0510](https://github.com/HangCui0510) on 2020-05-29 (commit [`591e7ff`](https://github.com/HangCui0510/pygaggle/commit/591e7ffd6cc826fd2bae5e721f9693452f9e4a49)) (Tesla P100)
+ Results replicated by [@kelvin-jiang](https://github.com/kelvin-jiang) on 2020-05-31 (commit [`82dc086`](https://github.com/HangCui0510/pygaggle/commit/82dc086b86d828147dad34d9a7f8bb66a3c23c88)) (GeForce RTX 2080 Ti)
+ Results replicated by [@justinborromeo](https://github.com/justinborromeo) on 2020-07-02 (commit [`70b2a9f`](https://github.com/castorini/pygaggle/commit/70b2a9fe625554aeae02f64eb68f1edc57f96860)) (GeForce GTX 960M)
+ Results replicated by [@mrkarezina](https://github.com/mrkarezina) on 2020-07-19 (commit [`c1a54cb`](https://github.com/castorini/pygaggle/commit/c1a54cb012a1d4ea24a2ce2bc24298417279a9c4)) (Tesla T4)
+ Results replicated by [@qguo96](https://github.com/qguo96) on 2020-09-08 (commit [`94befbd`](https://github.com/qguo96/pygaggle/commit/94befbd58b19c3e46d930e67797102bf174efd01)) (Tesla T4 on Colab)
+ Results replicated by [@yuxuan-ji](https://github.com/yuxuan-ji) on 2020-09-08 (commit[`94befbd`](https://github.com/castorini/pygaggle/commit/94befbd58b19c3e46d930e67797102bf174efd01)) (Tesla T4 on Colab)
+ Results replicated by [@LizzyZhang-tutu](https://github.com/LizzyZhang-tutu) on 2020-09-09 (commit[`8eeefa5`](https://github.com/castorini/pygaggle/commit/8eeefa578c65e2da78be129c87dfb40beb74099c)) (Tesla T4 on Colab)
+ Results replicated by [@wiltan-uw](https://github.com/wiltan-uw) on 2020-09-13 (commit[`41513a9`](https://github.com/castorini/pygaggle/commit/41513a9f496bd59523993ce134cc35a7b881e5a1)) (RTX 2070S)
+ Results replicated by [@jhuang265](https://github.com/jhuang265) on 2020-10-18 (commit[`e815051`](https://github.com/castorini/pygaggle/commit/e815051f2cee1af98b370ee030b66c07a8a287f3)) (Tesla P100 on Colab)