---
layout: post
title: Using Markov Chains to simulate human speech
subheading: Will Markov Chains be enough to implement a decent chatbot?
categories: [Projects, NLP]
tags: [Python, Markov Chains, Chatbot, NLP]
---

Nowadays there are many sophisticated ways to approach NLP, most of them involves
neural networks. For example, OpenAI's GPT2 has achieved phenomenonal results in NLP
and can be used in a variety of fields.

However, can something as simple as Markov Chains compete with sophisticated techniques
like RNNs, or even better, GPT-2? *Hint: It can't but I thought it was interesting*

Skip to the bottom for the technical implementation. [Github project link here](https://github.com/thebowenfeng/markov_chatbot-new-)

## What are markov chains?

In mathematical terms, a **Markov Chain** is defined as
> A directed graph with weighted edges connecting states with each other.

*Definition may vary*

In English, a Markov Chain is essentially an algorithm that selects the next most
possible event, based on previous events, using probabilities. Similar to a RNN
(Recurrent Neural Networks), it is able to predict the next *state* based on
information on previous states. However, unlike a Recurrent Neural Network, 
its algorithm is much more simplistic, and relies on pure probability.

What this means is that, if event A -> event B frequently leads to event C, then
event C will more likely to be chosen as the next "predicted state"

## How does Markov Chains work in a chatbot

Knowing how Markov Chains work, we could swap *states* to *words* in a sentence.
In other words, the next "state" being predicted will simply be the next word in 
a given sentence. 

This might sound like a brilliant idea. After all, words that appear together frequently
are more likely to make sense. For instance, given the words "I" and "am", the algorithm
is likely to predict the word "fine", as "I am fine" frequently appears together. 

Therefore, Markov Chains should generate perfect human sentences (or at least near perfect),
right?

![Duck saying NO](https://c.tenor.com/uYn6YAkOoi0AAAAM/duck-no.gif)

## Problems with Markov Chains and NLP

Its really easy to miss, and obviously my observations are with the benefit of hindsight, but
if we rely on pure probabilities, then sentences will not make logical sense, despite
possessing "flow".

You see, just because a sentence flows along nicely, does not necessarily mean it makes
logical sense, or contains any meaning. Let's take the following sentence:
- "I am really good with guns should be banned"

Now you may think "what a weird sentence", and you'd be absolutely right in your observation.
This sentence, by most definitions, makes **zero** sense. In fact, it looks like
two sentences clumped together, namely "I am really good with guns" and "guns should be banned". 

So, despite the fact that each word in the sentence "connects" to the next word,
the sentence as a whole fails to give any meaning. This is the problem with Markov Chains.

Now, as a reminder, Markov Chains work by taking in several previous words, and attempt
to predict the next words. This leads to several problems:

#### Corpus

As with most "Machine Learning" algorithms, this algorithm is trained on a large blob
of text. The problem with text is that it will typically vary in meaning, even if it came from a singular
source. Because of the way Markov Chains work, the text will have to be broken into
small tokens (discussed later), which maps a set of words with the next word in the sequence
(For instance, I am good thanks might become "I am good", "am good thanks").

This means that *tokens* from a variety of different sentences will be mixed together. 
As Markov Chains are based on probabilities, it does not care if a token exists within
the same context as a previous used token. So, obviously this can lead the problems like
the above sentence, where individual "tokens" might make sense, but because
said tokens exists in different context, the resulting sentence makes no sense.

#### Scope

Scope, in this case, refers to *How much stuff is the algorithm taking into account,
when predicting the next word?*, and the answer is *not much actually*.

You see, when predicting the next word, the algorithm only cares for the previous
few words. After predicting, it completely forgets and starts anew. This is akin to someone
with Alzheimer's so severe that they will forget what they are talking about mid-sentence.
In fact, our case is even worse, as the algorithm will forget every 3/4 words.

So if we were to entirely ignore the previous problem, and assume every single text
came from the exact same context, I highly doubt the sentence will still make logical
sense, as the algorithm will possess no knowledge of what they've generated, 2 seconds ago.

So now that we've discussed the Pros and Cons of Markov Chains, let's see how we can
implement it in code.

## Technical implementation

The implementation is actually quite simple. We will break down the text into
tri-grams, which is essentially every sequential group of "3" (hence tri) words in a sentence.
That might not make too much sense, so here is an example:

Suppose we have this sentence: "Hello there, how are you?"

The sentence will be broken down into the following:

- Hello there, how
- there, how are
- how are you?

*The punctuations are there purely for ease of understanding, and will be removed
in the actual dataset*

The last word in each tri-gram will be the "predicted state". The algorithm will take in
two words, search for a trigram whose first two words match, and will output the 
third word in the tri-gram as its prediction. 

So how does the whole probability thing plays in?

You see, when two tri-grams have the same first two words, but perhaps different
(or even same) last word, the algorithm will not intuitively know which one to choose.
To it, every single option makes sense, as that's what people really say.
So, instead, we will collate all "matched" tri-grams, and randomly
select one to be the predicted word. This way, the more identical tri-grams
you have, the higher the chance that particular tri-gram will be picked.

That's it, that's all there is to it. 
[Here is my implementation](https://github.com/thebowenfeng/markov_chatbot-new-)
, along with some scraping code to obtain the corpus.

## Final thoughts

Whilst it is an interesting experiment, pure probabilities obviously do not
cut it when dealing with something as complex and nuanced as human languages. 
However, it is important to recognize that some concepts used in Markov Chains,
such as previous states, is also used in more modern approaches, such as RNNs.

Of course, it is possible to make some improvements upon this bare-metal model. 
For instance, you could restrict the algorithm to select only tri-grams belonging 
in the same "context" (which will require some manual labelling), in order
to improve its "make-sense-ness". However, even with such improvements,
or whichever improvement you might've thought of, it is unlikely there will
be any fruitful results from such a rudimentary algorithm. 

So, for now, Markov Chains chatbot will remain, a fun experiment.
