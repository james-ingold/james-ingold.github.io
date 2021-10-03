---
layout: book
title: Refactoring Improving the Design of Existing Code
year: 2020
month: 3
review: 10
author: James Ingold
---

> As a code base changes its capabilities as most useful software does, we often find our abstraction boundaries shift.
>
> - Martin Fowler

[Refactoring by Martin Fowler (2nd Edition)](https://amzn.to/2Up0BRC){:target="\_blank"} was for me, the best Javascript book to pick up in 2020. While, not billed as a “Javascript” book, it is a book with Javascript examples and probably the best one I’ve read since the [Good Parts](https://amzn.to/3iwnV9w) by sensei Douglas Crockford. Fowler can come off as a highfalutin academic in his writings, but the 2nd Edition of Refactoring is a programmers programming book. It features a solid mix of code and explanation. It gave me names and the conceptual motivations for refactorings that I have learned over the years in the trenches while teaching me new techniques as well. Fowler represents restructuring your code in an easy to read and thought provoking way. One of the key messages from Refactoring is that it is an ongoing process and that you won’t get the code right out of the gates. As you learn more, you refactor the code, representing the evolution of knowledge in the domain. “As a code base changes its capabilities as most useful software does, we often find our abstraction boundaries shift.” You evolve the code as you learn more about the domain. He mentions yagni which means you aren't going to need it in reference to architecture. Yagni is a practical style of incorporating architecture and design into the development process which is more inclined to deal with issues later when they are better understood.

There are some annoyances in the print version. There is some bold pink font that is often blurry which causes a jarring effect when reading. Fowler will refer to topics he’s written about in the past, there are links to these articles in the bibliography. This is not so great in the print version. For example, Command Query Separation is dropped into a topic without a footnote. It would have been simple to provide footnotes with one sentence descriptions but he states in the bibliography that he decided against including definitions in the work. Personally, I would have preferred the footnotes but the point is minor.
Command Query Separation if you were wondering, means any function that returns a value should not have observable side effects.

### Selected Notes:

What is refactoring? The process of changing a software system in a way that does not alter the external behavior of the code yet improves its internal structures. Methodically improve structure without introducing bugs.

The first step in refactoring is a suite of tests.

When refactoring a large function, try to pick out parts that differ from the overall behavior.

Don’t worry about performance, it’s much easier to tune well-factored code. Refactor and then tune code if needed.

Treat data as immutable as possible, mutable data quickly becomes something rotten.

Always leave the codebase healthier then when you found it.

The true test of good code is how easy it is to change.

Refactoring is small steps, keeping the code in a working state and passing tests.

You don’t want a codebase that has lots of speculative flexibility. Instead you want one that responds rapidly to changing needs and is reliable.

Most of a programmers time is spent debugging. When you get a bug report, start by writing a unit test that exposes the bug.

Functions are the “joints” of our software system. The most important element of a joint is it’s name. There’s no right answer to a functions name or parameters but they will get better as the system evolves.

Naming things well is the heart of clear programming. The more widely a thing is used, the better it must be named. For dynamic languages its helpful to put the type in a name like aCustomer.

#### _Code Smells:_

Mysterious Name <br />
Duplicated Code <br />
Long functions <br />
Long parameter list <br />
Global Data - bad when mutable (Paracelsus’s maxim: the difference between a poison and something benign is the dose) <br />
Mutable data - functional programming helps here. <br />
Shotgun Surgery - when changes are all over the place and you have to make a lot of little edits <br />
Speculative Generality - simplicity through experience rather than generality through guesswork. <br />
Large Class <br />
Comments - often used as a deodorant for code smells. When you feel the need to write a comment, first try to refactor the code so that any comment becomes superfluous.

I'll save the bulk of the book which is the actual refactorings but my favorite refactor for Javascript in practice is probably "Introduce Object Parameter":
Replace groups of data items that regularly travel together with a single data structure. Makes the relationship between items explicit. Reduces the size of parameter list.

This is a strong book for programmers to read. From recommending that refactoring begins with a suite of tests, providing basic refactorings and then building on those to describe complex refactors. Fowler provides the pros and cons of each refactor, helping you become a better programmer one refactoring at a time. Highly recommended.

Review: 10/10

[Buy Refactoring: Improving the Design of Existing Code](https://amzn.to/2WXjk5A){:target="\_blank"}

<iframe style="width:120px;height:240px;" marginwidth="0" marginheight="0" scrolling="no" frameborder="0" src="//ws-na.amazon-adsystem.com/widgets/q?ServiceVersion=20070822&OneJS=1&Operation=GetAdHtml&MarketPlace=US&source=ss&ref=as_ss_li_til&ad_type=product_link&tracking_id=jamesingold08-20&language=en_US&marketplace=amazon&region=US&placement=0134757599&asins=0134757599&linkId=e125c50cecc62b05c2d3b762fc04e1db&show_border=true&link_opens_in_new_window=true"></iframe>
