---
layout: post
title: "Mechanistic Exploration Gemma 2 List Generation"
date: 2024-10-03
---


## Mechanistic Exploration of Gemma 2 2b list creation


### 0. Abstract




Sparse Autoencoders (SAEs) have recently emerged as powerful tools for exploring the mechanisms of large language models (LLMs) with greater granularity compared to previous methods. Despite the great potential of SAEs, concrete and non-trivial applications remain elusive (**honorable mentions** [Goodfire AI](https://goodfire.ai/blog/research-preview/),  [Golden Gate Claude](https://www.anthropic.com/news/golden-gate-claude) ) . This situation motivated the present investigation into how Gemma 2 2b Instructed generates lists. Specifically, we conduct an exploratory analysis to understand how the model determines when to end a list. To achieve this, we utilize the suite of SAEs known as GemmaScope. Although initial traction has been gained on this problem, a concrete mechanism remains elusive. However, several key results are significant for how MI should approach non-trivial problems.


### 1. Introduction

Gemma 2 2b is a Small Language Model, created by google made public in the summer of 2024. This model was released along a suite of Sparse Autoenocders and Transcoders trained on their activations in various locations.

Despite it's size the model, is incredible capable, excelling in it's instruction tuned variant with the ability to follow instruction and with similar performance to the original GPT-3 model on some benchmarks.

This facts makes Gemma 2 2b a great candidate to perform MI experiments on, offering a great balance between performance and size.

In this post we will explore the mechanisms behind Gemma 2 2b's ability to create lists of items, when prompted to.

Specially we are interested in the mechanism by which Gemma knows when to end a list, this is task is interesting for the following reasons:

- Due to the larger Gemma 2 vocabulary is easy to create one token per item list templates, avoiding the mess of position indexing.
- The instruction tuning of the model, enables the induction of different behaviors in model responses with minimal  changes in the prompt.
- The open endedness of this task enables taking into account sampling dynamics in the decoding process of the model (this is temperature and sampling method).
- The template structure enables a clear analysis of apriori very broad properties of the model like "list ending behavior" by proxies such as the probability of outputing a hypen after a list item which clearly indicates that the list is about to continue.



### 2.Data

To investigate the behavior of Gemma when asked for a list we create a synthetic dataset of model responses to several templates.


1) We ask GPT4-o to provide a list of topics to create lists about.

    This results in 23 final topics, that we will ask Gemma 2 to create lists about.

    Some examples, of those are: *Vegetables, Countries, Movies, Colors, etc*

2) We create a prompt, for Gemma:


    *Template 1 (Base)*
    ```
    Provide me  a with short list of {topic}. Just provide the names, no need for any other information.
    ```

    *Template 2 (Contrastive)*
    ```
    Provide me  a with long list of {topic}. Just provide the names, no need for any other information.
    ```

    *Template 3*
    ```
    Provide me  a with list of {topic}. Just provide the names, no need for any other information.
    ```


    **If not otherwise indicated, all the analysis and explorations where done with template 1**

3) For each topic, we sample 10 Gemma Completions with top-k = 0.9 and temperature=0.8


    *This step is crucial, because as can be seen in the Apendix actually sampling completions from Gemma allow us to observe the actual dynamics of the list ending behavior when compared with List Generated by GPT4o*





### 3. Exploratory Analysis

We start the Analysis with an exploratory overview of the completions provided by the model.
Few things that struck me as interesting, are the apparent consistency on the type of list that the model produces for a given template across topics, temperatures and samples, the tendency of the model of using filler blank tokens before ending the list.


```
Provide me  a with short list of Vegetables. 
Just provide the names, no need for any other information.</n> 
<start_of_turn><model>
- Carrots</n> 
- Peppers</n> 
- Celery< ></n> 
<end_of_turn>
```



1) Most of the item's in the list where a single token.
    
    As expected most of the item in the lists generated spanned just a single token, virtue of the large size of Gemma's vocabulary.
    
    

    *Plot of the number of items across topics and temperatures for Template 1*

    <p align="center">
      <img src="/assets/images/Gemma2_Lists/Token-Statistics-Temp.png" alt="Token Statistics Temp" />
    </p>

    For our prompt template, and with a few and notable exceptions most of the items in the lists where one-token long, this is likely a result of the expanded vocabulary size of Gemma 2 (roughly 5 times bigger than the one from GPT2 models).   Some notable exceptions where topics like Oceans, Cities or Countries.


