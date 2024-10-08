---
title: "Building a Recursive Descent Parser in Rust"
date: 2023-03-14T16:51:54-06:00
draft: false
cover: '/images/crab.jpg'
---

Before I began my journey in learning Rust, I had been warned of its steep learning curve. Undeterred, I started following through some trivial examples in Rust's (incredible) documentation. Things seemed intuitive, I really wasn't seeing the difficulty people were warning me about.

That was until I tried something less trivial. I decided to build a simple recursive descent parser in Rust, thinking I should be able to have something thrown together in an afternoon. Instead, after three or four complete rewrites and dozens of tabs of documentation, I had spend about half a week to get a bare minimum parser thrown together.

Let me tell you more about the project. You can look at the code here on [Github](https://github.com/jacksontheel/rust-recursive-descent-parser).

## My implementation

### The game plan

What is this parser going to be parsing, exactly? Here's the grammar I implemented. It supports sums or differences of as many numbers as you can string together. 

```
expr        : fact addOperator expr
            | fact
fact        : sign num
            | sign '(' expr ')'
sign        : '-'
            | ε
addOperator : '+'
            | '-'
```

For example, this parser supports input like `3 + 2 - 1` or `3 - (2 + 3 - 1)` or even `1 - -(-(2))`. Yes, when I say "recursive descent parser" I suppose I mean "calculator that only supports addition and subtraction," but building out this simple parser should give us the infrastructure to easily expand upon it (which we will later).

### The Scanner

The first piece to build is a struct for processing the input string. I need to be able to match what's next in the input to some arbitrary token, like "+" or "-". I also need to be able to take a number from the input string and parse it as an actual integer.

Here's what my Scanner struct looked like, and its associated function signatures.

```rust
pub struct Scanner {
    tokens: Vec<char> 
}

// Plain old constructor
pub fn new_parser(tokens: Vec<char>) -> Parser

impl Scanner {
    // Not public. Used internally by the two match
    // functions below so that valid input is more flexible.
    // Takes tokens off of the start of the token vector until it
    // reaches one that isn't whitespace.
    fn scan_whitespace(&mut self)

    // Checks if the next unread token matches some literal. "+", "-",
    // "(", or ")"
    // Returns true and takes the matched token out of the vec if there's a match
    // Returns false and does not affect the vec otherwise
    pub fn match_token(&mut self, literal: &str) -> bool

    // Checks if the next unread token is a number
    // Returns the number in an option and removes it from the vec if so, 
    // otherwise returns None and does not affect the vec
    pub fn match_number(&mut self) -> Option<i32>
}
```

### The Parser

Now here's where the magic happens. The parser is going to be a struct of its own. It will own a scanner and iterate through the input as it constructs a final value. Each of the functions parsing a piece of the grammar should return a Result. The Result will either be a value in the form of an `i32` (Rust's 32-bit integer type) or an error in the form of a `str` reference. The error should be some indication of how the parser failed to produce a result from the input string.

```rust
pub struct Parser {
    scanner: scanner::Scanner
}

// Plain old constructor
pub fn new_scanner(tokens: Vec<char>) -> Scanner

impl Parser {
    // The one public function of Parser. Recursively evaluates the expression
    pub fn parse(&self) -> Result<i32, &str>

    fn parse_expr(&self, scanner: &mut scanner::Scanner) -> Result<i32, &str>

    fn parse_fact(&self, scanner: &mut scanner::Scanner) -> Result<i32, &str>

    fn parse_sign(&self, scanner: &mut scanner::Scanner) -> sign::Sign

    fn parse_add_operator(&self, scanner: &mut scanner::Scanner) -> Result<add_operator::AddOperator, &str>
}
```

### Main

Not much to say for this one. If you're writing this yourself, you could implement [REPL](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop) to read expressions from the user and print out the values. I just have hard-coded values.

```rust
fn main() {
    let s = "1 + 2 + -(3 + 4)";
    let parser = node::parser::new_parser(s.chars().collect());

    match parser.parse() {
        Ok(v) => println!("{}", v),
        Err(err) => println!("{}", err)
    }
}
```

I've also written unit tests to check for correctness.

## Conclusion

### Lessons learned

I was struggling for quite some time with mutably borrowing a reference to `self` more than once in my Parser functions. I was stuck, each parse function needed to be able to modify the Scanner as tokens were being read, how could I not borrow mutably more than once?

As I was writing my code, I would ask ChatGPT for simple Rust syntax questions, and it had been answering anything I threw at it quite impressively. For a laugh I asked it to refactor my Parser so that it wouldn't borrow mutably more than once anywhere. Time after time, it would spit out code that still had the same issue. At least it was quite apologetic each time I let it know its refactor didn't cut it. If you're a software engineer worried about being replaced by AI, I suggest picking up Rust! You should be safe for at least a while longer.

The wisdom I found online suggested replacing `mut& self` parameters with mutable parameters for only the field that was going to be changed inside the function. So I made each of Parser's functions take an immutable `&self` and a `&mut scanner::Scanner`. To avoid breaking encapsulation I made these functions that take a Scanner reference as a parameter private. And _voila_, no more multiple mutable borrowing.

The result is something I'm happy with. I thought before that my documentation reading and syntax example following was something akin to me climbing the Rust learning curve. I think now that those things were just putting on my hiking boots. Having written my recursive descent parser, I can see I'm still towards the bottom of the mountain, but I really have begun to climb this time.

### Further work

So this parser is obviously pretty bare-boned. What comes next? Probably multipication and division. A grammar supporting multipication and division (as well as addition and subtraction) might look something like this:

```
expr        : term addOperator expr
            | term
term        : fact mulOperator term
            | fact
fact        : sign num
            | sign '(' expr ')'
sign        : '-'
            | ε
addOperator : '+'
            | '-'
mulOperator : '*'
            | '/'
```

With the current infrastructure in place, implementing this shouldn't be too difficult, but I suppose those are famous last words. I leave this as an exercise for the reader.