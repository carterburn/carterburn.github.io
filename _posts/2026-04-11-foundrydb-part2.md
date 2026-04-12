---
title: "Part 2 - Writing the Lexer for fdb"
layout: single
tags:
    - database
    - sqlite-series
    - foundrydb
---

Now that we have a simple to use REPL from [part 1](/foundrydb-part1), we need
to start supporting actual SQL syntax. As briefly touched on in the intro, the
goal here is to write a lexer and parser for SQL then generate bytecode to run
in the VM. After some thought between that first post and this one, we'll
actually end up writing what's known as a tree-walking interpreter because it's
a) easier to understand in code and b) enough of what we need in terms of
performance. If we were trying to really squeeze out some serious performance,
we'd want the VM, but we'll be handling extremely simple queries with fdb. 

Similarly, in the edit in [part 1](/foundrydb-part1), I introduced another blog
post that will serve as some inspiration for `fdb`, Phil Eaton's SQL database in
Zig using RocksDB as the storage layer. The bulk of that post deals with lexing
and parsing and that's the main inspiration for this departure from cstack's
original series. Zig and Rust have a lot of similarities but also some
differences that we'll see when writing this lexer (hint: the borrow checker is
the main culprit here).

At a high level, the goal for a lexer (and this post) is to take the raw input
string we receive from the user at the `fdb > ` prompt we created in [part
1](/foundrydb-part1) and chunk it up into meaningful tokens to pass to a parser.
For the most part, lexing doesn't really think *too* hard about the characters
and their meaning but just tries to group things together that make sense (like
strings, numbers, etc.) and will also identify keywords or "reserved" words that
have special meaning in the language (such as SELECT or WHERE in SQL). 

## Tokens
The first thing we will define is the base type: the `Token`. The `Token` is
going to be the thing we extract out of the input text and how we convey that
this chunk of input has some "meaning." A few goals with the `Token`: 

1. We want to be able to assign the meaning and access that easily. We'll call
   that `TokenKind`.
2. We also want to track where the `Token` exists in the input and the actual
   raw text that makes up the `Token`. To do that, we'll have a reference back
to the original input (giving a lifetime parameter to our `Token` struct; yay).

Here's what the `Token` looks like (we'll place this in `src/token.rs` and be
sure to add `pub mod token;` to `src/lib.rs`):

```rust
#[derive(Debug)]
pub struct Token<'a> {
    /// The type of Token this is
    pub kind: TokenKind,

    /// The actual text this Token refers to (lexeme in compiler terms)
    pub lexeme: &'a str,

    /// The position in the broader input this Token starts at
    pub position: usize,
}
```

We'll derive `Debug` so we can print it out after lexing during this part.
Notice how the lifetime of `Token` is tied to the lifetime of the `&str` that
stores the actual text. What this means is that the `Token` has to live as long
as the original input lives. If the original input `&str` goes away, we can't
keep around a `Token` that is referring to it. That shouldn't be a problem
because we'll have some input, lex it, parse it, process it, and then we'll
discard the input and all the things we used to process that input. 

The `TokenKind` is an `enum` that describes the type or meaning of the `Token`.
This is really the most important part for the parser. The parser will approach
the problem like: "I just saw a Create Token, so now I expect a Table Token; if
I don't then that is an error". Here's what it looks like:

```rust
#[derive(Debug)]
pub enum TokenKind {
    // Keywords
    Create,
    Table,
    Insert,
    Into,
    Select,
    Values,
    From,
    Where,

    // Operators
    Plus,
    Equal,
    LessThan,
    LessThanEqual,
    GreaterThan,
    GreaterThanEqual,
    LeftParen,
    RightParen,
    Comma,

    // Literals
    Identifier,
    String,
    Number,

    // EOF marker
    Eof,
}
```