2) For all the topics, in the last few items in the list the model sampled white space tokens after the item and before the line break.


    *Average number of tokens for each Item across topics, and temperatures (taking into account the hyphen and line break token 1+1+1)*
    <p align="center">
      <img src="/assets/images/Gemma2_Lists/Token-Statistics-Temp-blank.png" alt="Token Statistics Temp Blank" />
    </p>


    **By far the most interesting behavior that we've observed trough different topics and temperatures is the model behavior of including blank tokens near the end of the list.**

    *This is very speculative, and I don't have and hard prove but this is maybe an RLHF/Post Training artifact*

    We further investigate this behavior using attribution techniques.



3) The number of items in each list with a white space added is pretty consistent across topics, with a few outliers.

    In most of the cases just the last item in the list had the blank token, this was not the case though for some topics where more than just the last item had the strange blank token.



4) The number of items in each list is also very similar across topics.

5) There exist a correlation between the sampling temperature and the number of items in a list with a blank spaces token before the end of the list.


    This is an expected, given the fact that greater temperatures, might push the model to miss on the end of the list due to chance.
    <p align="center">
      <img src="/assets/images/Gemma2_Lists/Fraction-Blankcs.png" alt="Fraction Blankcs" />
    </p>

6) For prompts, where we asked for a long list, the average number of items is 30, and we no longer observe an abudance of white space tokens at the end of the list.


    Table with the number of items generated for each template,  across topics and samples.


    | Template| Average Number of Items | 
    |------------|:-----------------------:|
    | Template 1 | 5 |
    | Template 2 | 30 |
    | Template 1 | 10 |

    *The numbers are orientative, in reality outlires skew the average*






### 3.5. Entropy Analysis

When doing Mechansitic Interpretability analysis on Model Generated Outputs is very important to remember that the distribution of outputs for the model is not the same as the distributio of general text on the web or books.

