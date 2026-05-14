---
created: 2026-05-12T10:43:50 (UTC +08:00)
tags: []
source: https://arxiv.org/html/2403.10059v2
author: Di Wu

  
Wasi Uddin Ahmad

  
Dejiao Zhang

  
Murali Krishna Ramanathan

  
Xiaofei Ma
---

# Repoformer: Selective Retrieval for Repository-Level Code Completion

> ## Excerpt
> Recent advances in retrieval-augmented generation (RAG) have initiated a new era in repository-level code completion. However, the invariable use of retrieval in existing methods exposes issues in both efficiency and robustness, with a large proportion of the retrieved contexts proving unhelpful or harmful to code language models (code LMs). In this paper, we propose a selective RAG framework to avoid retrieval when unnecessary. To power this framework, we design a self-supervised learning approach to enable a code LM to accurately self-evaluate whether retrieval can improve its output quality and robustly leverage the potentially noisy retrieved contexts. Using this LM as both the selective RAG policy and the generation model, our framework achieves state-of-the-art repository-level code completion performance on diverse benchmarks including RepoEval, CrossCodeEval, and CrossCodeLongEval, a new long-form code completion benchmark. Meanwhile, our analyses show that selectively retrieving brings as much as 70% inference speedup in the online serving setting without harming the performance. We further demonstrate that our framework is able to accommodate different generation models, retrievers, and programming languages. These advancements position our framework as an important step towards more accurate and efficient repository-level code completion.

---
###### Abstract

Recent advances in retrieval-augmented generation (RAG) have initiated a new era in repository-level code completion. However, the invariable use of retrieval in existing methods exposes issues in both efficiency and robustness, with a large proportion of the retrieved contexts proving unhelpful or harmful to code language models (code LMs). In this paper, we propose a selective RAG framework to avoid retrieval when unnecessary. To power this framework, we design a self-supervised learning approach to enable a code LM to accurately self-evaluate whether retrieval can improve its output quality and robustly leverage the potentially noisy retrieved contexts. Using this LM as both the selective RAG policy and the generation model, our framework achieves state-of-the-art repository-level code completion performance on diverse benchmarks including RepoEval, CrossCodeEval, and CrossCodeLongEval, a new long-form code completion benchmark. Meanwhile, our analyses show that selectively retrieving brings as much as 70% inference speedup in the online serving setting without harming the performance. We further demonstrate that our framework is able to accommodate different generation models, retrievers, and programming languages. These advancements position our framework as an important step towards more accurate and efficient repository-level code completion.

## 1 Introduction