Again, we derive `Debug` to allow us to print the `enum` within the printing of
a `Token`. Next, we define the keywords that this clone is going to support. Of
note, this is a slight departure from Phil Eaton's blog post. I chose to
separate out things like "CREATE TABLE" and "INSERT INTO" as separate `Token`s.
The reason is because technically there are other options (so, extensibility
comes into play here) as well as it is slightly easier to actually lex the input
with just single-word keywords. Otherwise, we'd be trying to do some longest
substring matching with keywords (just like Phil did in his post). I chose to go
with separating them to make the lexing loop a bit easier and to just let the
parser down the line figure out the meaning and semantics.

Then, we define the operators we support. This will be in SELECT statements most
likely when the user may want to get all columns WHERE the id is greater than
some value or if the user wants to apply something like + 30 to a column. The
only departure here is that I added support for "<=" and ">=". 

Next, we have literals. Identifiers are things that aren't strings, but will be
used to identify some piece of the database. Think like column and table names
in this instance. Strings are anything enclosed in single quotes (`'`) and
numbers are...well, numbers. One note that I'm purposefully ignoring: a user
technically could have an unterminated string and that would be accepted just
fine as the 'last' `Token`. That is something we should probably fix but I am
going to leave it for now.

Finally, I added an Eof marker that will be automatically added to the end of
the final list of `Token`'s. I also chose to do this to help support the reading
of files in the future (multi-line SQL "scripts") and because it will make the
parser's job a lot easier.

That's it for `Token`! It's meant to be simple and foundational. We'll start to
produce `Token`'s when we get to the next section.

## Lexer
Now that we have `Token`s, let's start writing the thing that will make them.
Lexer's are surprisingly easy to write. The basic idea is that you maintain your
current position in the input, check what that character is, and decide based on
that character what you should do. Writing a lexer can obviously be more
complicated, but in this case, this is about the extent that we need. So, we'll
first define the `Lexer` struct as just that:

```rust
pub struct Lexer<'a> {
    /// The input string to lex
    input: &'a str,

    /// The current position in the input string we're processing
    current: usize,
}
```

Again, we tie the lifetime of the `Lexer` to the original input telling the
compiler that this struct can't live longer than its input. 

Before we write the main lexing loop, it is nice to add a few helper functions
to allow us to process through the input easier. We'll walk through each of
these one-by-one:

```rust
impl<'a> Lexer<'a> {
    pub fn new(input: &'a str) -> Self {
        Self { input, current: 0 }
    }

    /// Peeks at the next character to process but doesn't consume it
    fn peek(&self) -> Option<char> {
        self.input[self.current..].chars().next()
    }

    /// Advances the cursor and returns the character at the current position
    fn advance(&mut self) -> Option<char> {
        if let Some(next) = self.input[self.current..].chars().next() {
            // grabbed something from the input, advance current
            self.current += next.len_utf8();
            Some(next)
        } else {
            None
        }
    }

    /// Checks if we have exhausted the input
    fn finished(&self) -> bool {
        self.current >= self.input.len()
    }

    /// Provides a slice of the input from a given byte offset up to the current position
    /// (exclusive)
    fn input_slice(&self, from: usize) -> &'a str {
        &self.input[from..self.current]
    }
}
```

First, we have the constructor, classic `new`. All it does is saves off the
input and sets `current` to 0, saying we're starting from the beginning.

`fn peek(&self) -> Option<char>` is one we will use a lot. What this method does is
allows us to see what the **next** character is without actually advancing
`self.current`. This is really valuable and will be used all throughout the
lexing loop. Notice the deliberate use of `.chars().next()`. The reason we can't
just do something like `&self.input[self.current]` is to allow UTF-8 support. If
we were only allowing ASCII characters (and then subsequently enforcing it), we
could use the direct indexing. To support UTF-8, though, we have to use
`chars()` so that we get the entire `char` which could be anywhere between 1 and
4 bytes. The `char` type in Rust supports UTF-8, so we have to respect it.

`fn advance(&mut self) -> Option<char>` is basically the same thing as `peek`
but now we are also advancing the internal cursor. Ironically, I never use the
`char` that comes out of the `Option` in the simple lexing loop we have but its
a good thing to keep around in case we need it in the future.

