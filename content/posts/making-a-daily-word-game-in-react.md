---
title: "Making a Daily Word Game in React"
date: 2023-12-02T12:11:59-07:00
draft: false
cover: "/images/crossletters.png"
---

Not interested in reading any of this? You can play Crossletters at [jacksontheel.com/crossletters](https://jacksontheel.com/crossletters) or find the source code on [Github](https://github.com/jacksontheel/crossletters)

## Introduction

There's nothing like being stuck in an airport to get you spending your hard-earned money. A cup of coffee might cost you ten bucks, an ordinary cheeseburger, twenty. Airport vendors know you're without options, so they fleece you for as much as they can.

And when you're sat waiting for your flight, maybe still hours out, you might find yourself spending money for digital goods too. A few years ago I had let my subscription to the NYT's crossword lapse; I enjoyed the games the subscription offered, but didn't have the time to play often enough to make it worth it. At the airport, time was the one thing I was rich in. So I renewed, and got to work.

The crossword is always fun, but I became enraptured with another of the subscription's games: [Spelling Bee](https://www.nytimes.com/puzzles/spelling-bee). In it, you're given five letters and asked to make as many words as you can from them. My entire airport wait flew by as I played. At a certain point, far before my score was given the "genius" adjective, I was at a dead-end. My modest word sleuthing had found what it could. So I took to the internet looking for answers. Each day, the NYT has a page for Spelling Bee that gives some clues relating to the structure of the words you're looking for. There are 12 words starting with "H", 18 starting with "S" and so on. But in the comments section were people writing crossword-like clues to the words I sought after.

These crossword clue comments were unofficial, posted by other players like me, and some of them weren't great. Many practically handed the words to me on a silver platter, others were confusing and vague to the point of being useless. But finishing out the game until I had attained "genius" was satisfying and fun, in a completely different way than Spelling Bee ordinarily is.

I wanted to make a game that captured that fun, without the need for a community-supplied comments section.

## Crossletters

Thus began my development of Crossletters. You're given a set of letters (I went for seven), and given a number of clues. Each clue leads you to a word that uses only those seven letters. The name came after some amount of thinking. My dad was a fierce advocate for the title Cross Scramble. I opted for Crossletters instead because I thought it illustrated what the game was all about. It's smaller than crosswords, and a letter is a smaller unit than a word. Working with a limited set of letters also supported the title well.

### Front-End, the Only End

Many of the projects I start are grand in scale, supported by some back-end. That's where I start. And long before I even get to a point where I'm spinning up a React project as a front-end I lose steam. Coming home to work on a programming project after eight hours of writing software for work isn't always an attractive proposition.

Yet a simple, static site that I could hack away in an evening or two seemed perfect. No need for a backend at all (more on that later)

#### Material UI

Oh, Material UI, my beloved. If the aesthetic of Crossletters seems familiar to you, that's because I tried my best to have each of the elements of the site built off a [React Material UI component](https://mui.com/material-ui/). It's not innovative, but what's familiar should be intuitive, and I wanted my site to be instantly usable for anyone who might visit. Their brainpower should be spent on the puzzle, not on navigation.

#### The Game Editor

Unlike a game like Wordle, I would never be able to publish my site and let it run endlessly for years. Each of my clues needed to be written by something. A year ago, I might have said "by a human" though that's arguably no longer the case. A large language model writing my clues might be feasible, but I still wouldn't trust it to deliver on the creative aspect of the process. A good clue is subtle, maybe has a good pun, and leads the solver to the answer without handing it to them for free.

So, knowing I could only make so many games myself, I decided to dedicate a portion of the site to an editor: a place where a user could make their own puzzles. Obviously the puzzle would then have to be sharable in some way. Initially I was hoping I could encode an entire puzzle in base64 as part of the link, so users could share without me having to maintain a backend where I store them all. It worked, but the links were egregiously long. I was encoding around 15 full sentences, plus a bunch of other data, of course the encoding was long.

{{< figure src="/images/editor.png" alt="The puzzle editor portion of Crossletters" position="center" style="border-radius: 8px;" caption="The puzzle editor portion of Crossletters">}}

When complaining to a coworker of mine about the problem, I compared these ugly long URLs to the URL you would get from a Google search. He asked me, "so, Google can't solve that and you're trying to?" and I realized if I wanted a pretty and sharable URL, I couldn't do it without a backend.

### The Back-End, the other only end

My dream of a static front-end only site was shot. Feature creep had gotten to me. Luckily, all I wanted out of a backend was data storage, so it couldn't be too much work.

I'd played around before with Supabase, a backend-as-a-service with a generous free tier. All I needed was a way to create puzzles, and play them later, so [Supabase](https://supabase.com/) was perfect. With just a little setup, I could connect to my project's database, insert columns, and read them as well. 

So instead of encoding an entire puzzle into a URL, I could store the puzzle with some random 5-letter code, which could be shared and viewed with just that code.

### Feedback and refinement

Over Thanksgiving, I shared the site with my family, hoping for their feedback. The biggest surprise for me was how much they enjoyed creating puzzles of their own. We must have created and played dozens of puzzles over the holiday. This reaction told me I needed to focus more effort on the builder, so I added a few improvements.

#### Filtered word search for the editor

Before, as I was building puzzles, I would determine my 7 letters, and write a regex word search to decide my answers. Finding those words that fit, I would copy and paste them into the builder portion of the website. I didn't expect or want my users to have to do this, though, so I built a filtered-word dropdown for each answer input. Now, a user can build a puzzle seamlessly, no other tools necessary.

{{< figure src="/images/filteredAnswers.png" alt="The possible answers in the builder filtered by what letters are allowed" position="center" style="border-radius: 8px;" caption="Filtered possible answer dropdown">}}

#### Additional clues

Observing my beta testers (the fam), I noticed the unfortunate problem that if they hit a roadblock, there was just no way to get around it without further clues from me directly. I needed a way to give extra clues to the user without being present in the room with them myself.

In traditional crosswords, you can skip a clue and come back to it later, with more information thanks to the crosses. Yeah, you couldn't get 1 down, but you got 1 across, so you know the letter 1 down starts with, too. The clue system I implemented took inspiration from this concept. Every 3 answers you get right gives you access to an additional clue. Use it, and you get the first letter of the word. You can use another clue and get the second letter of a word if you're still stuck.

{{< figure src="/images/clue.png" alt="A crossletters word partially filled with a clue" position="center" style="border-radius: 8px;" caption="A crossletters word partially filled with a clue">}}

#### Virality

I think one of the pivotal features Wordle had that lead to its mass adoption was the score sharing feature. Copying those block emojis for a good score and pasting it into a group chat made the game a community activity.

I implemented something similar for Crossletters. You can share your score, which tells everyone else which answers you got, which you used a clue to solve but eventually persevered, and which extra-difficult answers still evaded you. This also has the added benefit of adding a feeling of risk vs. reward for using clues. You might get it, but the pesky yellow block is going to rear its ugly head in your score.

{{< figure src="/images/score.png" alt="A score dialog from Crossletters" position="center" style="border-radius: 8px;" caption="A score dialog from Crossletters">}}

## Conclusion

As someone with a tendency to start working on larger-than-life projects that are doomed to never be finished, to have this idea and see it through to the end has been an incredibly satisfying experience.

If I have any takeaways it's that an idea is never fully formed from the start. I needed real people using Crossletters, seeing where they were frustrated and where they spent the most time, to focus my efforts in the right place and minimize pain points. Thanks to Mom, Dad, and Jasper for playing Crossletters, I likely would never have finished if not for your interest and support.

You can play Crossletters at [jacksontheel.com/crossletters](https://jacksontheel.com/crossletters) words or find the source code on [Github](https://github.com/jacksontheel/crossletters)