![Refer to caption](https://arxiv.org/html/2403.10059v2/x1.png)

Figure 1: An overview of the proposed selective RAG framework. Given the current file context, the system first assesses whether retrieval is required and triggers the retriever if the question can likely be benefited from retrieval (right), abstaining from retrieval otherwise (left). Then, the code LM generates with optional retrieved contexts. With Repoformer, the two stages are streamlined via self-assessment.

Automatic code completion has attracted long-lasting research efforts due to its high practical value in improving programmer productivity (Ye & Fischer, [2002](https://arxiv.org/html/2403.10059v2#bib.bib38); Hill & Rideout, [2004](https://arxiv.org/html/2403.10059v2#bib.bib11); Hellendoorn & Devanbu, [2017](https://arxiv.org/html/2403.10059v2#bib.bib10)). One particularly challenging scenario is repository-level code completion, where a system is required to complete lines, API invocations, or functions in a file from user repositories. For this task, language models for code (code LMs) have emerged as a promising solution due to their ability to leverage the context of the current file to generate coherent code of flexible granularity (Tu et al., [2014](https://arxiv.org/html/2403.10059v2#bib.bib35); Svyatkovskiy et al., [2020](https://arxiv.org/html/2403.10059v2#bib.bib34); Chen et al., [2021](https://arxiv.org/html/2403.10059v2#bib.bib4)). However, these approaches fail to capture the holistic repository knowledge spanning beyond the current file, such as user-defined APIs and inter-module dependencies (Zan et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib39); Zhang et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib40); Ding et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib5)). Recently, the retrieval-augmented generation (RAG) paradigm was proposed to bridge the gap: cross-file contexts such as relevant code snippets or documentations are retrieved and provided to code LMs as augmentations to the current file. This approach has shown strong empirical performance and was further advanced by recent literature through designing better retrieval mechanisms for prompting black-box code LMs (Lu et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib21); Shrivastava et al., [2023b](https://arxiv.org/html/2403.10059v2#bib.bib33); Zhang et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib40)) and adapting the LM to better leverage structured retrieved contexts such as classes, functions, or APIs (Ding et al., [2024](https://arxiv.org/html/2403.10059v2#bib.bib6); Zan et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib39)).

Despite their encouraging performance, existing RAG-based approaches largely ignore to address a critical question:

Should we always perform retrieval augmentation?

Our findings suggest that the answer is predominantly negative. First, in various code completion tasks, we discover that up to 80% of the retrievals performed by a standard RAG method do not enhance the performance of common code LMs such as CodeGen (Nijkamp et al., [2023b](https://arxiv.org/html/2403.10059v2#bib.bib24)) and StarCoder (Li et al., [2023b](https://arxiv.org/html/2403.10059v2#bib.bib20)), and many degrade the performance by introducing irrelevant information ([Section 5.1](https://arxiv.org/html/2403.10059v2#S5.SS1 "5.1 Is retrieval always helpful? ‣ 5 Results ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")). Second, always retrieving introduces notable inefficiencies. For moderately sized repositories, sparse retrieval is already as time consuming as code completion with a 3B code LM ([Section 5.3](https://arxiv.org/html/2403.10059v2#S5.SS3 "5.3 Repoformer improves inference efficiency ‣ 5 Results ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") and [Section 6](https://arxiv.org/html/2403.10059v2#S6 "6 Analysis ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")). This inefficiency is more pronounced with dense retrieval, enterprise-scale repositories, and iterative RAG methods such as Zhang et al. ([2023](https://arxiv.org/html/2403.10059v2#bib.bib40)).

In this paper, we challenge the assumption of always retrieving by proposing a novel repository-level code completion framework underpinned by a selective retrieval mechanism: the system proactively abstains from performing unnecessary or potentially detrimental retrievals ([Figure 1](https://arxiv.org/html/2403.10059v2#S1.F1 "In 1 Introduction ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") (a)). At the core of our framework is Repoformer, an intelligent code LM fine-tuned for robust code completion with self-triggered retrieval augmentation. Repoformer reflects three core principles:

1.  1.

    Performance-oriented self-evaluation. After observing the current file, Repoformer explicitly expresses the likelihood that its prediction quality could be improved by cross-file retrieval. Our training strategy enables the model to combine two factors in this decision: the code LM already knowing the answer without retrieval (Kadavath et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib15)) and the code completion question not depending on cross-file information and thus retrieval is likely uninformative.

2.  2.

    Robustness to retrieved contexts. Repoformer learns to use the retrieved contexts to improve the quality of its output and avoid performance drops caused by potentially noisy retrieved information.

3.  3.

    Generalizability. The aforementioned two abilities must generalize to any completion granularity, programming language, and retriever choice. In addition, Repoformer should be able to function as a plug-and-play selective retrieval policy when other models are employed as the generation model.


We posit that these abilities can be faithfully obtained by learning from simulations of RAG. Specifically, we leverage a large number of permissively licensed repositories, sample diverse blanks to complete, and pair them with the retrieved repository-level cross-file contexts. Then, for a given code LM, the ground-truth label for selective retrieval is obtained by contrasting the quality of its outputs with and without retrieval augmentation. With this dataset, we design a self-supervised objective to jointly train code LMs to accurately self-evaluate the need for retrieval and robustly complete the code with the optional retrieval augmentation ([Section 3.3](https://arxiv.org/html/2403.10059v2#S3.SS3 "3.3 Self-supervised Multi-task Learning ‣ 3 Approach ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")).

We perform comprehensive evaluations on a range of repository-level code completion tasks from RepoEval (Zhang et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib40)), CrossCodeEval (Ding et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib5)), and CrossCodeLongEval a new large-scale benchmark focusing on code chunk and function completion. Results show that Repoformer achieves strong performance, outperforming always retrieving with the same-sized StarCoderBase by more than 3 absolute points for edit similarity across multiple tasks. The 3B Repoformer performs on par with always retrieving using the 16B StarCoder, and the 16B Repoformer achieves state-of-the-art performance across all the tasks ([Section 5.2](https://arxiv.org/html/2403.10059v2#S5.SS2 "5.2 Repoformer achieves strong code completion performance via selective RAG ‣ 5 Results ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")). Furthermore, our framework allows for up to 70% inference speedup without harming accuracy. We also establish that Repoformer can accelerate RAG with larger black-box LMs as a plug-and-play selective RAG policy, improving the performance while reducing the latency of line and API completion to 75% ([Section 5.3](https://arxiv.org/html/2403.10059v2#S5.SS3 "5.3 Repoformer improves inference efficiency ‣ 5 Results ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")).

Finally, in [Section 6](https://arxiv.org/html/2403.10059v2#S6 "6 Analysis ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"), we provide comprehensive analyses on Repoformer’s generalization ability. We show that Repoformer makes precise retrieval abstention decisions, is robust to retrieved contexts, and performs well when tested in other languages or with other retrievers. To facilitate future research on repository-level code completion, we will release our implementation and the CrossCodeLongEval benchmark at [https://repoformer.github.io/](https://repoformer.github.io/).

## 2 Related Work

#### Repository-level Code Completion

Accurately completing the code in repositories has been a challenging research problem due to cross-file dependency patterns caused by modular design (Parnas, [1972](https://arxiv.org/html/2403.10059v2#bib.bib26); Tu et al., [2014](https://arxiv.org/html/2403.10059v2#bib.bib35)). Early works propose application-specific training methods for n-gram LMs (Tu et al., [2014](https://arxiv.org/html/2403.10059v2#bib.bib35)), RNNs (Hellendoorn & Devanbu, [2017](https://arxiv.org/html/2403.10059v2#bib.bib10); Wang et al., [2021](https://arxiv.org/html/2403.10059v2#bib.bib36)), and Transformers (Svyatkovskiy et al., [2020](https://arxiv.org/html/2403.10059v2#bib.bib34)) to leverage structured knowledge beyond current file’s context. Recent studies investigate fine-tuning powerful pre-trained code LMs (Chen et al., [2021](https://arxiv.org/html/2403.10059v2#bib.bib4); Nijkamp et al., [2023b](https://arxiv.org/html/2403.10059v2#bib.bib24); Li et al., [2023b](https://arxiv.org/html/2403.10059v2#bib.bib20)) to better leverage retrieved knowledge provided in context such as code and documentation snippets (Zan et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib39); Ding et al., [2024](https://arxiv.org/html/2403.10059v2#bib.bib6); Shrivastava et al., [2023a](https://arxiv.org/html/2403.10059v2#bib.bib32)). Concurrently, other studies show that black-box code LMs can already take advantage of in-context knowledge, depending on how well the knowledge is retrieved and formatted (Lu et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib21); Zhou et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib41); Shrivastava et al., [2023b](https://arxiv.org/html/2403.10059v2#bib.bib33); Zhang et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib40)). This approach does not require one to train the LM and thus promises better generalization. Orthogonal to these studies, this paper identifies and addresses the robustness and efficiency issues caused by invariably performing the retrieval augmentation. Our solution takes the form of selective retrieval augmentation through self-assessment.

#### Adaptive RAG

This paper is consistent with the recent trend of making the RAG paradigm active and adaptive. A core question is finding an effective policy to decide when to retrieve. He et al. ([2021](https://arxiv.org/html/2403.10059v2#bib.bib9)) propose to learn to adjust the importance weight of retrieval based on language modeling performance. Drozdov et al. ([2022](https://arxiv.org/html/2403.10059v2#bib.bib7)) proposes to upweight the retrieved information when the retrieval has high quality. Li et al. ([2023a](https://arxiv.org/html/2403.10059v2#bib.bib19)) and Jiang et al. ([2023](https://arxiv.org/html/2403.10059v2#bib.bib14)) suggest that retrieval should be performed only when LMs have a high predictive uncertainty. Mallen et al. ([2023](https://arxiv.org/html/2403.10059v2#bib.bib22)) discover that retrieval can be avoided for popular facts. Concurrent to this work, two new studies approach adaptive RAG from a learning perspective. SKR (Wang et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib37)) collects instances where retrieval is not helpful for black-box LMs and proposes several methods to predict these instances. Self-RAG (Asai et al., [2024](https://arxiv.org/html/2403.10059v2#bib.bib1)) utilizes GPT-4 (OpenAI, [2023](https://arxiv.org/html/2403.10059v2#bib.bib25)) as a knowledge engine to distill a smaller LM to evaluate whether answering a question can be benefited from retrieval. In comparison, this paper highlights the importance of understanding whether an LM knows the answer (Kadavath et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib15)) in forming the retrieval policy. We introduce a simple yet effective scheme to fine-tune a code LM for faithful self-evaluation without extra modules (SKR), knowledge store (SKR), or labels generated by an oracle LM (Self-RAG). We show that our approach leads to no performance harms ([Section 5.2](https://arxiv.org/html/2403.10059v2#S5.SS2 "5.2 Repoformer achieves strong code completion performance via selective RAG ‣ 5 Results ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")), substantial speedup ([Section 5.3](https://arxiv.org/html/2403.10059v2#S5.SS3 "5.3 Repoformer improves inference efficiency ‣ 5 Results ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")), and a high decision accuracy ([Section 6](https://arxiv.org/html/2403.10059v2#S6 "6 Analysis ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")).

## 3 Approach

In this section, we first briefly formulate the repository-level code completion task and the considered RAG setup. Then, we illustrate the details of the proposed framework.

### 3.1 Background

#### Problem Formulation

We denote each repository-level code completion task as $(X_{l},X_{r},Y,F)$. $Y$ is the ground truth completion that needs to be generated. In this paper, $Y$ always contains one or more consecutive lines of code. $X_{l}$ and $X_{r}$ are the code to the left/right of $Y$ in the same file. We will use the left/right context to refer to them. $F$ is the set of other files in the repository. A code completion system utilizes $X_{l}$, $X_{r}$, and $F$ to generate a hypothesis $\hat{Y}$.

#### Retrieval-Augmented Generation

We follow the RG-1 formulation in Zhang et al. ([2023](https://arxiv.org/html/2403.10059v2#bib.bib40)) to execute RAG for code completion in four stages: indexing, query formation, retrieval, and generation. We consider two components:

-   •

    An in-repository retriever $\mathcal{R}$ that queries $F$ with information from $X_{l}$ and $X_{r}$ and returns relevant cross-file contexts $CC$. $CC$ consists of $k$ code chunks $cc_{1},cc_{2},...,cc_{k}$, each of which contains consecutive lines of code extracted from a file in $F$. We mainly use Jaccard similarity (Jaccard, [1912](https://arxiv.org/html/2403.10059v2#bib.bib12)) as $\mathcal{R}$ due to its speed and strong performance (Zhang et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib40)).

-   •

    A code LM $\mathcal{M}$ that leverages $X_{l}$, $X_{r}$, and $CC$ to output $\hat{Y}$. The inclusion of $X_{r}$ and $CC$ is optional. In this paper, we always directly provide $X_{r}$ in the prompt in addition to $X_{l}$ (Shrivastava et al., [2023b](https://arxiv.org/html/2403.10059v2#bib.bib33); Pei et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib27)). We provide empirical support for this design in [Appendix B](https://arxiv.org/html/2403.10059v2#A2 "Appendix B Why infilling? ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion").


Full documentation of the RAG stages and their hyperparameters are provided in [Appendix A](https://arxiv.org/html/2403.10059v2#A1 "Appendix A Detailed RAG Execution Setup ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") for further reference.

### 3.2 Self-selective RAG for Code Completion

Central to our framework is the idea of selective RAG, where the system decides whether the LM’s generation could benefit from retrieved contexts and abstains from retrieval augmentation when it is deemed unnecessary ([Figure 1](https://arxiv.org/html/2403.10059v2#S1.F1 "In 1 Introduction ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")).

For this selective decision, two traditional heuristics are relevant: (1) performing a trial retrieval and only augmenting the high-relevance contexts (e.g., Drozdov et al. ([2022](https://arxiv.org/html/2403.10059v2#bib.bib7))) or (2) performing a trial generation and conducting RAG only when the model’s uncertainty is high (e.g., Jiang et al. ([2023](https://arxiv.org/html/2403.10059v2#bib.bib14))). For repository-level code completion, these strategies are informative to some extent: in line completion and API completion from RepoEval, both heuristics can maintain the same level of performance with only 50% retrieval budget. However, we find that they fail to generalize well to all tasks and still incur a high latency cost as they need to conduct retrieval to make the decisions ([Appendix C](https://arxiv.org/html/2403.10059v2#A3 "Appendix C Trial Retrieval and Trial Generation ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")).

Instead, our framework adopts a self-selective RAG formulation. After observing $X_{l}$ and $X_{r}$, the LM directly self-triggers cross-file retrieval by generating a special token <cc> or abstains from retrieval via an empty token $\phi$<sup>1</sup><sup>1</sup>1In practice, instead of greedily decoding <cc>, we check whether its probability exceeds a certain threshold.. This approach is inspired by the explorations in Kadavath et al. ([2022](https://arxiv.org/html/2403.10059v2#bib.bib15)), which show that an LM can be trained to predict whether it knows the answer or not without retrieval. Beyond this self-knowledge, our model also combines the question’s characteristics (i.e., whether retrieving cross-file information can likely help or not) in its judgment, as we will discuss in the next section. Finally, after the optional retrieval, the LM proceeds with the code completion with $X_{l}$, $X_{r}$, combined with $CC$ if retrieval is triggered.

Implementation-wise, self-selective RAG’s inference is conveniently modeled as an extension to fill-in-the-middle (Bavarian et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib2)), with the entire process executed in a single left-to-right pass ([Figure 2](https://arxiv.org/html/2403.10059v2#S3.F2 "In Data construction ‣ 3.3 Self-supervised Multi-task Learning ‣ 3 Approach ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")). One advantage of this design is the flexibility. The LM possesses the ability for RAG and fill-in-the-middle, and can seamlessly self-switch between the two when encountering different questions. Users can also easily adjust the ratio between the two through the retrieval threshold. Another advantage is its efficiency. The selective decision overhead is only a single forward pass, a significant save compared to making the retrieval decision via trial generation or trial retrieval. When the LM abstains from retrieval, it can directly proceed with generation and the retrieval overhead is completely avoided.

### 3.3 Self-supervised Multi-task Learning

To power self-selective RAG, the LM needs two crucial abilities: accurate self-assessment and robustness to the retrieved context. We design a contrastive data labeling scheme to mine self-supervision from public repositories, followed by fine-tuning with a novel multi-task objective.

#### Data construction

We leverage large-scale permissively licensed repositories from the Stack (Kocetkov et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib16)) and create the fine-tuning data via a three-step procedure:

1.  1.

    Sample target lines $Y$ that are either (1) random code chunks of varied lengths or (2) function bodies.

2.  2.

    Retrieve $CC$ using the current file. We include $Y$ in the query for 50% of the data<sup>2</sup><sup>2</sup>2The main goal of the design is to align better with both non-iterative and iterative RAG use cases. During testing, a user may retrieve with both the in-file context and $Y’$, a model’s draft prediction, which results in a $CC$ distribution close to that with $Y$ in the query (Zhang et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib40)). .

3.  3.

    Label whether extending the current file with $CC$ can improve a code LM $\mathcal{M}$’s code completion quality by more than a threshold $T$, measured by Edit Similarity (ES, definition in [Section 4.1](https://arxiv.org/html/2403.10059v2#S4.SS1 "4.1 Repoformer Implementation Details ‣ 4 Experimental Setup ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")) against $Y$.


![Refer to caption](https://arxiv.org/html/2403.10059v2/x2.png)

Figure 2: A comparison between fill-in-the-middle and self-selective RAG. We mark the end of the current file with a new token <eof>, which triggers the LM’s self-evaluation. $\rightarrow$ denotes the invocation of the LM. We color current-file context, retrieved contexts, and LM-generated parts in blue, green, and red respectively. fim\_p, fim\_s, and fim\_m refer to the special tokens for fill-in-the-middle: fim\_prefix, fim\_suffix, and fim\_middle. These tokens are already learned during the pre-training.

The full algorithms are presented in [Appendix D](https://arxiv.org/html/2403.10059v2#A4 "Appendix D Data Creation for Repoformer Training and CrossCodeLongEval ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"). After running the algorithm, we obtain the fine-tuning instances, each in the form $(X_{l},X_{r},\ Y,\ CC,\ label)$.

#### Verbalization

Each instance is verbalized into a sequence for fine-tuning. If $label$ is false, only $X_{l}$ and $X_{r}$ are provided preceding $Y$. Otherwise, we additionally provide $CC$ after the special token <cc>. The two verbalizations correspond to the two branches in [Figure 2](https://arxiv.org/html/2403.10059v2#S3.F2 "In Data construction ‣ 3.3 Self-supervised Multi-task Learning ‣ 3 Approach ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") (b).

#### Training Objective

We introduce two losses, $\mathcal{L}_{eval}$ for self-assessment and $\mathcal{L}_{gen}$ for code generation.

1.  1.

    $\mathcal{L}_{eval}$: a cross-entropy loss on predicting <cc> immediately following <eof>.

    |$$\mathcal{L}_{eval}=-\log p_{\mathcal{M}}(\text{{\textless cc\textgreater}}|X_{l},X_{r})$$|
    |---|---|

2.  2.

    $\mathcal{L}_{gen}$: a cross-entropy loss on the tokens following <fim\_middle>. Depending on $label$, $\mathcal{L}_{gen}$ represents either code completion with only in-file information or retrieval-augmented code completion.

    |$$\mathcal{L}_{gen}=\begin{cases}-\log p_{\mathcal{M}}(Y|X_{l},X_{r},CC),&\text{if }label\\ -\log p_{\mathcal{M}}(Y|
    |---|---|


The final training objective is $\lambda\mathcal{L}_{eval}+\mathcal{L}_{gen}$, a weighted combination of the two losses. We do not supervise the model on predicting the other tokens in $X_{l}$, $X_{r}$, $CC$, or the special tokens for fill-in-the-middle. Teacher forcing is used just as in normal causal language model training.

## 4 Experimental Setup

### 4.1 Repoformer Implementation Details

#### Training Data

We sample Python repositories from the Stack (Kocetkov et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib16)). Basic filtering are applied to retain 18k repositories that have (1) at least five Python files, (2) at least three imports per file, and (3) at least two local imports per file. These criteria ensure the existence of local dependencies where RAG could be helpful. We use $\mathcal{M}$ = StarCoderBase-1B and $T$ = $0$ to label 240k chunk and 120k function completion instances. We reserve 500 repositories for validation and use the rest for training.

#### Training

We fine-tune the 1B, 3B, 7B, and 16B variants of StarCoderBase with $\lambda=1.0$, maximum sequence length 2048, learning rate 2e-5, batch size 512, 50 warmup steps, and a linear learning rate decay. The models are trained for 2 epochs, which approximately takes 8, 12, 20, and 50 hours for the 1B/3B/7B/16B models respectively with 8 Nvidia A100 GPUs (40G memory). Our implementation is based on Jain et al. ([2023](https://arxiv.org/html/2403.10059v2#bib.bib13))<sup>3</sup><sup>3</sup>3[https://github.com/amazon-science/ContraCLM](https://github.com/amazon-science/ContraCLM). We will call our models Repoformer-1B/3B/7B/16B. We have also applied the same method to train a multilingual version of Repoformer on a mixture of Python, Java, C#, and Typescript repositories. As we focus on the methodological discussion in the main text, we refer interested readers to [Section E.2](https://arxiv.org/html/2403.10059v2#A5.SS2 "E.2 CrossCodeEval and Multilingual Repoformer ‣ Appendix E Extended Analyses ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") for the detailed experiment setup and results.

#### Hyperparameter optimization

We conduct a grid search with StarCoderBase-1B on the following search space: learning rate {1e-5, 2e-5, 5e-5}, $\lambda$ {0.2, 1.0, 2.0, 5.0}, training epochs {1, 2, 5}, and warmup steps {50, 100}. The best hyperparameters are selected based on the code completion performance on the validation dataset.

### 4.2 Evaluation Setup

#### Evaluation Datasets

We evaluate on RepoEval (Zhang et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib40)), which consists of line, API, and function completion tasks created from 32 Python repositories. To investigate the generalization to other languages, we also evaluated the original CrossCodeEval (Ding et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib5)), which features line completion instances covering four languages: Python, Java, C#, and TypeScript ([Section E.2](https://arxiv.org/html/2403.10059v2#A5.SS2 "E.2 CrossCodeEval and Multilingual Repoformer ‣ Appendix E Extended Analyses ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")). Observing that RepoEval has a limited repositrory coverage and that CrossCodeEval has a limited task coverage, we additionally leverage 1500 raw Python repositories from CrossCodeEval to create a new chunk and function completion benchmark, which we call CrossCodeLongEval. We detail the dataset creation process and basic statistics in [Appendix D](https://arxiv.org/html/2403.10059v2#A4 "Appendix D Data Creation for Repoformer Training and CrossCodeLongEval ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"). For the rest of this paper, we will use CCEval to refer to both CrossCodeEval and CrossCodeLongEval interchangeably, and use the specific language and task (line, chunk, or function completion) to differentiate them.

![Refer to caption](https://arxiv.org/html/2403.10059v2/x3.png)

Figure 3: The performance gain on RepoEval API completion from retrieved cross-file contexts. Each bucket contains values ranging from label-10 to label+10 except for the central bucket, which corresponds to exactly 0. The retrieved contexts only improve the performance in about 20% of instances. The trend is consistent across all the evaluated LM families and sizes.

#### Evaluation Metrics

We evaluate $\hat{Y}$ with both reference-based and execution-based evaluation. For reference-based evaluation, exact match (EM) and edit similarity (ES) are reported. Following Zhang et al. ([2023](https://arxiv.org/html/2403.10059v2#bib.bib40)), ES is defined as

|$$ES(\hat{Y},Y)=\frac{1-Lev(\hat{Y},Y)}{\max(|\hat{Y}|
|---|---|

where $Lev$ is the Levenshtein distance (Levenshtein et al., [1966](https://arxiv.org/html/2403.10059v2#bib.bib18)). We report $ES\times 100$ in all the tables following Zhang et al. ([2023](https://arxiv.org/html/2403.10059v2#bib.bib40)) for better readability. For execution-based evaluation, we report the unit test pass rate (UT). $\hat{Y}$ is said to pass the unit tests if replacing $Y$ with $\hat{Y}$ does not cause any unit test to fail. We implement simple post-processing procedures to handle common cases such as excessive lines in model’s outputs, which are documented in [Appendix A](https://arxiv.org/html/2403.10059v2#A1 "Appendix A Detailed RAG Execution Setup ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion").

#### Models

We experiment on two families of strong code LMs. CodeGen-Mono (Nijkamp et al., [2023b](https://arxiv.org/html/2403.10059v2#bib.bib24)) is pretrained sequentially in natural language, multilingual code, and a Python corpus. StarCoder and StarCoderBase (Li et al., [2023b](https://arxiv.org/html/2403.10059v2#bib.bib20)) are trained with fill-in-the-middle ability on a large corpus of multilingual code, GitHub issues, Git commits, and Jupyter notebooks. StarCoder is obtained by training StarCoderBase on an additional Python corpus.

## 5 Results

### 5.1 Is retrieval always helpful?

As a proof of concept, we first show that on a range of repository-level code completion tasks, the retrieved contexts often fail to improve code LMs’ generation quality.

In [Table 1](https://arxiv.org/html/2403.10059v2#S5.T1 "In 5.2 Repoformer achieves strong code completion performance via selective RAG ‣ 5 Results ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") and [Figure 3](https://arxiv.org/html/2403.10059v2#S4.F3 "In Evaluation Datasets ‣ 4.2 Evaluation Setup ‣ 4 Experimental Setup ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"), we evaluate four code LMs on function completion and API completion from RepoEval. For each model, we report the instance-level performance change from code completion only using $X_{l}$ and $X_{r}$ to retrieval-augmented code completion with $X_{l}$, $X_{r}$, and $CC$ (detailed prompts in [Appendix A](https://arxiv.org/html/2403.10059v2#A1 "Appendix A Detailed RAG Execution Setup ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")).

The results reveal an intriguing pattern: for repository-level code completion, the help from cross retrieval is often sparse. Specifically, retrieval improves LMs’ performance on only 20% or fewer instances. For more than 60% of the instances, retrieval augmentation does not affect the performance at all<sup>4</sup><sup>4</sup>4Upon a manual inspection, we find that most of the outputs in this category are also not changed by retrieval at all.. Finally, another 20% retrievals actually harm the performance, almost as often as the first case. The observed trends are consistent for both API and function completion and hold for both small-sized (1B and 2B) and moderate-to-large (around 16B) code LMs. The generality of this observation is further confirmed by an analysis of Repoformer’s training data, where we find that retrieval improves the performance for only fewer than 30% instances ([Appendix D](https://arxiv.org/html/2403.10059v2#A4 "Appendix D Data Creation for Repoformer Training and CrossCodeLongEval ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")). Together, these findings highlight the suboptimality of the always retrieving and augmenting the cross-file contexts and thus motivate our selective retrieval proposal.

### 5.2 Repoformer achieves strong code completion performance via selective RAG

|Model|Size|Performance (UT)|UT Change|
|---|---|---|---|
|$X_{l}$|$X_{l}$|$\downarrow$|$=$|$\uparrow$|
|CodeGen-Mono|16B|23.74|24.18|23|407|25|
|CodeGen-Mono|2B|30.55|32.51|18|400|37|
|StarCoder|16B|34.73|42.86|16|386|53|
|StarCoderBase|1B|22.20|25.71|16|407|32|

Table 1: The performance change on RepoEval function completion exhibited by four models from retrieved cross-file contexts. For the majority of the instances, RAG does not improve the performance. “$\uparrow$”, “$=$”, “$\downarrow$” denote the counts for performance increase, no performance change, and performance drop.

|RepoEval|CrossCodeLongEval|
|---|---|
|(Line)|(API)|(Function)|(Chunk)|(Function)|
|Size|Model|RAG Policy|EM|ES|EM|ES|UT|ES|EM|ES|ES|
|1B|StarCoderBase|No|43.44|67.77|37.81|66.54|22.20|47.65|31.08|60.09|47.49|
|Always|51.19|72.30|43.94|69.17|25.71|55.64|37.22|63.73|50.50|
|Repoformer|Selective<sub id="S5.T2.2.1.6.3.2.1">G</sub>|51.90|74.50|43.50|71.00|24.00|53.10|38.52|68.08|52.09|
|Selective<sub id="S5.T2.2.1.7.4.1.1">T</sub>|54.40|76.00|46.10|72.70|28.79|57.30|41.92|69.97|53.71|
|3B|StarCoderBase|No|49.00|72.12|40.44|69.02|24.84|51.22|36.14|64.65|49.88|
|Always|56.69|76.68|47.00|72.62|29.67|57.68|42.26|67.74|53.39|
|Repoformer|Selective<sub id="S5.T2.2.1.10.7.2.1">G</sub>|56.30|77.60|46.10|73.60|28.57|54.70|42.06|70.70|54.47|
|Selective<sub id="S5.T2.2.1.11.8.1.1">T</sub>|59.63|79.02|49.31|74.96|32.96|60.56|46.66|72.23|56.24|
|7B|StarCoderBase|No|51.88|74.03|43.31|70.79|25.49|52.28|38.88|66.61|52.45|
|Always|59.44|78.15|49.56|73.65|31.43|58.51|44.44|69.53|55.41|
|Repoformer|Selective<sub id="S5.T2.2.1.14.11.2.1">G</sub>|56.00|76.63|48.06|75.03|30.77|55.27|43.80|72.46|56.14|
|Selective<sub id="S5.T2.2.1.15.12.1.1">T</sub>|59.63|78.63|50.87|76.89|35.16|60.64|46.88|74.20|57.18|
|16B|StarCoder|No|55.25|76.07|44.50|71.00|34.73|53.60|42.58|69.40|54.20|
|Always|61.25|79.24|51.12|74.50|42.86|60.96|47.90|71.90|58.06|
|Repoformer|Selective<sub id="S5.T2.2.1.18.15.2.1">G</sub>|58.13|78.81|48.69|76.23|42.42|58.42|45.00|73.36|57.71|
|Selective<sub id="S5.T2.2.1.19.16.1.1">T</sub>|61.75|80.34|51.88|77.93|44.18|62.58|49.18|75.50|58.93|

Table 2: Experiment results on RepoEval and CrossCodeLongEval. The best performance among each model size is boldfaced. We use Selective<sub id="S5.T2.8.2.1">G</sub> and Selective<sub id="S5.T2.8.2.2">T</sub> to denote the greedy selection and the threshold selection strategy for selective retrieval. Repoformer greatly outperforms StarCoderBase of the same size while consuming a smaller retrieval budget. Among the two selective policies, threshold selection enables the best selective RAG performance.

Next, we evaluate the code completion performance of Repoformer. We compare the following three settings<sup>5</sup><sup>5</sup>5We do not consider iterative retrieval because we find that single-iteration RAG already achieves the majority of the performance gains from multi-iteration RAG.. For the first two baselines, we use the state-of-the-art single-iteration prompting pipeline (Zhang et al. ([2023](https://arxiv.org/html/2403.10059v2#bib.bib40)), detailed in [Appendix A](https://arxiv.org/html/2403.10059v2#A1 "Appendix A Detailed RAG Execution Setup ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")). We use StarCoder models due to their strong performance among the open-source code LMs.

1.  1.

    No Retrieval. This baseline only provides $X_{l}$ and $X_{r}$ to the model in the prompt.

2.  2.

    Always Retrieving. This baseline always augments $X_{l}$ and $X_{r}$ with the retrieved $CC$.

3.  3.

    Selective Retrieval. We provide Repoformer with $X_{l}$ and $X_{r}$ in the prompt, optionally augmented with $CC$ based on two selective RAG policies:

    -   •

        Greedy Selection. Retrieval is performed if <cc> is the most likely token following <eof>.

    -   •

        Threshold Selection. If the probability of <cc> following <eof> is greater than a threshold $T$, retrieval augmentation is performed<sup>6</sup><sup>6</sup>6We find that $T=0.15$ for function completion and $T=0.2$ for the other tasks generally work well. These two thresholds are always used unless otherwise stated..



The results are summarized in [Table 2](https://arxiv.org/html/2403.10059v2#S5.T2 "In 5.2 Repoformer achieves strong code completion performance via selective RAG ‣ 5 Results ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"). Compared to no retrieval and always retrieving with StarCoderBase of the same size, Repoformer’s selective retrieval strategy exhibits strong performance improvements across all the tasks and both lexical-based and execution-based metrics. Via the threshold selection strategy, Repoformer-3B can outperform StarCoderBase-7B on most of the tasks and metrics except EM for API completion, even outperforming the 5x larger StarCoder in terms of ES for API and chunk completion. Finally, The Repoformer-16B model outperforms the strongest StarCoder baseline by 3%, averaged across all tasks, setting up the new start-of-the-art for repository-level code completion. We also experimentally confirm that the performance improvement from our framework can generalize to three languages beyond Python ([Section E.2](https://arxiv.org/html/2403.10059v2#A5.SS2 "E.2 CrossCodeEval and Multilingual Repoformer ‣ Appendix E Extended Analyses ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")) as well as dense retrieval instead of Jaccard similarity ([Section E.3](https://arxiv.org/html/2403.10059v2#A5.SS3 "E.3 Repoformer’s Robustness to the Retriever Choice ‣ Appendix E Extended Analyses ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")). In later sections, we demonstrate that the observed success is due to both the ability to accurately abstain from retrieval and the improved robustness to retrieval.

In terms of code completion accuracy, the threshold selection strategy outperforms the greedy selection strategy on all the tasks. In the next section, we show that the two strategies represent different ways to achieve a good balance between accuracy and inference budget.

### 5.3 Repoformer improves inference efficiency

We illustrate the benefits of Repoformer for saving the inference latency in a realistic “online serving” setting.

#### Latency Model

We assume that indexing has already been done for the working repository. Given a code completion request containing the current file $(X_{l},X_{r})$, the system issues three processes at the same time:

-   •

    $P_{1}$: make a retrieval decision using Repoformer.

-   •

    $P_{2}$: use a code LM $\mathcal{M}$ to generate $\hat{Y}$ without $CC$.

-   •

    $P_{3}$: retrieve $CC$ and generate $\hat{Y}$ with $CC$ using $\mathcal{M}$.


Depending on the result of $P_{1}$, the system waits for either $P_{2}$ or $P_{3}$ and ignores the other process. If $\mathcal{M}$ is Repoformer, $P_{1}$ can be merged with $P_{2}$ by forcing $\mathcal{M}$ to generate a hypothesis without $CC$ after collecting the retrieval decision. We consider three latency terms: (1) $T_{d}$, time required for the retrieval decision, (2) $T_{r}$, the retrieval latency, and (3) $T_{g}$, the generation latency. Then, the latency for $P_{1}$, $P_{2}$, and $P_{3}$ are $T_{d}$, $T_{g}$, and $T_{r}+T_{g}$. When $\mathcal{M}$ is Repoformer or a model larger than Repoformer, we have $T_{d}<T_{g}<T_{r}+T_{g}$. Therefore, the latency of the entire system is $T_{g}$ or $T_{r}+T_{g}$ depending on $P_{1}$. Using this latency model, we benchmark the latency of various selective retrieval settings on RepoEval with the vllm library (Kwon et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib17)) on a single Nvidia A100 GPU (80G).

|RAG Policy|API Completion|Line Completion|
|---|---|---|
|ES|%RAG|SU|ES|%RAG|SU|
|Always|72.02|$100\%$|$0\%$|75.91|$100\%$|0%|
|Selective<sub id="S5.T3.7.7.7.6.1">G</sub>|71.04|$18\%$|$69\%$|74.50|$19\%$|$61\%$|
|1B|Selective<sub id="S5.T3.11.11.11.6.1">T</sub>|72.72|$61\%$|$28\%$|76.00|$62\%$|$27\%$|
|Always|74.66|$100\%$|$0\%$|78.68|$100\%$|0%|
|Selective<sub id="S5.T3.18.18.18.6.1">G</sub>|73.60|$19\%$|$46\%$|77.60|$20\%$|$43\%$|
|3B|Selective<sub id="S5.T3.22.22.22.6.1">T</sub>|74.96|$78\%$|$17\%$|79.02|$74\%$|$16\%$|

Table 3: RAG latency of Repoformer with two self-selective RAG paradigms. %RAG = ratio of instances where RAG is performed. SU = Speedup compared to always retrieving (the higher, the better). Compared to the always retrieving baseline, the threshold selection strategy consistently demonstrates gains in both accuracy and latency. The greedy selection strategy shows much larger latency gains with a small performance degradation.

First, we consider $\mathcal{M}$ = Repoformer and present the results in [Table 3](https://arxiv.org/html/2403.10059v2#S5.T3 "In Latency Model ‣ 5.3 Repoformer improves inference efficiency ‣ 5 Results ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"). Line and API completion are presented to cover short and moderate target lengths<sup>7</sup><sup>7</sup>7We omit the function completion results as RepoEval uses very small repositories for function completion for easier unit testing.. Both selective strategies significantly improve the latency, with a different trade-off: threshold selection results in improvements for both accuracy and latency compared to always retrieving, while using greedy selection results in a larger latency gain with a minor performance degradation (around 1.0 ES). It is worth mentioning that the latency improvement from selective RAG could be further enhanced with a more advanced retrieval setup. For instance, conducting dense retrieval on large repositories often consumes more than 80% of the entire RAG pipeline’s latency. Then, a 20% RAG policy could translate into more than 70% speedup. We empirically verify this statement in [Section E.3](https://arxiv.org/html/2403.10059v2#A5.SS3 "E.3 Repoformer’s Robustness to the Retriever Choice ‣ Appendix E Extended Analyses ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion").

Next, we consider using diverse larger LMs as $\mathcal{M}$ in the code completion framework and using selection<sub id="S5.SS3.SSS0.Px1.p4.1.1">T</sub> with Repoformer-1B as a plug-and-play selective RAG policy to decide whether retrieval should be performed. We experiment on a diverse set of LMs: StarCoderBase, Code Llama (Roziere et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib30))<sup>8</sup><sup>8</sup>8We accessed the model through Amazon SageMaker ([https://docs.aws.amazon.com/sagemaker/](https://docs.aws.amazon.com/sagemaker/))., CodeGen25 (Nijkamp et al., [2023a](https://arxiv.org/html/2403.10059v2#bib.bib23)), and ChatGPT<sup>9</sup><sup>9</sup>9We use gpt-3.5-turbo-0613 via the OpenAI API.. As shown in [Table 4](https://arxiv.org/html/2403.10059v2#S5.T4 "In Latency Model ‣ 5.3 Repoformer improves inference efficiency ‣ 5 Results ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"), the selective predictions from Repoformer-1B successfully reduce the inference latency with different larger LMs by approximately 25% while improving their accuracy. Collectively, the findings indicate that Repoformer has acquired robust selective retrieval capabilities that could generalize to diverse types of code LMs.

|Model|RAG Policy|API Completion|Line Completion|
|---|---|---|---|
|ES|SU|ES|SU|
|Always Retrieving|73.65|$0\%$|78.15|$0\%$|
|SCB-7B|Repoformer-1B|74.10|$24\%$|78.31|$25\%$|
|Always Retrieving|74.50|$0\%$|79.24|$0\%$|
|SCB-16B|Repoformer-1B|74.84|$24\%$|79.48|$24\%$|
|Always Retrieving|63.07|0%|68.42|0%|
|CG25-7B|Repoformer-1B|63.37|20%|68.86|29%|
|Always Retrieving|58.75|0%|59.99|0%|
|CL-7B|Repoformer-1B|58.91|25%|60.47|28%|
|Always Retrieving|61.08|0%|61.58|0%|
|CL-16B|Repoformer-1B|62.10|32%|62.45|30%|
|Always Retrieving|63.38|0%|61.76|0%|
|ChatGPT|Repoformer-1B|64.01|28%|61.92|18%|

Table 4: Accuracy and latency of larger code LMs as the generation model and with Repoformer-1B as the policy model for selective RAG. SCB = StarCoderBase, CG25 = CodeGen25, CL = Code Llama. SU = Speedup compared to Always Retrieving (the higher, the better). Compared to the Always Retrieving baseline, Repoformer’s selective decisions improve both the accuracy and latency of these larger LMs.

## 6 Analysis

In this section, we present further analyses and ablation studies on Repoformer-1B.

#### Is Repoformer sensitive to threshold settings?

In [Figure 4](https://arxiv.org/html/2403.10059v2#S6.F4 "In Does Repoformer make accurate and calibrated selective retrieval decisions? ‣ 6 Analysis ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"), we present the code completion accuracy and latency of Repoformer as a function of the threshold. As the threshold increases, the model’s code completion performance first increases due to avoiding potentially harmful retrievals. At threshold 0.4, the model still maintains similar performance compared to always retrieving, with latency reduced by 50%. This result demonstrates that Repoformer can accommodate various threshold settings and provide a good accuracy-latency trade-off. We provide the visualization for other tasks in [Section E.4](https://arxiv.org/html/2403.10059v2#A5.SS4 "E.4 Full Latency-Accuracy Visualizations ‣ Appendix E Extended Analyses ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion").

#### Does Repoformer make accurate and calibrated selective retrieval decisions?

In [Figure 5](https://arxiv.org/html/2403.10059v2#S6.F5 "In Does Repoformer make accurate and calibrated selective retrieval decisions? ‣ 6 Analysis ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"), we evaluate the precision of retrieval abstention decisions made by Repoformer’s threshold selection strategy. We find that the abstentions are accurate for over 80% instances across all the tasks: when Repoformer abstains from retrieval, its code completion prediction either is already correct without retrieval or cannot be improved by retrieval. We also evaluate the calibration of the selective decisions and find Repoformer generally making near-calibrated predictions for line and API completion while the calibration is suboptimal for function completion with UT employed as the metric ([Section E.1](https://arxiv.org/html/2403.10059v2#A5.SS1 "E.1 Calibration of Repoformer’s Selective Retrieval Prediction ‣ Appendix E Extended Analyses ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")). We hypothesize that this could be caused by using ES to create the training signal and encourage future work to devise methods for labeling the quality of function completion more effectively.

![Refer to caption](https://arxiv.org/html/2403.10059v2/x4.png)

Figure 4: The accuracy and latency change with different threshold settings. Selective Retrieval with Repoformer achieves better accuracy and better latency than always retrieving. In addition, this behavior is relatively insensitive to the threshold.

![Refer to caption](https://arxiv.org/html/2403.10059v2/x5.png)

Figure 5: An analysis of the instances where Repoformer-1B abstains from retrieval. We divide the instances into (1) the model answering correctly without retrieval (dark blue), the model making a mistake that cannot be improved by retrieval (light blue), and the model achieving better performance when retrieval is performed (red). The precision of abstention is over 0.8 on all tasks except for Function (RepoEval), which has a precision of 0.78.

![Refer to caption](https://arxiv.org/html/2403.10059v2/x6.png)

Figure 6: The performance change on RepoEval from retrieved cross-file context for the instances where Repoformer self-selects retrieval. Compared to StarCoderBase, Repoformer is better at leveraging $CC$ to improve the generation quality.

#### Is Repoformer robust to retrieval?

In Figure [6](https://arxiv.org/html/2403.10059v2#S6.F6 "Figure 6 ‣ Does Repoformer make accurate and calibrated selective retrieval decisions? ‣ 6 Analysis ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"), we show the performance change caused by $CC$ on the instances where Repoformer requests for retrieval. Compared to StarCoderBase, Repoformer exhibits more and greater performance gains upon observing $CC$. The number of performance decreases is also significantly reduced, indicating an improved robustness to the potentially irrelevant retrieval contexts. In [Table 8](https://arxiv.org/html/2403.10059v2#A5.T8 "In E.3 Repoformer’s Robustness to the Retriever Choice ‣ Appendix E Extended Analyses ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") in the appendix, we further study the effect of using dense retrieval. Although dense retrieval returns an arguably different context distribution compared to sparse retrieval, Repoformer still exhibits strong improvements in both quality and latency.

#### Ablation Study

We study several alternative designs:

-   •

    (A1) Combining $\mathcal{L}_{eval}$ and $\mathcal{L}_{gen}$ as a single cross-entropy loss. In general, this down-weights $\mathcal{L}_{eval}$.

-   •

    (A2) Removing the self-evaluation loss $\mathcal{L}_{eval}$.

-   •

    (A3) Further removing all the $CC$ from A2. This amounts to only training on fill-in-the-middle.

-   •

    (A4) Placing <cc> and $CC$ after <fim\_middle> and marking its end with a new token <cc\_end>. A4 mainly studies whether it is more beneficial to train the LM to treat $CC$ as context fetched during fill-in-middle generation instead of part of the input context.


We fine-tune StarCoderBase-1B with the same setup as Repoformer and present the results on CCEval in [Table 5](https://arxiv.org/html/2403.10059v2#S6.T5 "In Ablation Study ‣ 6 Analysis ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"). Although A1 has slightly better RAG performance, it fails to make meaningful selective decisions due to $\mathcal{L}_{eval}$ being outweighed by $\mathcal{L}_{gen}$ in long sequences: the probability of <cc> is almost always 1. For A2, we find it only slightly outperforms Repoformer, suggesting learning $\mathcal{L}_{eval}$ does not harm the RAG ability a lot while bringing in the strong selective retrieval ability, which in turn boosts both accuracy and latency. A3 has the same performance for in-file completion as Repoformer, but exhibits worse RAG performance, indicating the necessity of training with $CC$. Finally, A4 achieves reasonable chunk completion performance but performs much worse in function completion. We hypothesize that placing $CC$ within the infilling part is detrimental due to breaking the fill-in-the-middle semantics learned in StarCoder pre-training.

|Model|RAG Policy|Chunk Completion|Function Completion|
|---|---|---|---|
|T|%RAG|ES|T|%RAG|ES|
|SC|No|\-|0%|60.09|\-|0%|47.49|
|Always|\-|100%|63.73|\-|100%|50.50|
|RF|No|\-|0%|66.22|\-|0%|49.77|
|Selective<sub id="S6.T5.2.1.6.6.1.1">T</sub>|0.20|75%|69.97|0.15|76%|53.71|
|Always|\-|100%|69.95|\-|100%|53.56|
|A1|No|\-|0%|66.14|\-|0%|49.25|
|Selective<sub id="S6.T5.2.1.9.9.1.1">T</sub>|0.99|100%|70.21|0.99|100%|53.93|
|Always|\-|100%|70.21|\-|100%|53.93|
|A2|No|\-|0%|66.49|\-|0%|49.02|
|Always|\-|100%|70.45|\-|100%|53.90|
|A3|No|\-|0%|66.25|\-|0%|49.01|
|Always|\-|100%|68.85|\-|100%|52.12|
|A4|No|\-|0%|64.96|\-|0%|25.44|
|Selective<sub id="S6.T5.2.1.16.16.1.1">T</sub>|0.10|86%|69.35|0.10|83%|26.50|
|Always|\-|100%|69.19|\-|100%|26.35|

Table 5: Ablation study results. We report the performance on two tasks from the CCEval dataset. SC = StarCoderBase-1B. RF = Repoformer-1B. T = threshold for the Selective<sub id="S6.T5.10.2.5">T</sub> policy. We found T = 0.10 works better for A4 and thus applied it to all the A4 results. %RAG = ratio of instances where RAG is performed.

## 7 Conclusion

In this paper, we challenge the common assumption of always performing retrieval for RAG-based repository-level code completion. In response, we propose a selective retrieval augmentation framework powered by Repoformer, a code LM that identifies whether cross-file context is necessary, and self-triggers retrieval. Extensive evaluations demonstrate our approach’s effectiveness in enhancing accuracy while significantly reducing latency, showcasing its potential in practical coding environments.

#### Discussion

Building upon Repoformer, future research may consider several important directions:

1.  1.

    Further speeding up large LMs. Beyond as a selective retrieval policy, Repoformer has the potential to serve as an effective plug-in draft model in settings such as speculative decoding (Chen et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib3)).

2.  2.

    More effective function completion. To enable a good scalability, we used lexical similarity as the signal for training label creation. Although this heuristics enables improvements in function completion evaluation, designing a more accurate and scalable labeling approach is an important future direction.

3.  3.

    Personalized retrieval. We apply a uniform selective policy across repositories. However, certain repositories could be inherently more RAG-friendly by exhibiting a higher level of duplication (Zhang et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib40)). Adapting the selective RAG paradigm towards accurate personalized policies is an important direction.


## Acknowledgement

We express our gratitude to anonymous reviewers for their valuable suggestions to improve the quality of the paper. The authors also thank Amita Kamath and Po-Nien Kung for their constructive feedback provided during the paper writing process. Additionally, we would like to express gratitude to some other team members from Amazon CodeWhisperer and UCLANLP for their insightful discussions, which have contributed to the refinement of our work.

## Impact Statement

Our research introduces a novel approach to repository-level code completion that significantly enhances efficiency and accuracy by employing selective retrieval, reducing unnecessary computational waste, and contributing to more sustainable software development practices. Although promising in streamlining development workflows and potentially applicable in various domains, it is important to consider the implications of increased automation in software development, programming education, and the potential for inadvertent biases. Ensuring the ethical use and ongoing evaluation of such code automation technologies is crucial to maximize their societal benefits while mitigating risks. In addition, as a general infrastructure, it is important to design additional mechanisms that prevent RAG systems from revealing sensitive data in the retrieval database.

In this work, we mainly rely on open-sourced, permissively-licensed repositories (the Stack, CrossCodeEval) and models (StarCoder, CodeGen) to perform the experiments. However, as mentioned by Ding et al. ([2023](https://arxiv.org/html/2403.10059v2#bib.bib5)), some of the repositories of RepoEval are with non-permissive licenses. We rely on the dataset and code distributed by the original RepoEval authors to perform the experiment and do not re-distribute the dataset or adapt it for other purposes.

## References

-   Asai et al. (2024) Asai, A., Wu, Z., Wang, Y., Sil, A., and Hajishirzi, H. Self-RAG: Learning to retrieve, generate, and critique through self-reflection. In _The Twelfth International Conference on Learning Representations_, 2024. URL [https://openreview.net/forum?id=hSyW5go0v8](https://openreview.net/forum?id=hSyW5go0v8).
-   Bavarian et al. (2022) Bavarian, M., Jun, H., Tezak, N., Schulman, J., McLeavey, C., Tworek, J., and Chen, M. Efficient training of language models to fill in the middle. _ArXiv preprint_, abs/2207.14255, 2022. URL [https://arxiv.org/abs/2207.14255](https://arxiv.org/abs/2207.14255).
-   Chen et al. (2023) Chen, C., Borgeaud, S., Irving, G., Lespiau, J.-B., Sifre, L., and Jumper, J. Accelerating large language model decoding with speculative sampling. _ArXiv preprint_, abs/2302.01318, 2023. URL [https://arxiv.org/abs/2302.01318](https://arxiv.org/abs/2302.01318).
-   Chen et al. (2021) Chen, M., Tworek, J., Jun, H., Yuan, Q., Pinto, H. P. d. O., Kaplan, J., Edwards, H., Burda, Y., Joseph, N., Brockman, G., et al. Evaluating large language models trained on code. _ArXiv preprint_, abs/2107.03374, 2021. URL [https://arxiv.org/abs/2107.03374](https://arxiv.org/abs/2107.03374).
-   Ding et al. (2023) Ding, Y., Wang, Z., Ahmad, W. U., Ding, H., Tan, M., Jain, N., Ramanathan, M. K., Nallapati, R., Bhatia, P., Roth, D., and Xiang, B. Crosscodeeval: A diverse and multilingual benchmark for cross-file code completion. In _Thirty-seventh Conference on Neural Information Processing Systems Datasets and Benchmarks Track_, 2023. URL [https://arxiv.org/abs/2310.11248](https://arxiv.org/abs/2310.11248).
-   Ding et al. (2024) Ding, Y., Wang, Z., Ahmad, W. U., Ramanathan, M. K., Nallapati, R., Bhatia, P., Roth, D., and Xiang, B. CoCoMIC: Code completion by jointly modeling in-file and cross-file context. pp.  3433–3445, May 2024. URL [https://aclanthology.org/2024.lrec-main.305](https://aclanthology.org/2024.lrec-main.305).
-   Drozdov et al. (2022) Drozdov, A., Wang, S., Rahimi, R., McCallum, A., Zamani, H., and Iyyer, M. You can’t pick your neighbors, or can you? when and how to rely on retrieval in the kNN-LM. In _Findings of the Association for Computational Linguistics: EMNLP 2022_, pp.  2997–3007, Abu Dhabi, United Arab Emirates, December 2022. Association for Computational Linguistics. doi: 10.18653/v1/2022.findings-emnlp.218. URL [https://aclanthology.org/2022.findings-emnlp.218](https://aclanthology.org/2022.findings-emnlp.218).
-   Guo et al. (2022) Guo, D., Lu, S., Duan, N., Wang, Y., Zhou, M., and Yin, J. UniXcoder: Unified cross-modal pre-training for code representation. In _Proceedings of the 60th Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)_, pp.  7212–7225, Dublin, Ireland, May 2022. Association for Computational Linguistics. doi: 10.18653/v1/2022.acl-long.499. URL [https://aclanthology.org/2022.acl-long.499](https://aclanthology.org/2022.acl-long.499).
-   He et al. (2021) He, J., Neubig, G., and Berg-Kirkpatrick, T. Efficient nearest neighbor language models. In _Proceedings of the 2021 Conference on Empirical Methods in Natural Language Processing_, pp.  5703–5714, Online and Punta Cana, Dominican Republic, November 2021. Association for Computational Linguistics. doi: 10.18653/v1/2021.emnlp-main.461. URL [https://aclanthology.org/2021.emnlp-main.461](https://aclanthology.org/2021.emnlp-main.461).
-   Hellendoorn & Devanbu (2017) Hellendoorn, V. J. and Devanbu, P. T. Are deep neural networks the best choice for modeling source code? In Bodden, E., Schäfer, W., van Deursen, A., and Zisman, A. (eds.), _Proceedings of the 2017 11th Joint Meeting on Foundations of Software Engineering, ESEC/FSE 2017, Paderborn, Germany, September 4-8, 2017_, pp.  763–773. ACM, 2017. doi: 10.1145/3106237.3106290. URL [https://doi.org/10.1145/3106237.3106290](https://doi.org/10.1145/3106237.3106290).
-   Hill & Rideout (2004) Hill, R. and Rideout, J. Automatic method completion. In _19th IEEE International Conference on Automated Software Engineering (ASE 2004), 20-25 September 2004, Linz, Austria_, pp.  228–235. IEEE Computer Society, 2004. doi: 10.1109/ASE.2004.10034. URL [https://doi.ieeecomputersociety.org/10.1109/ASE.2004.10034](https://doi.ieeecomputersociety.org/10.1109/ASE.2004.10034).
-   Jaccard (1912) Jaccard, P. The distribution of the flora in the alpine zone.1. _New Phytologist_, 11:37–50, 1912. URL [https://api.semanticscholar.org/CorpusID:85574559](https://api.semanticscholar.org/CorpusID:85574559).
-   Jain et al. (2023) Jain, N., Zhang, D., Ahmad, W. U., Wang, Z., Nan, F., Li, X., Tan, M., Nallapati, R., Ray, B., Bhatia, P., Ma, X., and Xiang, B. ContraCLM: Contrastive learning for causal language model. In _Proceedings of the 61st Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)_, pp.  6436–6459, Toronto, Canada, July 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.acl-long.355. URL [https://aclanthology.org/2023.acl-long.355](https://aclanthology.org/2023.acl-long.355).
-   Jiang et al. (2023) Jiang, Z., Xu, F., Gao, L., Sun, Z., Liu, Q., Dwivedi-Yu, J., Yang, Y., Callan, J., and Neubig, G. Active retrieval augmented generation. In Bouamor, H., Pino, J., and Bali, K. (eds.), _Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing_, pp.  7969–7992, Singapore, December 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.emnlp-main.495. URL [https://aclanthology.org/2023.emnlp-main.495](https://aclanthology.org/2023.emnlp-main.495).
-   Kadavath et al. (2022) Kadavath, S., Conerly, T., Askell, A., Henighan, T., Drain, D., Perez, E., Schiefer, N., Hatfield-Dodds, Z., DasSarma, N., Tran-Johnson, E., et al. Language models (mostly) know what they know. _ArXiv preprint_, abs/2207.05221, 2022. URL [https://arxiv.org/abs/2207.05221](https://arxiv.org/abs/2207.05221).
-   Kocetkov et al. (2022) Kocetkov, D., Li, R., Allal, L. B., Li, J., Mou, C., Ferrandis, C. M., Jernite, Y., Mitchell, M., Hughes, S., Wolf, T., et al. The stack: 3 tb of permissively licensed source code. _ArXiv preprint_, abs/2211.15533, 2022. URL [https://arxiv.org/abs/2211.15533](https://arxiv.org/abs/2211.15533).
-   Kwon et al. (2023) Kwon, W., Li, Z., Zhuang, S., Sheng, Y., Zheng, L., Yu, C. H., Gonzalez, J., Zhang, H., and Stoica, I. Efficient memory management for large language model serving with pagedattention. In Flinn, J., Seltzer, M. I., Druschel, P., Kaufmann, A., and Mace, J. (eds.), _Proceedings of the 29th Symposium on Operating Systems Principles, SOSP 2023, Koblenz, Germany, October 23-26, 2023_, pp.  611–626. ACM, 2023. doi: 10.1145/3600006.3613165. URL [https://doi.org/10.1145/3600006.3613165](https://doi.org/10.1145/3600006.3613165).
-   Levenshtein et al. (1966) Levenshtein, V. I. et al. Binary codes capable of correcting deletions, insertions, and reversals. In _Soviet physics doklady_, volume 10, pp.  707–710. Soviet Union, 1966. URL [https://nymity.ch/sybilhunting/pdf/Levenshtein1966a.pdf](https://nymity.ch/sybilhunting/pdf/Levenshtein1966a.pdf).
-   Li et al. (2023a) Li, J., Tang, T., Zhao, W. X., Wang, J., Nie, J.-Y., and Wen, J.-R. The web can be your oyster for improving language models. In _Findings of the Association for Computational Linguistics: ACL 2023_, pp.  728–746, Toronto, Canada, July 2023a. Association for Computational Linguistics. doi: 10.18653/v1/2023.findings-acl.46. URL [https://aclanthology.org/2023.findings-acl.46](https://aclanthology.org/2023.findings-acl.46).
-   Li et al. (2023b) Li, R., Allal, L. B., Zi, Y., Muennighoff, N., Kocetkov, D., Mou, C., Marone, M., Akiki, C., Li, J., Chim, J., Liu, Q., Zheltonozhskii, E., Zhuo, T. Y., Wang, T., Dehaene, O., Davaadorj, M., Lamy-Poirier, J., Monteiro, J., Shliazhko, O., Gontier, N., Meade, N., Zebaze, A., Yee, M., Umapathi, L. K., Zhu, J., Lipkin, B., Oblokulov, M., Wang, Z., V, R. M., Stillerman, J., Patel, S. S., Abulkhanov, D., Zocca, M., Dey, M., Zhang, Z., Moustafa-Fahmy, N., Bhattacharyya, U., Yu, W., Singh, S., Luccioni, S., Villegas, P., Kunakov, M., Zhdanov, F., Romero, M., Lee, T., Timor, N., Ding, J., Schlesinger, C., Schoelkopf, H., Ebert, J., Dao, T., Mishra, M., Gu, A., Robinson, J., Anderson, C. J., Dolan-Gavitt, B., Contractor, D., Reddy, S., Fried, D., Bahdanau, D., Jernite, Y., Ferrandis, C. M., Hughes, S., Wolf, T., Guha, A., von Werra, L., and de Vries, H. Starcoder: may the source be with you!, 2023b. URL [https://doi.org/10.48550/arXiv.2305.06161](https://doi.org/10.48550/arXiv.2305.06161).
-   Lu et al. (2022) Lu, S., Duan, N., Han, H., Guo, D., Hwang, S.-w., and Svyatkovskiy, A. ReACC: A retrieval-augmented code completion framework. In _Proceedings of the 60th Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)_, pp.  6227–6240, Dublin, Ireland, May 2022. Association for Computational Linguistics. doi: 10.18653/v1/2022.acl-long.431. URL [https://aclanthology.org/2022.acl-long.431](https://aclanthology.org/2022.acl-long.431).
-   Mallen et al. (2023) Mallen, A., Asai, A., Zhong, V., Das, R., Khashabi, D., and Hajishirzi, H. When not to trust language models: Investigating effectiveness of parametric and non-parametric memories. In _Proceedings of the 61st Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)_, pp.  9802–9822, Toronto, Canada, July 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.acl-long.546. URL [https://aclanthology.org/2023.acl-long.546](https://aclanthology.org/2023.acl-long.546).
-   Nijkamp et al. (2023a) Nijkamp, E., Hayashi, H., Xiong, C., Savarese, S., and Zhou, Y. Codegen2: Lessons for training llms on programming and natural languages. _ICLR_, 2023a. URL [https://arxiv.org/abs/2305.02309](https://arxiv.org/abs/2305.02309).
-   Nijkamp et al. (2023b) Nijkamp, E., Pang, B., Hayashi, H., Tu, L., Wang, H., Zhou, Y., Savarese, S., and Xiong, C. Codegen: An open large language model for code with multi-turn program synthesis. In _The Eleventh International Conference on Learning Representations, ICLR 2023, Kigali, Rwanda, May 1-5, 2023_. OpenReview.net, 2023b. URL [https://openreview.net/pdf?id=iaYcJKpY2B\_](https://openreview.net/pdf?id=iaYcJKpY2B_).
-   OpenAI (2023) OpenAI. Gpt-4 technical report. _ArXiv preprint_, abs/2303.08774, 2023. URL [https://arxiv.org/abs/2303.08774](https://arxiv.org/abs/2303.08774).
-   Parnas (1972) Parnas, D. L. On the criteria to be used in decomposing systems into modules. _Commun. ACM_, 15(12):1053–1058, 1972. doi: 10.1145/361598.361623. URL [https://doi.org/10.1145/361598.361623](https://doi.org/10.1145/361598.361623).
-   Pei et al. (2023) Pei, H., Zhao, J., Lausen, L., Zha, S., and Karypis, G. Better context makes better code language models: A case study on function call argument completion. _Proceedings of the AAAI Conference on Artificial Intelligence_, 37(4):5230–5238, Jun. 2023. doi: 10.1609/aaai.v37i4.25653. URL [https://ojs.aaai.org/index.php/AAAI/article/view/25653](https://ojs.aaai.org/index.php/AAAI/article/view/25653).
-   Ram et al. (2023) Ram, O., Levine, Y., Dalmedigos, I., Muhlgay, D., Shashua, A., Leyton-Brown, K., and Shoham, Y. In-context retrieval-augmented language models. _Transactions of the Association for Computational Linguistics_, 11:1316–1331, 2023. doi: 10.1162/tacl˙a˙00605. URL [https://aclanthology.org/2023.tacl-1.75](https://aclanthology.org/2023.tacl-1.75).
-   Ren et al. (2020) Ren, S., Guo, D., Lu, S., Zhou, L., Liu, S., Tang, D., Sundaresan, N., Zhou, M., Blanco, A., and Ma, S. Codebleu: a method for automatic evaluation of code synthesis. _ArXiv preprint_, abs/2009.10297, 2020. URL [https://arxiv.org/abs/2009.10297](https://arxiv.org/abs/2009.10297).
-   Roziere et al. (2023) Roziere, B., Gehring, J., Gloeckle, F., Sootla, S., Gat, I., Tan, X. E., Adi, Y., Liu, J., Remez, T., Rapin, J., et al. Code llama: Open foundation models for code. _ArXiv preprint_, abs/2308.12950, 2023. URL [https://arxiv.org/abs/2308.12950](https://arxiv.org/abs/2308.12950).
-   Shi et al. (2023) Shi, W., Min, S., Yasunaga, M., Seo, M., James, R., Lewis, M., Zettlemoyer, L., and Yih, W.-t. Replug: Retrieval-augmented black-box language models. _ArXiv preprint_, abs/2301.12652, 2023. URL [https://arxiv.org/abs/2301.12652](https://arxiv.org/abs/2301.12652).
-   Shrivastava et al. (2023a) Shrivastava, D., Kocetkov, D., de Vries, H., Bahdanau, D., and Scholak, T. Repofusion: Training code models to understand your repository. _ArXiv preprint_, abs/2306.10998, 2023a. URL [https://arxiv.org/abs/2306.10998](https://arxiv.org/abs/2306.10998).
-   Shrivastava et al. (2023b) Shrivastava, D., Larochelle, H., and Tarlow, D. Repository-level prompt generation for large language models of code. In Krause, A., Brunskill, E., Cho, K., Engelhardt, B., Sabato, S., and Scarlett, J. (eds.), _International Conference on Machine Learning, ICML 2023, 23-29 July 2023, Honolulu, Hawaii, USA_, volume 202 of _Proceedings of Machine Learning Research_, pp.  31693–31715. PMLR, 2023b. URL [https://proceedings.mlr.press/v202/shrivastava23a.html](https://proceedings.mlr.press/v202/shrivastava23a.html).
-   Svyatkovskiy et al. (2020) Svyatkovskiy, A., Deng, S. K., Fu, S., and Sundaresan, N. Intellicode compose: code generation using transformer. In Devanbu, P., Cohen, M. B., and Zimmermann, T. (eds.), _ESEC/FSE ’20: 28th ACM Joint European Software Engineering Conference and Symposium on the Foundations of Software Engineering, Virtual Event, USA, November 8-13, 2020_, pp.  1433–1443. ACM, 2020. doi: 10.1145/3368089.3417058. URL [https://doi.org/10.1145/3368089.3417058](https://doi.org/10.1145/3368089.3417058).
-   Tu et al. (2014) Tu, Z., Su, Z., and Devanbu, P. T. On the localness of software. In Cheung, S., Orso, A., and Storey, M. D. (eds.), _Proceedings of the 22nd ACM SIGSOFT International Symposium on Foundations of Software Engineering, (FSE-22), Hong Kong, China, November 16 - 22, 2014_, pp.  269–280. ACM, 2014. doi: 10.1145/2635868.2635875. URL [https://doi.org/10.1145/2635868.2635875](https://doi.org/10.1145/2635868.2635875).
-   Wang et al. (2021) Wang, Y., Shi, E., Du, L., Yang, X., Hu, Y., Han, S., Zhang, H., and Zhang, D. Cocosum: Contextual code summarization with multi-relational graph neural network. _ArXiv preprint_, abs/2107.01933, 2021. URL [https://arxiv.org/abs/2107.01933](https://arxiv.org/abs/2107.01933).
-   Wang et al. (2023) Wang, Y., Li, P., Sun, M., and Liu, Y. Self-knowledge guided retrieval augmentation for large language models. In Bouamor, H., Pino, J., and Bali, K. (eds.), _Findings of the Association for Computational Linguistics: EMNLP 2023_, pp.  10303–10315, Singapore, 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.findings-emnlp.691. URL [https://aclanthology.org/2023.findings-emnlp.691](https://aclanthology.org/2023.findings-emnlp.691).
-   Ye & Fischer (2002) Ye, Y. and Fischer, G. Supporting reuse by delivering task-relevant and personalized information. In Tracz, W., Young, M., and Magee, J. (eds.), _Proceedings of the 24th International Conference on Software Engineering, ICSE 2002, 19-25 May 2002, Orlando, Florida, USA_, pp.  513–523. ACM, 2002. doi: 10.1145/581339.581402. URL [https://doi.org/10.1145/581339.581402](https://doi.org/10.1145/581339.581402).
-   Zan et al. (2022) Zan, D., Chen, B., Lin, Z., Guan, B., Yongji, W., and Lou, J.-G. When language model meets private library. In _Findings of the Association for Computational Linguistics: EMNLP 2022_, pp.  277–288, Abu Dhabi, United Arab Emirates, December 2022. Association for Computational Linguistics. doi: 10.18653/v1/2022.findings-emnlp.21. URL [https://aclanthology.org/2022.findings-emnlp.21](https://aclanthology.org/2022.findings-emnlp.21).
-   Zhang et al. (2023) Zhang, F., Chen, B., Zhang, Y., Keung, J., Liu, J., Zan, D., Mao, Y., Lou, J.-G., and Chen, W. RepoCoder: Repository-level code completion through iterative retrieval and generation. In Bouamor, H., Pino, J., and Bali, K. (eds.), _Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing_, pp.  2471–2484, Singapore, 2023. Association for Computational Linguistics. doi: 10.18653/v1/2023.emnlp-main.151. URL [https://aclanthology.org/2023.emnlp-main.151](https://aclanthology.org/2023.emnlp-main.151).
-   Zhou et al. (2023) Zhou, S., Alon, U., Xu, F. F., Jiang, Z., and Neubig, G. Docprompting: Generating code by retrieving the docs. In _The Eleventh International Conference on Learning Representations, ICLR 2023, Kigali, Rwanda, May 1-5, 2023_. OpenReview.net, 2023. URL [https://openreview.net/pdf?id=ZTCxT2t2Ru](https://openreview.net/pdf?id=ZTCxT2t2Ru).

## Appendix A Detailed RAG Execution Setup

Below, we describe the four steps we follow for executing RAG as well as the related hyperparameters.

1.  1.

    Indexing. All files in $F$ are divided into fix-sized code chunks with a sliding window. We set the chunk size to 20 for line, API, and chunk completion and set 50 for function completion. We use half of the chunk size as the stride size. Despite the duplication caused by the overlap between adjacent chunks, this design improves retrieval accuracy with tolerable cost, as the number of files is limited in a repository compared to large open-domain code corpora.

2.  2.

    Query Formation. A query is constructed based on $X_{l}$. We always use a fixed number of lines at the end of $X_{l}$ (i.e., immediately preceding $Y$) as the query. The query contains the same number of lines as the chunks in the index.

3.  3.

    Retrieval. A similarity function $f$ is used to compare the query with every chunk and identify $k$ most similar code chunks. We use $k=10$ and Jaccard similarity (Jaccard, [1912](https://arxiv.org/html/2403.10059v2#bib.bib12)) for $f$ for the main results. Fragment alignment (Lu et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib21)) is then applied: for each of the $k$ most similar code chunks, the chunk immediately following is included in $CC$ instead of the original chunk. We explored other choices mentioned in [Figure 8](https://arxiv.org/html/2403.10059v2#A3.F8 "In C.1 Trial Retrieval ‣ Appendix C Trial Retrieval and Trial Generation ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") such as cosine similarity with UniXCoder (Guo et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib8)) or CodeBLEU (Ren et al., [2020](https://arxiv.org/html/2403.10059v2#bib.bib29)), but find them failing to outperform Jaccard similarity.

4.  4.

    Generation. $CC$ is concatenated with the in-file context as a prompt for $\mathcal{M}$. The prompt is provided below.


#### Prompt

Recent literature demonstrates the effectiveness of directly providing the retrieved information as part of the context of LMs (Ram et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib28); Shi et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib31)). Following these studies, we directly concatenate the in-file context with $CC$ to provide it to the model ([Figure 1](https://arxiv.org/html/2403.10059v2#S1.F1 "In 1 Introduction ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")). To prompt CodeGen-Mono, we use the following input ordering:

\[Right Context\] \[Cross-file Context\] \[Left Context\]

To prompt StarCoder, we use the following fill-in-the-middle-prompt:

<fim\_prefix> \[Left Context\] <fim\_suffix> \[Right Context\] \[Cross-file Context\] <fim\_middle>

For the cross-file contexts, we add a # symbol to present them as comments and add the following line before each $cc_{i}$:

\# the below code fragment can be found in: \[file path\]

After concatenating the verbalized $cc_{i}$ together, we add another line to the start of the $CC$:

\# Here are some relevant code fragments from other files of the repo:

For the in-file completion baselines such as in [Section 5.1](https://arxiv.org/html/2403.10059v2#S5.SS1 "5.1 Is retrieval always helpful? ‣ 5 Results ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") and [Appendix B](https://arxiv.org/html/2403.10059v2#A2 "Appendix B Why infilling? ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"), our prompts are exactly the previous prompts with the \[Cross-file Context\] part removed.

#### Decoding and Post-processing

For all the experiments, we follow previous work and use greedy search (Zhang et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib40); Ding et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib5)). We left-truncate the left context to 1024 tokens, right-truncate the right context to 512 tokens, and right-truncate the cross-file context to 512 tokens. The max generation length is set to 50 tokens for line, API, and chunk completion, and 256 tokens for function completion. We perform task-specific post-processing on the model’s raw predictions. For line, API, and chunk completion, we truncate the prediction to having the same number of lines as in $Y$. For function completion, we first add a placeholder pass function body and use tree-sitter<sup>10</sup><sup>10</sup>10[https://tree-sitter.github.io/tree-sitter/](https://tree-sitter.github.io/tree-sitter/) to determine the position of the function in the file. Then, we concatenate the $X_{l}$ and $\hat{Y}$, parse the string again with tree-sitter, and extract the function body as the final $\hat{Y}$ if the string can be parsed. Otherwise, we directly return the raw $\hat{Y}$ without post-processing.

## Appendix B Why infilling?

As part of the in-file context, $X_{r}$ contains rich information about how the future execution relies on the code to complete. Right contexts are also shown useful for tasks such as function call argument completion (Pei et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib27)). However, previous literature such as Zhang et al. ([2023](https://arxiv.org/html/2403.10059v2#bib.bib40)) suggests splitting $X_{r}$ and retrieving code chunks from it. With code LMs trained on fill-in-the-middle such as StarCoder, we argue that directly providing $X_{r}$ in the prompt is more preferable.

To illustrate, we investigate the effect of directly providing $X_{r}$ in the prompt for CodeGen-Mono 16B and StarCoder on current-file code completion and retrieval-augmented code completion. Figure [7](https://arxiv.org/html/2403.10059v2#A2.F7 "Figure 7 ‣ Appendix B Why infilling? ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") presents the performance on RepoEval with different types of contexts provided in the prompt. Whether cross-file contexts are present or not, providing right contexts can greatly improve the code completion performance. The gain is consistent for both API and function completion. Compared to CodeGen, StarCoder can better leverage the right context to generate more accurate code. Overall, we observe that leveraging the entire right context to perform infilling represents a much stronger baseline. Therefore, in this paper we have exclusively focused on the infilling setting with the StarCoder family.

![Refer to caption](https://arxiv.org/html/2403.10059v2/x7.png)

Figure 7: A comparison between four prompting strategies for RepoEval by combining left context (L), right context (R), and cross-file contexts (CC). Leveraging right contexts to build infilling-style prompt generally improves the performance regardless whether CC is present or not. StarCoder exhibits larger gains from right contexts, potentially due to its fill-in-the-middle pre-training.

## Appendix C Trial Retrieval and Trial Generation

In this section, we present a detailed evaluation of two selective RAG strategies: trial retrieval and trial generation.

### C.1 Trial Retrieval

To gauge the relevance of retrieved context, using the similarity scores from the retrievers is a natural option. In this section, we investigate trial retrieval as a baseline for informing the decisions for selective RAG. We apply three off-the-shelf retrievers on RepoEval. For each retriever, we score each of the instances with the similarity between the top-1 retrieved code chunk and the query. The score is compared to a threshold decide whether the prompt should feature $CC$ or not. If score is higher than the threshold, we use top-10 code chunks retrieved by the same retriever as the cross-file context. We consider the following three retrievers:

-   •

    jaccard: the Jaccard index (Jaccard, [1912](https://arxiv.org/html/2403.10059v2#bib.bib12)).

-   •

    weighted\_ngram: the weighted n-gram matching term introduced in the CodeBLEU metric (Ren et al., [2020](https://arxiv.org/html/2403.10059v2#bib.bib29)).

-   •

    unixcoder: the cosine similarity of UniXcoder embedding (Guo et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib8)).


Figure [8](https://arxiv.org/html/2403.10059v2#A3.F8 "Figure 8 ‣ C.1 Trial Retrieval ‣ Appendix C Trial Retrieval and Trial Generation ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") presents the selective RAG performance of StarCoder under different budgets.We observe that the retrievers’ similarity scores serve as a promising signal for deciding whether the retrieved information can improve the RAG performance. For most retrievers and tasks, the performance of full retrieval could be reached with at most 60% retrieval budget. This trend also aligns with the remark in Zhang et al. ([2023](https://arxiv.org/html/2403.10059v2#bib.bib40)) on the correlation between in-repository duplication and the gain from $CC$. However, it is worth noting that this strategy brings no latency gain as it still implements always retrieving. In addition, the knowledge of whether the LM could be benefited by the retrieved context is not leveraged.

![Refer to caption](https://arxiv.org/html/2403.10059v2/x8.png)

Figure 8: A comparison of the effectiveness of different similarity functions for selective RAG with StarCoder 16B. We plot the retrieval budget in the x-axis, which is the percentage of instances to perform retrieval. We report score on the entire testing dataset for each budget. Specifically, the retriever’s similarity score is used select a subset to perform retrieval, and for the other instances in-file completion is performed without retrieval. In most of the cases, 40% retrieval can be saved without sacrificing the code completion performance.

![Refer to caption](https://arxiv.org/html/2403.10059v2/x9.png)

Figure 9: A comparison of the effectiveness of two uncertainty metrics for selective RAG with StarCoder 16B. We plot the retrieval budget in the x-axis and report score on the entire testing dataset for each budget. We observe that the uncertainty-based metrics fail for long sequence generation such as function completion. Token uncertainty outperforms entropy for line completion while entropy is slightly better for API completion. Overall, we find that uncertainty-based selective RAG is not as effective as retriever-based ([Figure 8](https://arxiv.org/html/2403.10059v2#A3.F8 "In C.1 Trial Retrieval ‣ Appendix C Trial Retrieval and Trial Generation ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion")).

### C.2 Trial Generation

Next, we evaluate two uncertainty-based selective RAG strategies that have been explored by previous works.

-   •

    entropy: the sequence-level entropy as used in Li et al. ([2023a](https://arxiv.org/html/2403.10059v2#bib.bib19)). We estimate the entropy by performing vanilla sampling for 20 times without any temperature scaling or distribution truncation.

-   •

    token uncertainty: the probability of the most unlikely token in the sequence decoded with greedy search, as used in Jiang et al. ([2023](https://arxiv.org/html/2403.10059v2#bib.bib14)). This metric can be seen as the lower bound of the per-token maximum probability.


Figure [9](https://arxiv.org/html/2403.10059v2#A3.F9 "Figure 9 ‣ C.1 Trial Retrieval ‣ Appendix C Trial Retrieval and Trial Generation ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") presents the selective RAG performance of StarCoder under different budgets, similar to the previous evaluation setting. We find that the selective RAG performance of uncertainty-based metrics is inconsistent across sequence lengths. As the length of $\hat{Y}$ increases (from line to API, and form API to function), the effectiveness of uncertainty-based metrics drops significantly. In addition, the selective performance cannot outperform the methods based on trial retrieval.

## Appendix D Data Creation for Repoformer Training and CrossCodeLongEval

We present the full self-supervised data creation algorithm in [Algorithm 1](https://arxiv.org/html/2403.10059v2#alg1 "In Appendix D Data Creation for Repoformer Training and CrossCodeLongEval ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") (for chunk completion data) and [Algorithm 2](https://arxiv.org/html/2403.10059v2#alg2 "In Appendix D Data Creation for Repoformer Training and CrossCodeLongEval ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") (for function completion data). $R_{filtered}$ stands for the remaining repositories after applying the filtering criteria in [Section 3.3](https://arxiv.org/html/2403.10059v2#S3.SS3 "3.3 Self-supervised Multi-task Learning ‣ 3 Approach ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"). In the next section, we present further analyses on the training data distribution.

Algorithm 1 Repoformer Training Data Creation (Chunk Completion)

  

Input: Filtered set of repositories

$R_{filtered}$

, language model

$\mathcal{M}$

, label threshold

$T$

  

Output: chunk completion training dataset

$\mathcal{D}$

  $\mathcal{D}\leftarrow\emptyset$

  

for each

$r\in R_{filtered}$ 

do

     $\mathcal{D}_{r}\leftarrow\emptyset$

     $\mathcal{C}_{raw}\leftarrow$

Break

$r$

into non-overlapping chunks of 10 lines each

     $\mathcal{C}_{r}\leftarrow$

Cluster

$\mathcal{C}_{raw}$

with KMeans using TF-IDF features, with the constraint

$|\mathcal{C}_{r}|=0.2|\mathcal{C}_{raw}|$

     

for each

$c\in\mathcal{C}_{r}$ 

do

        $k\sim\text{Poisson}(\lambda=3)$

        $s\leftarrow$

Randomly sample a chunk from

$c$

        $Y\leftarrow$

Randomly cut a sub-chunk from

$s$

that spans

$k$

consecutive lines

        $X_{l},X_{r}\leftarrow$

Recover the in-file left context and right context corresponding to

$Y$

        

if

$rand(0,1)>0.5$ 

then

           $\mathcal{Q}\leftarrow$

Concatenate(last

$5k$

lines of

$X_{l}$

,

$Y$

, first

$5k$

lines of

$X_{r}$

) // query formation

else

           $\mathcal{Q}\leftarrow$

Concatenate(last

$5k$

lines of

$X_{l}$

, first

$5k$

lines of

$X_{r}$

)

end if

        $CC\leftarrow$

Retrieve top-3 cross-file contexts from

$r$

using

$\mathcal{Q}$

via jaccard similarity, each of length

$10k$

        $\hat{Y}_{base}\leftarrow\mathcal{M}(X_{l},X_{r})$

        $\hat{Y}_{RAG}\leftarrow\mathcal{M}(X_{l},X_{r},CC)$

        $label\leftarrow ES(\hat{Y}_{RAG},Y)-ES(\hat{Y}_{base},Y)>T$

// boolean value

        Append

$(X_{l},X_{r},\ Y,\ CC,\ label)$

to

$\mathcal{D}_{r}$

end for

     $\mathcal{D}\leftarrow\mathcal{D}\cup\mathcal{D}_{r}$

end for

Algorithm 2 Repoformer Training Data Creation (Function Completion)

  

Input: Filtered set of repositories

$R_{filtered}$

, language model

$\mathcal{M}$

, label threshold

$T$

  

Output: function completion training dataset

$\mathcal{D}$

  $\mathcal{D}\leftarrow\emptyset$

  

for each

$r\in R_{filtered}$ 

do

     $\mathcal{D}_{r}\leftarrow\emptyset$

     $\mathcal{C}_{raw}\leftarrow$

Gather all the functions between 3 and 30 lines

     $\mathcal{C}_{r}\leftarrow$

Cluster

$\mathcal{C}_{raw}$

with KMeans using TF-IDF features, with the constraint

$|\mathcal{C}_{r}|=0.2|\mathcal{C}_{raw}|$

     

for each

$c\in\mathcal{C}_{r}$ 

do

        $s\leftarrow$

Randomly sample a function from

$c$

        $Y\leftarrow$

Cut only the body part of the function

        $X_{l},X_{r}\leftarrow$

Recover the in-file left context and right context corresponding to

$Y$

        

if

$rand(0,1)>0.5$ 

then

           $\mathcal{Q}\leftarrow$

Concatenate(last

$20$

lines of

$X_{l}$

,

$Y$

, first

$20$

lines of

$X_{r}$

)

else

           $\mathcal{Q}\leftarrow$

Concatenate(last

$20$

lines of

$X_{l}$

, first

$20$

lines of

$X_{r}$

)

end if

        $CC\leftarrow$

Retrieve top-3 cross-file contexts from

$r$

using

$\mathcal{Q}$

via jaccard similarity, each of length

$10k$

        $\hat{Y}_{base}\leftarrow\mathcal{M}(X_{l},X_{r})$

        $\hat{Y}_{RAG}\leftarrow\mathcal{M}(X_{l},X_{r},CC)$

        $label\leftarrow ES(\hat{Y}_{RAG},Y)-ES(\hat{Y}_{base},Y)>T$

// boolean value

        Append

$(X_{l},X_{r},\ Y,\ CC,\ label)$

to

$\mathcal{D}_{r}$

end for

     $\mathcal{D}\leftarrow\mathcal{D}\cup\mathcal{D}_{r}$

end for

#### Training Data Analysis

For the 240k chunk completion and 120k function completion instances, we plot the performance change after providing $CC$ in [Figure 10](https://arxiv.org/html/2403.10059v2#A4.F10 "In Training Data Analysis ‣ Appendix D Data Creation for Repoformer Training and CrossCodeLongEval ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"). In total, 30.18% chunk completion instances and 35.16% function completion instances are labeled with positive (i.e., retrieval should be triggered). The average length of $Y$ is 3.53 lines for chunk completion and 11.77 lines for function completion.

![Refer to caption](https://arxiv.org/html/2403.10059v2/x10.png)

Figure 10: The performance gain on Repoformer training data exhibited by StarCoderBase-1B from retrieved cross-file context. The sign of the performance change is used to generate the label for Repoformer training. Each (start, end) bucket contains values ranging from start to end except for the central bucket, which corresponds to exactly 0.

#### CrossCodeLongEval Construction

One drawback of RepoEval is its limited repository coverage. To verify the performance on diverse repositories, we collect and curate a new evaluation dataset for repository-level code completion.

-   •

    Repository collection. We first solicited 1744 raw Python repositories from the authors of CrossCodeEval (Ding et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib5)). These repositories were created between 2023-03-05 to 2023-06-15 and collected on 2023-09-01. They have been ensured to not overlap with the Stack (Kocetkov et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib16)).

-   •

    Target line sampling. We avoided using the CrossCodeEval benchmark as the original benchmark explicit removed the instances where StarCoderBase-1B can correctly answer without the retrieved context. To simulate a more natural distribution of code completion, we sample new blanks from the raw repositories. Specifically, we run [Algorithm 1](https://arxiv.org/html/2403.10059v2#alg1 "In Appendix D Data Creation for Repoformer Training and CrossCodeLongEval ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") and [Algorithm 2](https://arxiv.org/html/2403.10059v2#alg2 "In Appendix D Data Creation for Repoformer Training and CrossCodeLongEval ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") to gather chunk completion and function completion instances.

-   •

    Data analysis In [Table 6](https://arxiv.org/html/2403.10059v2#A4.T6 "In CrossCodeLongEval Construction ‣ Appendix D Data Creation for Repoformer Training and CrossCodeLongEval ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"), we present the basic statistics of RepoEval and CrossCodeLongEval.


|RepoEval|CrossCodeLongEval|
|---|---|
|Line|API|Function|Chunk|Function|
|\# repositories|16|16|16|944|1460|
|\# instances|1600|1600|455|5000|5000|
|$|X_{l}|_{line}$|30.7|30.8|31.1|
|$|X_{l}|_{token}$|796.3|890.7|761.1|
|$|X_{r}|_{line}$|15.1|13.9|16.2|
|$|X_{r}|_{token}$|449.9|430.4|412.4|
|$|Y|_{line}$|1.0|2.1|7.8|
|$|Y|_{token}$|12.0|25.4|97.8|

Table 6: Descriptive statistics of RepoEval and CrossCodeLongEval. For $|Y|$, $|X_{l}|$, and $|X_{r}|$, we report both the number of lines as well as the number of tokens (using the StarCoder tokenizer) in the groundtruth, left context, and the right context.

## Appendix E Extended Analyses

### E.1 Calibration of Repoformer’s Selective Retrieval Prediction

We evaluate the calibration of Repoformer-1B’s selective decisions. [Figure 11](https://arxiv.org/html/2403.10059v2#A5.F11 "In E.1 Calibration of Repoformer’s Selective Retrieval Prediction ‣ Appendix E Extended Analyses ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") plots the probability of <cc> against the probability of the model’s performance could be improved by the $CC$, measured by comparing the prediction with and without $CC$. When ES is used as the evaluation metric, Repoformer-1B generally makes near-calibrated predictions for Line and API Completion. However, when it comes to longer-formed function completion, especially when UT is employed as the metric, Repoformer-1B’s predictions are not calibrated. One possible reason is the use of ES as the training signal. We encourage future work to devise methods for effectively labeling the correctness of function completion. In addition, future work should consider training Repoformer to perform multiple self-assessments for long-form generations.

![Refer to caption](https://arxiv.org/html/2403.10059v2/x11.png)

Figure 11: The calibration of selective retrieval predictions. Repoformer makes generally calibrated predictions when ES is used as the metric and the generation is of moderate lengths. The prediction is not calibrated for function completion when the metric is UT.

### E.2 CrossCodeEval and Multilingual Repoformer

This section provides additional results on the 4-language original CrossCodeEval test set (Ding et al., [2023](https://arxiv.org/html/2403.10059v2#bib.bib5)). We choose to not present the results in the main text as the data creation process of CrossCodeEval explicitly selected the instances where cross-file information is generally required, thus making the contributions from selective retrieval incomplete. On this dataset, we evaluate StarCoder, Repoformer-1B/3B/7B trained on Python and Repoformer-M trained on multilingual repository-level code completion. Despite the setup difference, we are still able to observe substantial performance gains.

#### Multilingual Repoformer

We experimented with applying the Repoformer training scheme to multiple languages. Specifically, we collect public Python, Java, C#, and TypeScript repositories from the Stack (Kocetkov et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib16)) that contain at least 20 files and 20,000 lines of code. We do not apply the local import criteria due to implementation difficulties. Then, we follow the algorithm described in [Appendix D](https://arxiv.org/html/2403.10059v2#A4 "Appendix D Data Creation for Repoformer Training and CrossCodeLongEval ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") to create 90k chunk completion and 30k function completion instances per language. Using this dataset, we fine-tune StarCoderBase following the setup described in [Section 4.1](https://arxiv.org/html/2403.10059v2#S4.SS1 "4.1 Repoformer Implementation Details ‣ 4 Experimental Setup ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") (same infrastructure and hyperparameters). We call this model Repoformer-M.

#### Evaluation Results

We present the results on CrossCodeEval in [Table 7](https://arxiv.org/html/2403.10059v2#A5.T7 "In Evaluation Results ‣ E.2 CrossCodeEval and Multilingual Repoformer ‣ Appendix E Extended Analyses ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") and summarize the observations below:

-   •

    Strong cross-lingual transfer. Repoformer trained on Python data achieves strong performance across multiple languages, including three languages it is not fine-tuned on. The result highlights the generalizability of the learned self-evaluation and robust code completion abilities.

-   •

    Multi-lingual Repoformer. Repoformer-M outperforms the same-sized StarCoderBase by a large margin. For the 1B, 7B, Repoformer-M outperforms Repoformer by a small margin. For 3B, the two models give similar performance. This is reasonable as the two models are learned on similar sized training data.


|Model|RAG Policy|Python|Java|C#|TypeScript|
|---|---|---|---|---|---|
|Code ES|ID F1|Code ES|ID F1|Code ES|ID F1|Code ES|ID F1|
|StarCoderBase-1B|No|68.83|58.18|73.60|63.69|79.30|66.40|67.09|60.15|
|Always|71.57|62.42|74.54|65.83|79.04|66.82|67.66|60.60|
|\\hdashlineRepoformer-1B|Selective<sub id="A5.T7.2.1.5.3.2.1">T</sub>|71.29|62.81|75.12|67.16|83.08|74.24|69.90|64.07|
|Repoformer-M-1B|Selective<sub id="A5.T7.2.1.6.4.2.1">T</sub>|71.55|62.89|75.92|67.86|84.44|76.00|70.07|64.41|
|StarCoderBase-3B|No|71.07|61.63|76.10|67.56|81.46|69.95|70.56|64.83|
|Always|73.65|65.93|77.52|70.15|81.75|71.26|70.91|65.09|
|\\hdashlineRepoformer-3B|Selective<sub id="A5.T7.2.1.9.7.2.1">T</sub>|74.57|66.86|78.40|71.26|85.92|78.62|73.70|68.66|
|Repoformer-M-3B|Selective<sub id="A5.T7.2.1.10.8.2.1">T</sub>|73.80|66.72|77.68|71.01|85.31|77.70|72.51|67.06|
|StarCoderBase-7B|No|72.47|63.76|77.21|68.97|83.06|72.06|72.34|67.06|
|Always|75.02|67.69|77.70|70.57|83.64|74.39|73.01|67.56|
|\\hdashlineRepoformer-7B|Selective<sub id="A5.T7.2.1.13.11.2.1">T</sub>|75.34|68.27|78.90|72.35|83.80|76.88|73.59|69.10|
|Repoformer-M-7B|Selective<sub id="A5.T7.2.1.14.12.2.1">T</sub>|75.35|67.88|79.11|72.82|86.53|79.77|74.60|70.01|

Table 7: Evaluation results on CrossCodeEval. We report edit similarity for code matching as well as the F1 score for identifier matching. The best scores across all models are boldfaced.

### E.3 Repoformer’s Robustness to the Retriever Choice

In this section, we investigate the performance of Repoformer with the cosine similarity of UniXcoder embedding (Guo et al., [2022](https://arxiv.org/html/2403.10059v2#bib.bib8)) as the retriever instead of Jaccard similarity. As shown in [Table 8](https://arxiv.org/html/2403.10059v2#A5.T8 "In E.3 Repoformer’s Robustness to the Retriever Choice ‣ Appendix E Extended Analyses ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"), we are able to observe similar patterns compared to [Table 3](https://arxiv.org/html/2403.10059v2#S5.T3 "In Latency Model ‣ 5.3 Repoformer improves inference efficiency ‣ 5 Results ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"): selective retrieval is able to improve both the accuracy and the latency of the entire RAG system. In addition, as retrieval consumes a larger proportion of latency than when sparse retriever is used, selective retrieval brings more substantial performance gains, with threshold selection bringing more than 70% speedup.

|Model|RAG Policy|API Completion|Line Completion|
|---|---|---|---|
|ES|%RAG|SU|ES|%RAG|SU|
|Always|71.69|$100\%$|$0\%$|75.25|$100\%$|0%|
|Selective<sub id="A5.T8.7.7.7.6.1">G</sub>|70.82|$18\%$|$71\%$|73.70|$19\%$|$71\%$|
|Repoformer-1B|Selective<sub id="A5.T8.11.11.11.6.1">T</sub>|72.39|$61\%$|$33\%$|75.65|$62\%$|$33\%$|
|Always|74.48|$100\%$|$0\%$|78.24|$100\%$|0%|
|Selective<sub id="A5.T8.18.18.18.6.1">G</sub>|73.26|$19\%$|$65\%$|76.74|$20\%$|$66\%$|
|Repoformer-3B|Selective<sub id="A5.T8.22.22.22.6.1">T</sub>|74.69|$78\%$|$21\%$|78.63|$74\%$|$31\%$|

Table 8: RAG performance of Repoformer with two self-selective RAG paradigms and dense retrieval used instead of Jaccard similarity. %RAG = ratio of instances where RAG is performed. SU = Speedup compared to always retrieving. Compared to the always retrieving baseline, the Selective<sub id="A5.T8.30.2.4">T</sub> strategy consistently demonstrates gains in both accuracy and latency. The Selective<sub id="A5.T8.30.2.5">G</sub> strategy shows much larger latency gains with a small performance degradation. Compared to sparse retrieval, we observe more substantial latency gains.

### E.4 Full Latency-Accuracy Visualizations

In this section, we present the latency-accuracy trade-off plots for Repoformer-1B, Repoformer-3B, StarCoderBase-7B, and StarCoder on the three tasks from RepoEval. We use self-selective RAG for the Repoformer models and for StarCoder, we use Repoformer-1B to make the selective RAG decisions. The results are presented in [Figure 12](https://arxiv.org/html/2403.10059v2#A5.F12 "In E.4 Full Latency-Accuracy Visualizations ‣ Appendix E Extended Analyses ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion") to [Figure 15](https://arxiv.org/html/2403.10059v2#A5.F15 "In E.4 Full Latency-Accuracy Visualizations ‣ Appendix E Extended Analyses ‣ Repoformer: Selective Retrieval for Repository-Level Code Completion"). Overall, we observe that no matter for self-selective RAG or making selective predictions for a larger model, Repoformer is able to improve the accuracy and latency at the same time. The improvement is more apparent in the line and API completion tasks. For function completion, as discussed in the main text, RepoEval uses very small repositories to enable easy unit testing. As a result, the retrieval overhead is low in general and thus does not significantly affect the latency of the entire RAG system.

![Refer to caption](https://arxiv.org/html/2403.10059v2/x12.png)

Figure 12: Latency-accuracy trade-off of self-selective RAG for Repoformer-1B.

![Refer to caption](https://arxiv.org/html/2403.10059v2/x13.png)

Figure 13: Latency-accuracy trade-off of self-selective RAG for Repoformer-3B.

![Refer to caption](https://arxiv.org/html/2403.10059v2/x14.png)

Figure 14: Latency-accuracy trade-off of selective RAG for StarCoderBase-7B. Repoformer-1B is used for the selective decisions.

![Refer to caption](https://arxiv.org/html/2403.10059v2/x15.png)

Figure 15: Latency-accuracy trade-off of selective RAG for StarCoder. Repoformer-1B is used for the selective decisions.