Finally, we have two more helpers `fn finished(&self) -> bool ` and 
`fn input_slice(&self, from: usize) -> &'a str`. 
`finished` is a simple check to see
if we have processed all of the input. `input_slice` is a helper to just get a
slice from a provided value up to the current cursor position. Don't worry about
the lifetime added to the return value of the method, it actually isn't needed.
You'll notice that we had to be explicit with the lifetime on the return value
of `input_slice`. At first, I didn't catch this, but lifetime elision caused a
compile-time error. Originally the signature was 
`fn input_slice(&self, from: usize) -> &str` but the compiler with lifetime
elision will assign the return value's lifetime as the same as the single
reference in the paramters (see [chapter
3](https://tfpk.github.io/lifetimekata/chapter_3.html) of LifetimeKata for a
good explanation on how the compiler automatically assigns lifetimes). What this
communicated was that the string slice needed to have the same lifetime as
`self` but that isn't actually correct. The returned string slice is the same
lifetime (lives as long as) the input string stored **within** `self`, so we
needed to assign that lifetime. Remember `'a`'s lifetime on `Lexer` is tied to
the `&str` that is stored within it, so when we refer to `'a`, we are referring
to the lifetime of that string slice, which is what we're slicing in
`input_slice`.

Once we have the helpers, we can start writing the frame of the main lex loop.
Here is some pseudocode of how it will look:

```txt
while !finished()
    while input[index] is whitespace
        advance()
    
    next = peek()
    if next is alphabetic
        lex_word()
    else if next is digit
        lex_number()
    else if next is '
        lex_string()
    else 
        lex_operator()

return tokens
```

Basically, we'll keep processing until we're out of characters in the input
string, eat any whitespace that we encounter, check the next character and
decide what kind of `Token` we'll try to lex. At the end, we return the
`Token`'s. Let's first tackle the loop and we'll do the individual `lex_*` 
methods later. Here's the pseudocode transformed to Rust. Note that this is
within the larger `impl` block for the `Lexer<'a>` struct.

```rust
    /// The brains of the lexer to actually lex out a sequence of Token's
    pub fn lex(&mut self) -> Result<Vec<Token<'a>>, LexerError<'a>> {
        let mut tokens = Vec::new();

        while !self.finished() {
            // eat whitespace
            while let Some(c) = self.peek() && c.is_whitespace() {
                self.advance();
            };

            // peek at the next character to choose our path
            let Some(c) = self.peek() else {
                // break if there is somehow no more
                break;
            };

            if Self::is_alphabetic_with_underscore(c) {
                tokens.push(self.lex_word());
            } else if c.is_numeric() {
                tokens.push(self.lex_number());
            } else if c == '\'' {
                tokens.push(self.lex_string());
            } else {
                // anything else is either an operator or an invalid character
                tokens.push(self.lex_operator(c)?);
            }
        }

        // push final Eof token
        tokens.push(Token { kind: TokenKind::Eof, lexeme: "", position: self.input.len() });

        Ok(tokens)
    }
```

The only thing that may look slightly different is the addition of the Eof
`Token` at the end of the method. Once we've exhausted the input, we've hit the
enf of the input and will add that end of file `Token`. You may also recognize
the return type of the method is `Result<Vec<Token<'a>>, LexerError<'a>>`.
`Vec<Token<'a>>` is the list of `Token`'s we've lexed and `LexerError<'a>` is a
custom error type that we'll talk about later. This could easily have been a
`std::io::Error` that has a simple `String` to explain the error but you'll see
we can give a lot better context with a custom error. The
`is_alphabetic_with_underscore` is a very simple associated method with
`Lexer<'a>` that looks like this:

```rust
    /// A more comprehensive "alphabetic" check
    #[inline(always)]
    fn is_alphabetic_with_underscore(c: char) -> bool {
        c.is_alphabetic() || c == '_'
    }
```

It just makes the condition simpler in the lex loop, but we easily could have
written this. That's why I chose to annotate `#[inline(always)]` to strongly
suggest to the compiler that this should be inlined (but it probably would have
been anyway without my hint).

Again, this loop isn't particularly special or complex; the complexity comes in
the individual `lex_*` methods, so let's dive into `lex_word()` first.

## Lexing Methods 
`lex_word` looks like this:

```rust
    /// Lex a word after peeking that it is alphabetic. The first character of the word should have
    /// been 'peeked' but will be advanced on behalf of the caller.
    fn lex_word(&mut self) -> Token<'a> {
        let start = self.current;
        self.advance();
        while let Some(ch) = self.peek() && Self::is_alphanumeric_with_underscore(ch) {
            self.advance();
        }
        let lexeme = self.input_slice(start);
        // if the word is a keyword, a fully formed Token is provided otherwise we construct one as
        // an identifier
        Self::check_keyword(start, lexeme).unwrap_or(Token {
            kind: TokenKind::Identifier, lexeme, position: start,
        })
    }
```

We know that `self.current` points at something that looks like a word (its an
alphabetic character or an underscore). Next, what we'll do is continue to
advance the cursor while we keep seeing alpha*numeric* characters or an
underscore (in SQL, you are allowed to have an identifier with a number as along
as its not the first character). Once we do that, we'll slice the lexeme (the
actual word) and then check to see if it is a keyword with
`Self::check_keyword`. If it is, that associated function will return a new
`Token` with the keyword's `TokenKind` otherwise when it returns `None`, our
`lex_word` method will unwrap and provide the default as a `Token` with
`TokenKind::Identifier` with the word.

