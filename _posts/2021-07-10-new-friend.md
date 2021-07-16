---
layout: post
title: NewFriend -- A Slackbot to Replace your Friends
categories:
- Programming
tags: []
comments: []
---
There's something I love about bad robots. To this day, I can't watch this KetchupBot video without smiling.

<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/JcniyQYFU6M" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</center>

Part of what makes this robot so great, is that it hits the perfect level of crappiness. I think if it were any better or worse at applying ketchup, it would be less funny.
There's also something about putting a lot of careful work into a machine that's both crappy _and_ useless. 

Along these lines, Declan, Max, Phil and I built a Slackbot to imitate our friends. Every hour or so, it simulates a conversation among the users of our Slack workspace and posts it to a special channel in real time. It does a pretty good job of matching each person's style, vocabulary, subject matter, punctuation, and spelling. It doesn't do a good job of generating sentences that make sense. I think it kind of landed in KetchupBot territory. 

The basis of the bot is a statistical model called a Markov chain trained on our Slack history.

## What is a Markov Chain
A Markov chain models a system as a set of discrete states and transition probabilities between these state. I think a diagram is the best way to explain. 

Let's model the weather in Somerville, MA with a Markov chain model. The state will be the weather during one hour of the day.

![Rain and sun model](/assets/img/2021/07/sun_rain.svg){: .center}

In this model, the set possible states is `sun` and `rain`. This diagram says if it is sunny right now, there is a 90% chance that it will also be sunny in the next hour, and a 10% chance it will be raining. If it is raining right now, there's an 80% chance it will be raining in the next hour and a 20% chance it will be sunny in the next hour. The arrows leading out of each node sum to one, because it will always be doing _something_ in the next hour. 

This whole structure conforms to what's known as the _Markov property_, where the only factor that determines the probability of future states is the present state. To put it another way, the model is memory-less. To predict the state in an hour with a Markov chain, all we need to know is the current state.


## How can I use it to make fun of my friends?
To connect this to the problem of generating the sentence in a particular style, we need to change what a state means and what a transition represents. What would it look like if we view a sentence as a sequence of states. The current state is the last word typed, and the next state is the next word. The transition probabilities represent the probabilities of saying each word next. We'd be treating a sentence as a kind of semi-random walk through word space. 

### A Markov chain for sentences
Imagine you heard me say `I have a doctor's appointment on _____`. You've got a pretty good idea of what I'm about to say. Almost certainly the name of a day, and probably a weekday. Maybe I'll say `the` on my way to `the twenty-fifth of July`. There's an outside chance I'll surprise you by saying `a boat` or `Aquidneck Island`. Still, you can narrow it down a lot.

Problem is, the Markov model only gets to look at the current state, which we said was just the last word. In this example that would be `on _____`. With this restriction, you're probably much less sure of what I'm about to say, but there's still some information here. The next word is probably not another proposition like `at`, for example. 

Well, we can play kind of a funny trick and decide to call the last _two_ words the state. Doesn't this break the Markov property? No, the transition probabilities still only depend on the current state, we've just changed the definition of a state. Now the first state is `I have` and the second state is `have a`. The final state is `appointment on`. This state actually has a lot of information about what the next word will be, almost as much as the full sentence.

There's a tradeoff to opening up the state like this, and it has to do with the transition probabilities. There are way more of them now! As the number of words in the state increases, the number of possible states grows exponentially and the number of possible transitions does too. 

### Estimating the probabilities
Having a large number of states is a problem, because we're going to have a finite amount of data to estimate these probabilities from. Let's see what that looks like. A simple way to find these probabilities is to look at a corpus of text, find all the number of times each state ocurred, and count the number of times each possible transition actually happened. It's a little silly to build a Markov chain from a single sentence, but let's use that as the corpus, and take a look at the graph with a two word state and the above strategy for calculating transition probabilities.

![Doctor model](/assets/img/2021/07/doctor.svg){: .center}

This graph is pretty boring. Each state occurs exactly once, so the graph is linear. We only have one example of a transition from `I have`, and it's to `have a`, so the estimated probability of that transition is one. 

What if we use this corpus of four sentences instead:

> My favorite color is red

> My favorite color is blue

> My favorite color is green

> Your favorite color is blue

Now the graph would look like:

![Color model](/assets/img/2021/07/color.svg){: .center}

This Markov chain is a bit more interesting and illustrates the probability calculation. Since we have two instances of `color is` being followed by `is blue`, it is twice as probable in the Markov chain. There are also two states that lead to the `favorite color` state, each with probability one. 

This model doesn't reflect the fact that three of the four sentences start with `My`. We can solve that by adding special start and end tokens to the beginning and endings of the sentence. Effectively, just pretend that each sentence looks like this:

> START START My favorite color is red END END

We need two of each because our states are two words.

![Full color model](/assets/img/2021/07/color_full.svg){: .center}

### Generating sentences
Generating a new sentence from a Markov chain is simple. Start at the `START START` state and pick the next state according to the listed probabilities. This Markov chain could generate a sentence that doesn't appear in the input list. It has 12.5% chance of generating this sentence, for example:

> Your favorite color is red

## Markov Chain for Conversations
So far we've been generating sentences, but our Slackbot generates conversations between multiple people instead. We considered different ways of doing this, but settled on a method that I think is kind of clever. We replace each word with a tuple representing the word and the speaker. We also added an extra token indicating that a single message has ended. A conversation like this:

> Alice: Hey  
> Bob: Hey there

Is now represented with these tokens:
* `START`
* `(alice, Hey)`
* `SEND`
* `(bob, Hey)`
* `(bob, there)`
* `END`

Alice saying `Hey` is now a completely different token from Bob saying `Hey`. Instead of generating sentences by generating sequences of words, we can generate conversations as sequences of these tuple tokens. Here's what a model for this one sentence would look like:

![Conversation model](/assets/img/2021/07/convo.svg){: .center}

This graph looks little uglier, but it can encode a bunch of information we care about. Most importantly, it has the power to bridge the gap between individual messages so that they can be related to each other. It also captures when the pattern people choose to break up their messages. For example,
> I don't know  

is very different from
> I  
> don't  
> know

The `SEND` tokens allow us to encode that.

The downside of the tuple token approach is, of course, that it increases the number of states and possible transitions. We tried to limit this effect by ignoring case and punctuation on the output side of nodes. For example, these conversations:
> Alice: HEY  
> Bob: Hey there

> Alice: Hey  
> Bob: Hey there

would lead to this Markov chain:
![Conversation case model](/assets/img/2021/07/convo_case.svg){: .center}

In this case, Alice retains the ability to say both `Hey` and `HEY`, but Bob doesn't care about that distinction.

Want to try it out? Checkout our [GitHub repo](https://github.com/perciplex/new-friend).