This is one of the possible explanations behind the phenomena of [AI Models preferring their own text](https://arxiv.org/pdf/2404.13076).

One exploratory metrics that we can analyze is the evolution of the entropy of the models logits.

Concretely focus on the entropy at the item positions over the whole dataset.

*This means that if a generated list has 5 items, we take entropy readings in 5 different places*

This was done to the whole dataset of generated outputs for templates 1 and 2.
To get a better understanding of the entropy we also corrupted the prompt for templates 1 and 2 in the following way.



- Template 1: Clean

    `Provide me with a short list of ... - Item 1 \n Item 2 ...`
- Template 1: Corrupted (Short $\rarr$ Long)

    `Provide me with a long list of ... - Item 1 \n Item 2 ...`
- Template 2: Clean

    `Provide me with a long list of ... - Item 1 \n Item 2 ...`
- Template 1: Corrupted (Long $\rarr$ Short)

    `Provide me with a short list of ... - Item 1 \n Item 2 ...`


The arrow indicated that the token " short" or " long" where replaced while mantianing the rest of the Instuction + Generated List intact.


It's important to note that the generated list either long or short where not changed, just the short/long token.

Also note, tat there's a left padding hence the x-axis size is not representative of the average number of items in generated lists.

**Template 1 Clean**


For the generated outputs of the model with the base prompt we have the following entropy plot.

<p align="center">
  <img src="/assets/images/Gemma2_Lists/Entropy-item-last-tok-short-clean.png" alt="Entropy Item Last Token Short Clean" />
</p>


We can clearly see how the entropy increase is gradual trough out the item positions, with a slight increase at the last positions.

**Template 1 Corrupt**

If we corrupt this base prompts by interchanging the token " short" with the token " long" and leaving everything else (including the generate list) intact.


<p align="center">
  <img src="/assets/images/Gemma2_Lists/Entropy-item-last-tok-short-corrupted.png" alt="Entropy Item Last Token Short Corrupted" />
</p>

We observe that there's no increase in entropy over the item positions, the entropy just increases at the last position.

Which might indicate that the model wasn't expecting the list to end due to introduced specification that asked for a long list.

**Template 2 Clean**

We can also do the opposite analysis, we can generate completions for prompts asking for long lists (Template 2).


<p align="center">
  <img src="/assets/images/Gemma2_Lists/Entropy-item-last-tok-long-corrupted.png" alt="Entropy Item Last Token Long Corrupted" />
</p>

Again we see a gradual increase in the model's entropy as the list approaches it's end.

**Template 2 Corrupt**

If we corrupt this prompts by interchanging the token " long" with " short" and leaving everything else (including the generated list) intact.

<p align="center">
  <img src="/assets/images/Gemma2_Lists/Entropy-item-last-tok-long-clean.png" alt="Entropy Item Last Token Long Clean" />
</p>

We can see and abrupt spike in entropy similar to the analogous case with Template 1.


### 4. First Approximations 



**Structure of the Prompt+Generation**

*Just the tokens that are difficult to infer are indicated*


<p align="center">
  <img src="/assets/images/Gemma2_Lists/type-pos-tok.png" alt="Entropy Item Last Token Long Clean" />
</p>

To investigate what is the mechanism behind Gemma ending a list, we must establish a proxy for it.

Given the nature of the problem is easy to stablish proxies for the behavior of ending a list:

There's several way in which we can approach it.

- The list ends when the model outputs the <end_of_turn> token instead of a new hyphen token .

    This seems to me to be the most natural way to go about this problem, still  this approach was discarded in favor of the following.

    *This can be operationalized like the difference between predicting the <end_of_turn> vs <-> tokens at the last </n> position*
    
- The list ends when the model uses a blank filler token, instead of the usual line break.
    
    This approach leverage the experimental findings from the previous section to establish a proxy for when the model finishes the list.
    *This can be operationalized like the difference between predicting the < > vs <\n> tokens at the last < Celery> position*

At first this can seem a little bit convoluted and unjustified, but this make's more sense when you realize using the first approach wouldn't account for the effect of the blank token that precedes the </n> token.  



**Logit Lens**

We investigate the logit difference between the blank and line break tokens (which we can call "list ending behavior").


Taking advantage of the shared RS across layers, we can use the unembedding matrix to project the activations across the layers into vocabulary space. 

This enables us to get an intuition of how a behavior builds trough the layers.

Using such technique we inspect the relevant positions for the list ending behavior across the dataset.

We use the difference between decoder's < > and <\n> direction as the list ending direction, to inspect the activations trough the layers.

In the example above the relevant positions would be the once corresponding to Carrots, Peppers and Celery.




*We plot the logit differnce across layrers for the various positions of interest*


<p align="center">
  <img src="/assets/images/Gemma2_Lists/Logit-Lens-Positions.png" alt="Logit Lens Positions" />
</p>

We can see that the 5'th item is the first one to clearly have positive logit difference.
A progressive increase in logit lens can be seen trough out the items.

The layers 20 to 26 seem the most relevant when it comes to express this direction.

*This behavior can also be seen trough out the whole dataset, with slight variations.*





### 6. Feature Attribution


Following the release of Gemma 2 2b, Google Deep Mind released a suite of sparse autoencoders trained of the model activations with multiple levels of sparsity, location, and dictionary size.

The decision of  which SAE's to select for a given ingestigation should we be carefuly examiend, taking into accounts multiple factors like reconstruction loss vs sparsity and dictionary size vs available memory.

For this investigation we used SAE's trained in the RS and attention outputs, with dictionary size of 16k (for ease of use), and we selected the sparsity based on availability of explanations in [Nueronpedia](https://www.neuronpedia.org)



One problem that is apparent to anyone that has tried to use SparseAutoencoders for real world task is that the memory footprint of SAE experiments rapidly explodes as we add layers.

*For reference a 2b model and 2 x 16k SAEs filled RTX3090's memory when grad was enabled.*


The intuitive solution to this problem is to come up with heuristics to select a few layers to use SAEs in, to maximize the faithfulness/GB vram ratio.

Possible heuristics are:

- Use Logit Lens style techniques to select the most important layers. 
- Use ablation over positions and layers to select the most important layers. (Some discounting should be done to no give too much importance to the last position or last layers)


Proper benchmarks are needed to rate this heuristics.

In the investigation we selected 5 layers for RS SAEs and 5 layers for Attn SAEs with mixture of patching and guessing by plots.


<div style="text-align: center;">

| SAE | Layers |
|:-------:|-----------:|
| Attn | 2, 7, 14, 18, 22 |
| RS | 0, 5, 10, 15, 20 |
</div>





### 8. Causal ablation of RS features over the layers


### Appendix

**Item break vs White space Locations**

**Should we divide Logit Lens by temperature**

**Heuristic for selecting the layers**

**Activation patching**

One of the nicest properties of this setup is that due to the instruction tuning of the model is very easy to induces certain model behaviors with minimal changes in the prompt.


In this case just changing the tokens "short list with a few" with "long list with a many" make the model output much longer list, and hence display different list ending behaviors for a given prompt.


This enables easy creation of the contrastive templates.



With this easy trick we are in an very similar situation (while not perfectly analogous ) to the IOI paper.

This enable simple patching experiments to identify crucial model components for the list ending behavior to occur.