Here is `check_keyword`:

```rust
    /// Checks if a alphanumeric string is a SQL keyword
    fn check_keyword(start: usize, lexeme: &'a str) -> Option<Token<'a>> {
        let normalized = lexeme.to_lowercase();
        let mut token = Token { kind: TokenKind::Create, lexeme, position: start};

        token.kind = match normalized.as_str() {
            "create" =>  TokenKind::Create,
            "table" => TokenKind::Table,
            "insert" => TokenKind::Insert,
            "into" => TokenKind::Into,
            "select" => TokenKind::Select,
            "values" => TokenKind::Values,
            "from" => TokenKind::From,
            "where" => TokenKind::Where,
            _ => {
                // anything else will be treated as an identifier
                return None;
            }
        };

        Some(token)
    }
```

We normalize to all lowercase to support case insensitivity. This method has another lifetime 
learning moment. Originally, I had `check_keyword(&self, start: usize, lexeme:
&'a str) -> Option<Token<'a>>` but notice I never actually access anything in
`self`. So, the compiler didn't like that because all of the `lex_*` methods
take the same `&mut self` in `lex()` except this one. This one created an
immutable reference within the loop and that means we had an immutable reference
along with a mutable one (the other `lex` methods). That's not allowed in Rust,
so I had to think a bit more. It turns out I really didn't have a need to take
any reference to `self` because I wasn't using it, I was just referring to the
start position and actual lexeme we already grabbed, so I could keep the `Token`
lifetime to the input string (same lifetime that is tied to the `lexeme`
argument) and make this an associated method. **The takeaway**: don't take a
reference to `self` unless you need it. If you need it to be called in a method
that already takes a mutable reference for `self`, any methods called within
that method also need to have a mutable reference. I didn't do that originally 
and learned the hard way :). The assoicated functio was the right fix and we
compiled just fine.

Next up is `lex_number` and `lex_string`. They are both very similar, we just
keep taking until we find something that isn't a number (in the number case) or
the terminating single quote to terminate the user's string. I'll note again
that we don't enforce there is one but we'll keep it simple for now and not do
it. It would be easy in `lex_string` to do, we just keep going until we hit the
single quote and if we run out of characters (exhaust the input), then we would
just return an error. 

Here are the methods, they shouldn't need too much explanation:

