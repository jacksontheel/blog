---
title: "Recycling Old Crossword Puzzles Into Something New"
date: 2024-07-22T18:53:22-06:00
draft: false
cover: "/images/crossword.png"
---

## Introduction

With the growing popularity of daily word games, I threw my hat in the ring this past November and made [Crossletters](https://jacksontheel.com/crossletters/). Each day, there's  a new set of 7 letters, along with clues to lead you to words consisting of just those letters. Creating the game was a lot of fun, as was sharing it with my friends. However, by the time I finished the project I still had two major unresolved problems.

1. By the nature of the game, creating content was a difficult process. I had to come up with all of the clues myself!
2. Since I wrote the clues, I couldn't play the game!

As much as I appreciated my friends and family giving the game a try, I wasn't going to make writing clues a part-time job. So, I wrote 7 sets of clues, enough for a single week, and called it good. I also included a puzzle-builder for those wanting to create and share a puzzle of their own, which helped with the content drought a bit.

Still, Crossletters wasn't something I could passively enjoy each morning, like the New York Times' Connections or Wordle. I needed a way to generate a truckload of content, in such a way that I would be as oblivious to each day's answers as everyone else. In December, I mused on a couple ways to do this, including the use of an LLM to generate puzzles for me. But one idea stuck out.

There exists a repository of historical New York Times crossword data (in JSON format no less) dating back from 1970. I won't link it here, for fear of being responsible for it being taken down, but it shouldn't be difficult to find through the search engine of your choice. If I could process these crossword clue and answer combinations, I should be able to write a script that creates puzzles, which I could feed into my game.

## Can you do that?

This question initially stopped me from pursuing the idea further, back in December of 2023. I wasn't sure about the legality of such a thing.

I decided to go ahead and do it anyway, for the sake of creating. I wanted to do it right, crediting the authors behind each clue. I made sure to never separate an author's name from the clues that they created, and in the final product you can tap the information icon in the corner to see the original authors of each respective clue.

Also, I am not, and never have, made profit off of Crossletters. The game (and this blog, let's be honest) is made for an audience of myself, and maybe my mom.

If you're a lawyer at the New York Times, send me an email and I'll take these puzzles down!

## Creating the database tables

I'll be using Postgres here, as it's the SQL dialect I'm familiar with, and the one I would marry as soon as marriages between humans and relational database management systems are legalized.

The JSON dataset I'm working with is flush with enough information to completely recreate any of the crosswords from scratch. All I'll be needing, though, is the set of clues, answers, and the names of the author.

```sql
CREATE TABLE clue (
  id SERIAL PRIMARY KEY,
  clueText VARCHAR (50) NOT NULL,
  answer VARCHAR (50) NOT NULL,
  author VARCHAR (50) NOT NULL,
  UNIQUE (clueText, answer)
);
```

I've made the combination of `clueText` and `answer` a composite key, because the NYT reuse a whole lot of clues! I can't blame them, in fact it's economical. Certainly nobody remembers a clue from years ago, so why not reuse it? Still, I don't want these duplicates in my database.

Since Crossletters depends on every answer each day being made up of the same subset of 7 letters, I'll also need a table for the allowable letters in an answer. I can populate this at database creation, since it's been a while since the last letter was added to the alphabet. [Wikipedia](https://en.wikipedia.org/wiki/History_of_the_alphabet#Latin_alphabet) suggests it's been about 400 years, when `J` became officially distinct from `I`.

```sql
CREATE TABLE letter (
  character CHAR(1) PRIMARY KEY 
);

INSERT INTO letter (character) VALUES 
('A'), ('B'), ('C'), ('D'), ('E'), ('F'), ('G'), ('H'), ('I'), ('J'), 
('K'), ('L'), ('M'), ('N'), ('O'), ('P'), ('Q'), ('R'), ('S'), ('T'), 
('U'), ('V'), ('W'), ('X'), ('Y'), ('Z');
```

So, each answer will be associated with one to many letters. But each letter will certainly be used by many answers... We have ourselves a many-to-many connection, folks. We're gonna need a junction table, or as Wikipedia likes to call them, an [associative entity.](https://en.wikipedia.org/wiki/Associative_entity)

```sql
CREATE TABLE clueLetterMapping (
    id SERIAL PRIMARY KEY,
    clue_id INTEGER NOT NULL,
    letter CHAR(1) NOT NULL,
    FOREIGN KEY (clue_id) REFERENCES clue(id),
    FOREIGN KEY (letter) REFERENCES letter(character),
    UNIQUE (clue_id, letter)
);
```

That should be enough to get me started. I wrote up [a program](https://github.com/jacksontheel/crosswords-to-crossletters) in Go (my beloved) and populated the database by shaping the original JSON into the above format.

## Cleaning my data

Naively, I thought it would be as easy as that. However, with this first attempt I realized certain clues that absolutely belong in a traditional crossword have no place in Crossletters.

### Across and Down... what?

Take this innocuous clue and answer pair for instance:

```
clue: Part of opus of 20-Across
answer: SPANGLED
```

That's neat and all, but in my data shaping process, I completely remove each individual clue from the greater context of its original crossword. There *is* no 20-Across anymore. If I let this through to my final generated puzzles, people `Across` the world will see to it that i'm taken `Down`.

I made a small change to ensure that my program skips clues which have some number hyphenated with `Down` or `Across`. Interestingly, there's just a couple of times since 1970 that these keywords were lowercase instead, `down` and `across`. I excluded those, too.

### Words I don't know

I'm a smart guy. Some even call me the smartest guy they've ever met. Still, when I first shaped my data into this new dataset, there was an alarming number of answers that weren't words I had ever heard in my life. Worse, there were answers that weren't English.

```
clue: French family member
answer: TANTE
```

*pardon moi?*

I decided I would need to filter out these uncommon and non-English words if I wanted the puzzles to be at all solvable. Filtering this out on the Go side would be time-inefficient -- I would have to check if each answer exists in some gargantuan txt file, which would slow database population down when done hundreds of thousands of times.

Instead, I opted to create a new table in my database, and fill it with what I deemed to be "Legal Answers".

```sql
CREATE TABLE word (
  word VARCHAR(50) PRIMARY KEY
);
```

When I eventually write the query to get my clues, I'll join on this table to make sure the answers included in the query results are "legal." Initially, I populated this table with the Scrabble English dictionary, but that still gave me some funky results.

```
clue: Meadowsweet
answer: SPIRAEA
```

That's certainly not an English word. If you tell me it's a flower I will kill you.

I then switched my set of legal words to be just the [10,000 most common English words](https://github.com/first20hours/google-10000-english). This was a reduction of 96%, but it still left me with plenty of possible answers. This strategy is still imperfect, for example:

```
clue: "I ___ man with..."
answer: META
```

Clearly the answer here is the concatenation of two words: `MET` and `A`. Crosswords do this all the time, but it was something I hoped to filter out of the data set for my purposes. There's no indication in the original JSON data whether or not a clue is a single word or a concatenated set of words, so certain concatenations which happen to also be legal words will slip through the cracks.
## Querying my data

I'll start by showing you this monster of a query. Then, I'll break it down.

```sql
-- Create a temporary table with the allowed letters
WITH allowed_letters AS (
    SELECT 'A' AS character
    UNION ALL SELECT 'B'
    UNION ALL SELECT 'C'
    UNION ALL SELECT 'D'
    UNION ALL SELECT 'E'
    UNION ALL SELECT 'F'
    UNION ALL SELECT 'G'
),
-- Find all clues where the letters are within the allowed subset
clues_with_allowed_letters AS (
    SELECT clm.clue_id
    FROM clueLetterMapping clm
    JOIN allowed_letters al ON clm.letter = al.character
    GROUP BY clm.clue_id
    HAVING COUNT(clm.letter) = (
        SELECT COUNT(*)
        FROM clueLetterMapping clm2
        WHERE clm2.clue_id = clm.clue_id
    )
)
-- Select clues that match the above criteria
SELECT c.clueText, c.answer, c.author
FROM clue c
JOIN clues_with_allowed_letters cal ON c.id = cal.clue_id
-- Filter to only include answers in the english dictionary
JOIN word ON c.answer = word.word;
-- Only allow words of a certain length
WHERE LENGTH(c.answer) >= 4;
```

First, I create a temporary table of the allowed characters I want for a given puzzle. In this example, those letters are A through G, but they'll be different for each puzzle I generate.

Then, I join that temporary table with my `clueLetterMapping` table, filtering out clues whose answers aren't a subset of the 7 legal letters I've allowed.

Then, I select the clue text, answer, and author from my `clue` table, joining it the `clues_with_allowed_letters` table I created earlier in the query.

Finally (and what is probably the only portion of the query that doesn't look arcane) I filter such that I'm only left with clues that have answers that are English words of a certain length. 4 was chosen here because words that are any shorter aren't typically interesting.

## Building the puzzles

By using the query above, I was ready to start building my puzzles. [I wrote a simple program](https://github.com/jacksontheel/crosswords-to-crossletters) which essentially ran this query and generated JSON puzzles in a loop, until it finished popping out 365 of them. For the 7 letters of each puzzle, my program randomly selected 4 consonants and 3 vowels. I chose this because it seemed like that would generate a good possible mix of words.

By generating puzzles sequentially in one program run, I was able to maintain some state to make guarantees about the collection of puzzles.
1. No question-answer pair is repeated throughout the collection.
2. In each individual puzzle, there are no repeated answers.

I had Guarantee 1 in my head from the start. What I didn't anticipate was just how many questions lead to repeated answers. In my first draft of the puzzle generation program, I got a puzzle which included the following questions:

```
clue: Western home
answer: ADOBE

clue: Hacienda material
answer: ADOBE

clue: Hacienda brick
answer: ADOBE

clue: Sun-dried brick
answer: ADOBE
```

Seeing these all together made me laugh, but it certainly didn't make for a good Crossletters puzzle.

Even if you play Crossletters every day for the entire year, you won't be seeing any questions you've seen before. Of course, after that, the same puzzles will loop around again, but who remembers a crossword you did a year ago?

## Conclusion and what comes next

The day this article is published I'll release the update to [jacksontheel.com/crossletters](https://jacksontheel.com/crossletters), which means a year of content! I'll certainly be playing every morning. 

The best part? If you're frustrated about a question being too cryptic, you can be sure I'm frustrated, too. I've shifted any blame onto the New York Times, which means I'm one of you now. We can all be frustrated together.

Suffice to say these questions are a lot harder than the ones I authored myself back in December. So, I've decided to give the user a hint for every single answer they get right, rather than for every three answers. You're welcome!

I doubt that I'll be touching Crossletters again as a developer, unless I'm struck with an exciting idea. If you have such an idea, feel free to make an issue on [the Github page](https://github.com/jacksontheel/crossletters), or fork the project and implement it yourself! Contributions are welcome.

