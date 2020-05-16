---
title: Social Distancing, Moving Offices, and an Introduction to Probabilistic Thinking
description: Introducing probability theory with social distance guidelinies when moving offices
author: James Ingold
published: true
---

[Juxly](https://juxly.com){:target="\_blank"} is moving offices, it's not a big move, just right down the street but we have to pack and unpack: all the fun stuff that goes with moving. We've been working remotely for around two months at this point due to Covid-19 but we'll need to go to the office to pack or take stuff home that we don't want moved.

A co-worker created a schedule to use for who is going to the office and what time. This is to stay within the social distancing guidelines. My knee-jerk reaction to the calendar was that it "seems like overkill". We have 14 days to pack and ~12 in-office employees. I thought, we don't need a schedule (too many days, too few employees) but I wanted to examine hat further with a mini introduction into probabilistic thinking.

Probabilistic thinking is using probability theory (statistics) in every day life. What is the expectation that X will happen when you know Y sorts of things? You probably do this naturally from habit and life in general. You see dark clouds in the sky and internally your view of _probable_ rain goes up a tick.
In probability terms, things are expressed as percentages (0 to 1). When you see that dark cloud in the sky, depending on its nature maybe your expectation of rain goes from 0 to 20% or 90% if it looks really nasty.  
When you start to look around, probability is baked into pretty much everything in life.

### Life Skill: Thinking in Percentages

Annie Duke has a great book called ["Thinking in Bets"](https://amzn.to/2yWSgKI){:target="\_blank"} which details how to go from thinking in binary (yes/no) to expressing thoughts as percentages. This is immensely powerful, because black and white outcomes are actually quite rare. For most outcomes, the best answer is "it depends". Things in real life are not simple but we tend to simplify our thinking because it's easy. We can only make so many decisions in each day. But there is a big difference between "No, that can't happen", to "there's probably a 5% chance of that happening". “I’m not sure” is simply a more accurate representation of the world than it _can't_ happen. When you change the way you think, you start to illuminate the unknown and enter a world of possibilities.

### Back to the Task at Hand

That's enough of the map, let's get into the territory. What is the probability that multiple people will be in the office at the same time?  
We're going to examine this from a probabilistic nature, there are probably more ways to evaluate this and **spoiler - being extra cautious in a time like this is the right call. We're going to use a schedule**, this is just a learning exercise to examine my thought processes and explain why my gut reaction was initially adverse to the schedule.

Let's start with the basics, we have 14 days to pack up and 24 hours in each day.

That's 14 \* 24 = 336 hours  
20,160 minutes (336 \* 60)

How long does it take someone to move? This is definitely a flexible metric, some people will have more stuff and some less. I picked a half hour out of thin air. It seems reasonable, it's just a workspace right?

20,160 / 30 = 672

We have 672 half hour chunks to work with.

My co-workers are rationale people though and probably are not going to be burning the midnight oil to pack their workspaces up. Let's remove between 10pm and 6am from our time slots. That gives us only 16 hours in a day.

224 hours (14 \* 16)

(224 \* 60) / 30 = 448 available half hour chunks

What are the odds that two people will pick the same half hour chunk to pack up?

This is essentially a [birthday problem](https://en.wikipedia.org/wiki/Birthday_problem){:target="\_blank"}. What is the likelihood that N randomly chosen people will have the same birthday. Or in another way, what is the likelihood that 12 people will choose the same number out of 448 numbers. The choosing is completely independent, every participant is free to choose any half hour chunk they like.

Solving the birthday problem is a basic statical function that requires some contrarian thinking. We're going to solve the likelihood that two people do not pick the same half hour and then work from there. This is called [the complement](https://en.wikipedia.org/wiki/Complementary_event){:target="\_blank"}. We'll subtract the probable space (100%) from the number of times people did not pick the same chunk to pack up. That will result in the percentage that two people do pick the same number.

If someone chose one half hour chunk out of 448, that leaves 447 chunks that other people could attend.

X = (447/448)

Y = 1 - X

or

1 - (447/448) = 0.0022321428571429047

That's a great sign, two people randomly choosing the same time period to move is pretty unlikely.

This is the odds of a unique pair, but when you start saying what are the chances that two of the twelve people pick the same chunk, the odds become more likely.

The number of pairs of chosen chunks will be

66 = (12 \* 11)/2

Our chances of 66 unique pairs is

U = (Chance of unique pair)^pairs
or
87.09% = (0.9977678571)^66

The chance that 2 people out of 12 select the same chunk is actually another complement
12.91% = 1 - 87.09%

Although the chance of two people randomly selecting the same half hour to move their stuff is quite low, the chances increase dramatically when we start adding people to the mix. There's almost a 13% chance that two people will choose the same half hour chunk with twelve people involved.
It would only take ~27 people in an office to make it statically likely (>50%) that two people would show up in the same half hour to move.

Two people in one office is probably fine with social distancing but what are the probabilities when we start increasing the number of people in the office at the same time? This where things get weird.

What are the odds that three people will pick the same half hour chunk to pack up?

We're going to use [Poisson approximation](https://en.wikipedia.org/wiki/Poisson_distribution){:target="\_blank"}. A Poisson distribution can answer the question, how many times will X occur? Checking every triple and calling it X or a "success" if all three picked the same number. The total number of successes is approximately Poisson with mean value:

(12/3)/478^2 = 0.0009628682971

[WolframAlpha Equation](https://www.wolframalpha.com/input/?i=choose%2812%2C3%29%2F478%5E2%5D){:target="\_blank"}

Why 478^2? Take the three random people one at a time. The first one picks a time, say May 19th, 1:00pm. The chance that the second person picks the same time is 1/478, and the chance that the third person also picks that same time is 1/478. Multiplying these gives 1/478^2.

You would need to have around ~90 employees before it's statically likely that three people would choose the same time.

[WolframAlpha Equation](https://www.wolframalpha.com/input/?i=choose%2890%2C3%29%2F478%5E2%5D){:target="\_blank"}

Probability of all 12 employees picking the same half hour chunk to pack:

1/478^12 = 7.0285419e-33

### Conclusion

The odds that two people out of twelve go to the office at the same time are small but still higher than one would expect at 12.91% with twelve in office employees. Those odds increase more and get astronomical when you start adding more people being in the office at the same time. This model could be better, there will be decaying likelihoods, when one person goes into the office, they would be removed from the pool. Also, half hour chunks are not going to fall into a uniform distribution, some hours of the day will be more popular than others. This exercise was just to track down my thought process.

**To state it again, it is better to be overly cautious in times like these, using a schedule to make sure multiple people are not in the office at the same time is a great way to practice being socially responsible**, especially if you have the luxury to do so. I just wanted to track down my gut level instinct and explain it in probabilistic thinking terms.

Further Reading / References

[Thinking in Bets by Annie Duke - Affiliate Link](https://amzn.to/2yWSgKI){:target="\_blank"}

[Numerical^2 iPhone App - Way Better Calculator](https://apps.apple.com/us/app/numerical/id804548449){:target="\_blank"}

[Better Explained - Understanding the Birthday Paradox](https://betterexplained.com/articles/understanding-the-birthday-paradox/){:target="\_blank"}