```rust
    /// Lex a number. The first chracter should have been 'peeked' but will be advanced on behalf
    /// of the caller.
    fn lex_number(&mut self) -> Token<'a> {
        // lex a number until we can't
        let start = self.current;
        self.advance();
        while let Some(ch) = self.peek() && ch.is_numeric() {
            self.advance();
        }
        let lexeme = self.input_slice(start);
        Token { kind: TokenKind::Number, lexeme, position: start }
    }

    /// Lex a string. The first quote should have been 'peeked' but will be advanced on behalf of
    /// the caller.
    fn lex_string(&mut self) -> Token<'a> {
        // lex a string until another '
        let start = self.current;
        self.advance();
        while let Some(ch) = self.peek() && ch != '\'' {
            self.advance();
        }
        // now we are at the ', so slice from one past the quote then advance
        let lexeme = self.input_slice(start + 1);
        self.advance();
        Token { kind: TokenKind::String, lexeme, position: start + 1 }
    }
```

Finally, we have `lex_operator`. This one is fairly straightforward, it is just
a bit long and has some gotchas. Here it is:

```rust
/// Try to lex an operator with the 'peeked' character, 'c'. This could return an Error if an                                                                                                            [41/1827]
    /// invalid character that isn't an operator is encountered.
    fn lex_operator(&mut self, c: char) -> Result<Token<'a>, LexerError<'a>> {
        Ok(match c {
            '+' => {
                let start = self.current;
                self.advance();
                Token {kind: TokenKind::Plus, lexeme: self.input_slice(start), position: start }
            },
            '=' => {
                let start = self.current;
                self.advance();
                Token {kind: TokenKind::Equal, lexeme: self.input_slice(start), position: start }
            },
            '<' => {
                let start = self.current;
                self.advance();
                let kind = if let Some(ch) = self.peek() && ch == '=' {
                    self.advance();
                    TokenKind::LessThanEqual
                } else {
                    TokenKind::LessThan
                };
                Token {kind, lexeme: self.input_slice(start), position: start}
            },
            '>' => {
                let start = self.current;
                self.advance();
                let kind = if let Some(ch) = self.peek() && ch == '=' {
                    self.advance();
                    TokenKind::GreaterThanEqual
                } else {
                    TokenKind::GreaterThan
                };
                Token {kind, lexeme: self.input_slice(start), position: start}
            },
            '(' => {
                let start = self.current;
                self.advance();
                Token {kind: TokenKind::LeftParen, lexeme: self.input_slice(start), position: start }
            },
            ')' => {
                let start = self.current;
                self.advance();
                Token {kind: TokenKind::RightParen, lexeme: self.input_slice(start), position: start }
            },
            ',' => {
                let start = self.current;
                self.advance();
                Token {kind: TokenKind::Comma, lexeme: self.input_slice(start), position: start }
            },
            _ => {
                // anything else, we have hit an error at this spot
                //return Err(std::io::Error::new(std::io::ErrorKind::InvalidInput, format!("Invalid character at position {}: {}", self.current, c)));
                return Err(LexerError { input: self.input, position: self.current, character: c });
            }
        })
    }
```

The main gotcha is the `<`/`<=` and `>`/`>=`. If `c` is one of the starters, we
have to `peek()` to see if the next character is `=`. If it is, we consume it
and emit the corresponding `Token`. This is a gotcha based ont eh order, you
first have to advance, peek, and then advance if and only if the peeked
character was `=`.

You'll also notice the commented out `std::io::Error`. This is the original
design without a custom `Error` type. As mentioned, this is a simple `String`
based error that definitely works, but doesn't give a whole lot of context. That
is left in now just to see what it would look like. Next, we'll dive into
`LexerError` to see how we can take this simple error and provide some context. 

## Custom Error
As noted in the introduction, we're doing our absolute best to avoid bringing in
any dependencies. Normally, we would want to use something like `miette` or
`thiserror` or `anyhow` to help with `Error` types. `miette` is specifically
designed for the types of errors that compilers typically emit: ones with
context around the error and allows you to produce helpful reports. We'll try to
make an error type that sort of mimics that. The goal with our custom error is
going to give some context to the error within the input.

To do that, we'll start with the struct definition. We'll need to keep the
input string slice and the position the error occurred. We'll also include the
character to make the error reporting simple:

```rust
#[derive(Copy, Clone, Debug)]
pub struct LexerError<'a> {
    /// The original input
    input: &'a str,

    /// The position in the original input
    position: usize,

    /// The character that caused the error
    character: char,
}
```

Again, we take the lifetime of the input saying that the `LexerError` has to live as
long as the input string. Not much different than the `Token` or `Lexer`
structs. The key will be the `Display` implementation for `LexerError`, that's
where we'll make it useful to the user what caused the error and where:

```rust
impl<'a> Display for LexerError<'a> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        let padding = " ".repeat(self.position);
        writeln!(f, "Syntax error: {}", self.input)?;
        writeln!(f, "              {}^", &padding)?;
        writeln!(f, "              {}|", &padding)?;
        writeln!(
            f,
            "              {} ------- Invalid character ({})",
            &padding,
            self.character
        )
    }
}
```
We end up using a nice little arrow to point right in the input where the error
occurred. Granted, this won't be perfect if we have multiline input, but it'll
work as a simple replacement for miette at this stage in the game. We'll see an
example later. 

Notice back in `lex_operator` that we create this error type when we encounter a
character that we don't recognize. It's important to note that this is the
"bottom", we've tried to lex everything before that and didn't have a match, so
we failed to get a lex and we emit the error.

Let's get it into the REPL and see!

## Hooking it Up
First things first, we'll fix up the REPL loop to clearly delineate between the
"." commands and regular SQL syntax. If the input string starts with a dot,
we'll process it in a separate function, otherwise we attempt to lex the SQL
syntax.

Here's the updated reading in the REPL loop.

```rust
match stdin.read_line(&mut buffer) {
    Ok(0) => {
        // EOF; just return (no error)
        return Ok(());
    }
    Ok(_n) => {
        if buffer.trim().starts_with(".") {
            process_dot_command(buffer.trim());
        } else {
            process_command(buffer.trim());
        }
    }
    Err(e) => {
        eprintln!("Error while reading input: {e:?}");
        return Err(e);
    }
}
```

And here's the updated `process_dot_command` which is basically
`process_command` from the last post:

```rust
fn process_dot_command(cmd: &str) {
    let cmd = cmd.to_lowercase();
    match cmd.as_str() {
        ".exit" => {
            // Exit with success
            eprintln!("Goodbye!");
            std::process::exit(0);
        }
        _ => {
            println!("Unrecognized command: {cmd}");
        }
    }
}
```

And finally, the use of our new lexer which will just lex the input and print
out the `Vec<Token<'a>>` returned from the `lex` method:

```rust
fn process_command(input: &str) {
    let mut lexer = Lexer::new(input);

    let tokens = match lexer.lex() {
        Ok(t) => t,
        Err(e) => {
            println!("{e}");
            return;
        }
    };

    println!("Tokens processed: {tokens:?}");
}
```

Our next step will be to pass `tokens` to the parser to get an abstract syntax
tree (basic one for us!) and then interpret that AST to run it against the
database.

Here's me taking it for a spin with a decent chunk of the tokens we support. And
check out the pretty printed error!

```bash
$ cargo run
fdb > create table users (column1 int, column2 varchar(255))
Tokens processed: [Token { kind: Create, lexeme: "create", position: 0 }, Token { kind: Table, lexeme: "table", position: 7 }, Token { kind: Identifier, lexeme: "users", position: 13 }, Token { kind: LeftParen, lexeme: "(", position: 19 }, Token { kind: Identifier, lexeme: "column1", position: 20 }, Token { kind: Identifier, lexeme: "int", position: 28 }, Token { kind: Comma, lexeme: ",", position: 31 }, Token { kind: Identifier, lexeme: "column2", position: 33 }, Token { kind: Identifier, lexeme: "varchar", position: 41 }, Token { kind: LeftParen, lexeme: "(", position: 48 }, Token { kind: Number, lexeme: "255", position: 49 }, Token { kind: RightParen, lexeme: ")", position: 52 }, Token { kind: RightParen, lexeme: ")", position: 53 }, Token { kind: Eof, lexeme: "", position: 54 }]
fdb > insert into users (column1, column2) VALUES (2, 'carter')
Tokens processed: [Token { kind: Insert, lexeme: "insert", position: 0 }, Token { kind: Into, lexeme: "into", position: 7 }, Token { kind: Identifier, lexeme: "users", position: 12 }, Token { kind: LeftParen, lexeme: "(", position: 18 }, Token { kind: Identifier, lexeme: "column1", position: 19 }, Token { kind: Comma, lexeme: ",", position: 26 }, Token { kind: Identifier, lexeme: "column2", position: 28 }, Token { kind: RightParen, lexeme: ")", position: 35 }, Token { kind: Values, lexeme: "VALUES", position: 37 }, Token { kind: LeftParen, lexeme: "(", position: 44 }, Token { kind: Number, lexeme: "2", position: 45 }, Token { kind: Comma, lexeme: ",", position: 46 }, Token { kind: String, lexeme: "carter", position: 49 }, Token { kind: RightParen, lexeme: ")", position: 56 }, Token { kind: Eof, lexeme: "", position: 57 }]
fdb > insert into users (column1, column2) values (3, 'john')
Tokens processed: [Token { kind: Insert, lexeme: "insert", position: 0 }, Token { kind: Into, lexeme: "into", position: 7 }, Token { kind: Identifier, lexeme: "users", position: 12 }, Token { kind: LeftParen, lexeme: "(", position: 18 }, Token { kind: Identifier, lexeme: "column1", position: 19 }, Token { kind: Comma, lexeme: ",", position: 26 }, Token { kind: Identifier, lexeme: "column2", position: 28 }, Token { kind: RightParen, lexeme: ")", position: 35 }, Token { kind: Values, lexeme: "values", position: 37 }, Token { kind: LeftParen, lexeme: "(", position: 44 }, Token { kind: Number, lexeme: "3", position: 45 }, Token { kind: Comma, lexeme: ",", position: 46 }, Token { kind: String, lexeme: "john", position: 49 }, Token { kind: RightParen, lexeme: ")", position: 54 }, Token { kind: Eof, lexeme: "", position: 55 }]
fdb > select column1 from users
Tokens processed: [Token { kind: Select, lexeme: "select", position: 0 }, Token { kind: Identifier, lexeme: "column1", position: 7 }, Token { kind: From, lexeme: "from", position: 15 }, Token { kind: Identifier, lexeme: "users", position: 20 }, Token { kind: Eof, lexeme: "", position: 25 }]
fdb > select column1, column2 from users where column1 < 3
Tokens processed: [Token { kind: Select, lexeme: "select", position: 0 }, Token { kind: Identifier, lexeme: "column1", position: 7 }, Token { kind: Comma, lexeme: ",", position: 14 }, Token { kind: Identifier, lexeme: "column2", position: 16 }, Token { kind: From, lexeme: "from", position: 24 }, Token { kind: Identifier, lexeme: "users", position: 29 }, Token { kind: Where, lexeme: "where", position: 35 }, Token { kind: Identifier, lexeme: "column1", position: 41 }, Token { kind: LessThan, lexeme: "<", position: 49 }, Token { kind: Number, lexeme: "3", position: 51 }, Token { kind: Eof, lexeme: "", position: 52 }]
fdb > select $ from users
Syntax error: select $ from users
                     ^
                     |
                      ------- Invalid character ($)

fdb >
```

For the entire repo at this stage in the game, checkout [this commit on
Github](https://github.com/carterburn/foundrydb/tree/01dedc45f3136b01800e39ad681bc392e80c0347). 
That will give you the exact code as I see it after this post.

That's it for now! Next up is parsing! See you soon.

## Series
This section contains the order of the series for easier navigation.

| Previous Article | Next Article |
|:----------------:|:------------:|
| [Part 1 - SQLite Introduction and Setting up the REPL in fdb](/foundrydb-part1) | **Coming Soon!** |
