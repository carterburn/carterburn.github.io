---
title: "Part 3 - Writing the Parser for fdb"
layout: single
tags:
    - database
    - sqlite-series
    - foundrydb
---

In [part 2](/foundrydb-part2), we finished writing the Lexer for `fdb`. That
Lexer produced a `Vec<Token<'a>>` that tokenized the input string provided by
the user at the REPL. Now, we want to take those tokens and **parse** them into
an Abstract Syntax Tree (AST) which defines the actual grammatical meaning to
those `Token`'s. ASTs are a recursive data structure that allow us to interpret
the *meaning* of the statement. With the lexer, we had errors appear when
characters we didn't recognize appeared but we didn't pay attention to the order
of characters. Now, with the parser, we will enforce meaning and order and throw
errors when those aren't met. For example, we can't have `SELECT VALUES INTO
users` because `SELECT` _must_ be followed by a comma-separated list of columns,
followed by `FROM`, followed by a table name, and then an optional `WHERE` with
a clause. We'll see this in play in the code.

For the parser, we're keeping it simple. We're using nothing more than the
standard library and keeping our scope to the simplest SQL syntax possible with
only `SELECT`/`INSERT`/`CREATE` statements and even restricting the freedom of
the comparisons in the `WHERE` clause and limiting data types to integers,
`TEXT` (unbounded strings), and `VARCHAR` (bounded strings). We'll write a
recursive descent parser because its the easiest to reason about since we simply
start at the top level of a `Statement` and then try to parse the individual
pieces of that `Statement` recursively. We just do it in a big loop. We also
only parse one at a time because we don't have semicolons or files and just take
one line of SQL from the REPL each time. As inspiration, this parser is heavily
inspired from Phil Eaton's post on the [parser](https://notes.eatonphil.com/zigrocks-sql.html) 
for his Zig DB.

## AST 
First, I'll define the AST shape. Before we even start processing `Token<'a>`'s,
there needs to be something to parse it into. The top level type will be called
`Statement` and just be a simple `enum` of the three statement types we support.
Each of the statements will hold their respective 'pieces' that come together to
make up the whole statement. Here's what it looks like:

```rust
pub enum Statement<'a> {
    Create {
        table: &'a str,
        columns: Vec<ColumnDef<'a>>,
    },
    Insert {
        table: &'a str,
        columns: Option<Vec<&'a str>>,
        values: Vec<Literal<'a>>,
    },
    Select {
        table: &'a str,
        columns: Vec<&'a str>,
        where_clause: Option<Comparison<'a>>,
    },
}
```

For `CREATE` statements, the AST will hold the name of the table to create and
a `Vec` of the column definitions that include the column name and data type.
`ColumnDef` is defined as:

```rust
pub struct ColumnDef<'a> {
    pub name: &'a str,
    pub data_type: DataType,
}
```

And `DataType` serves as the data type of the column which is defined as another
simple `enum`:

```rust
pub enum DataType {
    Int,
    Text,
    VarChar(usize),
}
```

I will point out that a lot of those types (including the top level `Statement`)
hold `&str` references to the original input data. This allows me to use the
super high speed term of "zero copy" meaning that I am not allocating a new
`String` for each of these types and just re-using the original input. Since I
am going to get input, lex it, parse it, execute it, and throw it away before
the next statement from the REPL, I can get away with those references. It also
doesn't really make weird lifetime issues because the `Statement` is going to
live a shorter life than the original input. 

For the `INSERT` statement, it is very similar to the `CREATE` in that the table
name is stored and then an optional comma-separated list of columns. That's why
the `columns` field is wrapped in `Option` because it is technically optional.
I'll parse it if it's there, otherwise I continue parsing the statement. This
statement also has a `Vec<Literal<'a>>` to denote literals like numbers and
strings. While the `TokenKind` delineates those as well, this type defines it as
the parser sees it:

```rust
pub enum Literal<'a> {
    Number(i64),
    String(&'a str),
}
```

Finally, for the `SELECT` statement, there is a table name and a comma-separated
list of columns and then an optional `WHERE` clause to filter the output.
Normally, the columns and the clauses for the `WHERE` would be "Expressions"
(using compiler terms), but because of the simplicity of this clone, I hardcoded
the intent with only allowing comparisons. If it were an expression, it would
become a recursive data type that allowed nesting like `WHERE (col + 3) > 5` but
the focus on this project is far less on parsing SQL and far more on the
internals of a database! So, I kept it simple. Also note that the `where_clause`
is an `Option` because it is not required. The `Comparison` type is defined as:

```rust
pub struct Comparison<'a> {
    pub column: &'a str,
    pub op: ComparisonOp,
    pub value: Literal<'a>,
}
```

And the only unseen type in that definition is the `ComparisonOp` which is
basically just a mapping of the comparison `TokenKind`'s to a "parser" type used
exclusively in the parser:

```rust
pub enum ComparisonOp {
    Eq,
    Lt,
    LtEq,
    Gt,
    GtEq,
}
```

This gives the types that will be used by the actual parser to parse the stream
of Token's into. Before actually writing the parser, I want to introduce the
error type. 

## ParseError
For errors encountered during parsing, it is a little difficult to give the
exact location in the input stream because the lexer sort of "collapsed" that
already. The best I can do is provide the starting point of the `Token` that
caused the error and then give a helpful message. With that in mind, the error
type will hold those two things!

```rust
pub struct ParseError<'a> {
    token: Token<'a>,

    msg: String,
}
```

This is about as simple as it gets. The `Token` will hold where the `Token`
exists in the input as well as the raw value (lexeme) of the `Token` and a nice
contextual message can be printed on errors. These will be used when the parser
expects to see something like `INTO` after `INSERT` but sees an identifier token
or something else. 

For this post, I am going to omit the `Display` implementations to save on
reading. Checkout the full commit found at the end of the post if you're curious
on those implementations. 

## Parser 
It is time to define the `Parser` struct. For this struct, it will take
ownership of the `Vec<Token<'a>>` the lexer produced and then will track a
current position in that `Vec`. It's very simple to setup, just take the `Vec`
from the lexer and set the position to 0. Here's the definition and `new()`
method:

```rust
pub struct Parser<'a> {
    tokens: Vec<Token<'a>>,

    current: usize,
}

impl<'a> Parser<'a> {
    pub fn new(tokens: Vec<Token<'a>>) -> Self {
        Self { tokens, current: 0 }
    }
}
```

Can't get simpler than that. To help the `Parser`, much like the `Lexer`, a few
helper methods will make life easier. I'll put them here and explain each
quickly:

```rust
fn peek(&self) -> Token<'a> {
    self.tokens[self.current]
}

fn advance(&mut self) -> Token<'a> {
    let tok = self.peek();
    self.current += 1;
    tok
}

fn expect(&mut self, kind: TokenKind) -> Result<Token<'a>, ParseError<'a>> {
    let next = self.peek();
    if next.kind == kind {
        Ok(self.advance())
    } else {
        Err(ParseError {
            token: next,
            msg: format!("Expected {:?}; got {:?}", kind, next.kind),
        })
    }
}

fn match_kind(&mut self, kind: TokenKind) -> bool {
    if self.peek().kind == kind {
        self.advance();
        true
    } else {
        false
    }
}
```

`peek()` will just return the current `Token` without moving the `current`
position in the stream of `Token`'s. `advance()` basically does the same thing,
it just actually advances the position. This method was the one that made me
make an executive decision to slap `#[derive(Copy, Clone)]` on `Token` and
`TokenKind`. Without it, there was going to be some gymnastics with lifetimes
and ownerships because the token that is at current needs to first be grabbed
(either by reference or value through `Copy`) and then advance the position.
That is difficult because the `Token` is automatically assumed to be immutably 
borrowed from `&mut self` and then `self` is actually modified through the
`&mut`. It is definitely possible to store references, but in reality it was
easier to derive `Copy` and still holds the Rust rule that a struct under 32
bytes is fine to have `Copy`. The `Token` just holds a `usize` (64-bits on my
system), the `TokenKind` which is probably no more than a `u8` since there are
only 21 variants (checked on the Rust playground, yes that is the size), and a
fat pointer for the `&str` meaning the entire struct is 25 bytes. 

`expect()` is a nice helper that will allow the parser to expect a certain
`Token` and return a `ParseError` if the next `Token` is not of that kind. This
will be helpful when the parser expects, say, a right parenthesis and a comma is
found. The method will advance the position and return the `Token`.
`match_kind()` is similar, but it only advances if the next `Token` matches the
specified `TokenKind` but won't error out if that `Token` is not found. This
will be helpful in situations like parsing a comma separated list when there may
be a comma as the next `Token` but it isn't necessarily an error if the next
`Token` is not a comma (it's just the last element in that comma separated
list). `match_kind()` will be used frequently. 

## Other Helpers
The previous four helpers are the lowest level logic and fairly generic over the
`TokenKind`. Next, I want to introduce two smaller parsers that can be composed
in larger ones. (Note, I didn't reach for `nom` or another parser-combinator
crate even though I was tempted; but I like to take some of the terms from it).
The first is `parse_literal()`; this method will attempt to parse out one of the
`Literal` variants (a Number or String) and return that type, erroring if
something bad happens:

```rust
fn parse_literal(&mut self) -> Result<Literal<'a>, ParseError<'a>> {
    let val_tok = self.peek();
    let literal = match val_tok.kind {
        TokenKind::String => Literal::String(val_tok.lexeme),
        TokenKind::Number => {
            Literal::Number(val_tok.lexeme.parse().map_err(|e| ParseError {
                token: val_tok,
                msg: format!("Error parsing number: {e}"),
            })?)
        }
        _ => {
            return Err(ParseError {
                token: val_tok,
                msg: format!(
                    "Expected a literal (number or string); got {:?}",
                    val_tok.kind
                ),
            });
        }
    };
    self.advance();
    Ok(literal)
}
```

Since this is the first use of some of the helpers, I'll go a little deep.
First, I grab the next `Token` via `peek()` because I want to be able to inspect
the `TokenKind`. Then, matching on the `TokenKind`, I will accept either a
`TokenKind::String` or `TokenKind::Number` and error if there are any other
`TokenKind`'s. For the `TokenKind::String`, it's an easy conversion to the
`Literal` by just using the `Token`'s lexeme. For the `Number` variant, the
first step is to actually parse the number and since parsing can fail, I map
that error into the `ParseError` type or just make the `Literal::Number`. Once
the matching is done, a simple `advance()` to move to the next `Token` is needed
to show this method has fully processed the `Token`. Then, return the new
`Literal`. One call out here is that because of the lexing phase, this method
doesn't have to worry about whitespace or the fact that the `Token` might not be
an actual number or string; the lexer handled that and now this method just
operates on the higher level `Token`.

The other helper is `parse_datatype()` to parse data types in the column
definition of a `CREATE TABLE` statement:

```rust
fn parse_datatype(&mut self) -> Result<DataType, ParseError<'a>> {
        let data_tok = self.peek();
        if data_tok.kind != TokenKind::Identifier {
            return Err(ParseError {
                token: data_tok,
                msg: format!("Expected a data type identifier; got {:?}", data_tok.kind),
            });
        }
        let normalized = data_tok.lexeme.to_lowercase();
        match normalized.as_str() {
            "int" => {
                self.advance();
                Ok(DataType::Int)
            }
            "text" => {
                self.advance();
                Ok(DataType::Text)
            }
            "varchar" => {
                self.advance(); // advance past the data type
                self.expect(TokenKind::LeftParen)?; // left paren needs to be there
                let size_tok = self.expect(TokenKind::Number)?; // need a number

                let data_type =
                    DataType::VarChar(size_tok.lexeme.parse().map_err(|e| ParseError {
                        token: size_tok,
                        msg: format!("Error parsing varchar size: {e}"),
                    })?);
                self.expect(TokenKind::RightParen)?; // final right paren
                Ok(data_type)
            }
            _ => {
                // if it's not, we've exhausted datatypes, so this is an error
                Err(ParseError {
                    token: data_tok,
                    msg: format!("Unknown data type: {}", data_tok.lexeme),
                })
            }
        }
    }
```

This one seems longer, but it is fairly straightforward. For this parsing, there
must be an `TokenKind::Identifier`, so if it is not, the method returns an `Err`
right away. Then, the method normalizes the identifier's lexeme for easier
comparison. On "int" and "text", an easy `advance()` to advance the position in
the parser and then returning the appropriate `enum` variant. For "varchar",
though, there is an additional set of parsing needed to grab the parentheses and
the actual number to define the size of the varchar. It's not necessarily
complicated, but shows that the parser will consume `Token`'s in different ways
based on the `Token` it processes.

## Comma Separated Lists 
In a lot of the parsing of statements (I'll get there soon), there are comma
separated lists. These occur in all three statements that are supported and the
`INSERT INTO` can even have it twice. At first, I wrote the basic loop in each
of the statement's individual methods, but that is obviously bad practice to
repeat code, so I decided to extract it as a separate method.

The issue is that with each list, the inner body of the loop was slightly
different based on which comma separated list I was parsing at the time. The
normal way, then, to hoist that loop out is with a method that accepts a closure
and run the loop that is pretty much the same each time, just doing different
processing based on the comma separated list. 

First, the method signature looks like this:

```rust
fn parse_comma_separated<T>(&mut self, mut parse_item: impl FnMut(&mut Self) -> Result<T, ParseError<'a>>) -> Result<Vec<T>, ParseError<'a>> 
```

The method still takes `&mut self` but it is now generic over `T` because the
types that the comma separated lists could produce are going to be different.
Then, the closure is defined as a function that takes a mutable receiver. In
this case, this will be the parser itself because we want the closure to be able
to call simple things like `self.parse_datatype()`. That closure is going to
return a single `T` (like `DataType` or `Identifier`) wrapped in a `Result` and
then the loop collects those into a `Vec`. Here's what the actual method looks
like:

```rust
let mut items = Vec::new();
loop {
    items.push(parse_item(self)?);
    if !self.match_kind(TokenKind::Comma) {
        break;
    }
}
Ok(items)
```

Pretty simple, right? The method defines a `Vec` to store results in then loops
by calling the closure to parse the "item" that is in the comma separated list.
Then, the break condition is whether or not another `Comma` is seen. This is the
best use of `match_kind`. If there is a comma, then the loop continues and tries
to parse the next item, if not, the loop breaks and the `Vec` is returned. 

## Statement Parsing
Now all of the little pieces are ready for the full `Statement` parsing. The top
level method is just `parse_statement()` seen here:

```rust
fn parse_statement(&mut self) -> Result<Statement<'a>, ParseError<'a>> {
    let nxt = self.peek();
    match nxt.kind {
        TokenKind::Select => self.parse_select(),
        TokenKind::Insert => self.parse_insert(),
        TokenKind::Create => self.parse_create(),
        _ => Err(ParseError {
            token: nxt,
            msg: format!(
                "Expected the start of a statement (SELECT, CREATE, INSERT), found {:?}",
                nxt.kind
            ),
        }),
    }
}
```

Exceptionally simple; just `peek()` at the next `Token` and dispatch to another
helper for each of the `Statement` types based on that `peek()`. I'll post the
code for each of the statements below and quickly talk about them, but they
really just compose the other methods together to parse each type.

```rust
fn parse_select(&mut self) -> Result<Statement<'a>, ParseError<'a>> {
   self.expect(TokenKind::Select)?;
   let columns =
       self.parse_comma_separated(|p| Ok(p.expect(TokenKind::Identifier)?.lexeme))?;

   self.expect(TokenKind::From)?;
   let table_token = self.expect(TokenKind::Identifier)?;

   let where_clause = if self.match_kind(TokenKind::Where) {
       let column = self.expect(TokenKind::Identifier)?;
       let op_tok = self.peek();
       let op = match op_tok.kind {
           TokenKind::Equal => ComparisonOp::Eq,
           TokenKind::LessThan => ComparisonOp::Lt,
           TokenKind::LessThanEqual => ComparisonOp::LtEq,
           TokenKind::GreaterThan => ComparisonOp::Gt,
           TokenKind::GreaterThanEqual => ComparisonOp::GtEq,
           _ => {
               return Err(ParseError {
                   token: op_tok,
                   msg: format!("Expected a comparison operator; got {:?}", op_tok.kind),
               });
           }
       };
       self.advance();

       let literal = self.parse_literal()?;

       Some(Comparison {
           column: column.lexeme,
           op,
           value: literal,
       })
   } else {
       None
   };

   Ok(Statement::Select {
       table: table_token.lexeme,
       columns,
       where_clause,
   })
}

fn parse_insert(&mut self) -> Result<Statement<'a>, ParseError<'a>> {
    self.expect(TokenKind::Insert)?;
    self.expect(TokenKind::Into)?;

    let table_token = self.expect(TokenKind::Identifier)?;

    let columns = if self.peek().kind == TokenKind::LeftParen {
        self.expect(TokenKind::LeftParen)?;
        let cols =
            self.parse_comma_separated(|p| Ok(p.expect(TokenKind::Identifier)?.lexeme))?;
        self.expect(TokenKind::RightParen)?;
        Some(cols)
    } else {
        None
    };

    self.expect(TokenKind::Values)?;
    self.expect(TokenKind::LeftParen)?;
    let values = self.parse_comma_separated(|p| p.parse_literal())?;

    self.expect(TokenKind::RightParen)?;

    Ok(Statement::Insert {
        table: table_token.lexeme,
        columns,
        values,
    })
}

fn parse_create(&mut self) -> Result<Statement<'a>, ParseError<'a>> {
    self.expect(TokenKind::Create)?;
    self.expect(TokenKind::Table)?;

    let table_token = self.expect(TokenKind::Identifier)?;

    self.expect(TokenKind::LeftParen)?;
    let columns = self.parse_comma_separated(|p| {
        let col_tok = p.expect(TokenKind::Identifier)?;
        let data_type = p.parse_datatype()?;
        Ok(ColumnDef {
            name: col_tok.lexeme,
            data_type,
        })
    })?;
    self.expect(TokenKind::RightParen)?;

    Ok(Statement::Create {
        table: table_token.lexeme,
        columns,
    })
}
```

`SELECT` is the most complicated because of the optional `WHERE` so the parser
just checks if the `TokenKind::Where` is there and parses it if it needs to
parse it. `INSERT` also has the optional listing of columns that branches based
on if there is a `TokenKind::LeftParen` after the table name. Notice the use of
`parse_comma_separated` in each method. Sometimes its as simple as calling
`expect()` and getting the lexeme and sometimes it requires getting an
`Identifier` and then a `DataType`. That's the power of the closure, it can be
complicated while using the original `Parser` type's methods and hoist out that
loop.

### Quick Notes on Semantics
It was definitely tempting when writing these statement parsers to start
thinking "I should check that the number of columns in the `INSERT INTO` matches
the number of values" but quickly realized that is not the part of the parser.
The parser cares about the grammar itself - such as "after I parse this I MUST
have a comma or it is invalid syntax". The executor (which will get to in the
future) will take cares of the semantics and make sure everything lines up
correctly. The executor also will ensure that an `INSERT INTO` statement's
values match the schema defined by the table. Also something the parser
shouldn't worry about!

## Display
A common exercise for compilers is to take your AST and then print out the
original syntax. That's what the `Display` implementations do for each of the
`Parser` types; the implementations will just print out their 'original'
version. It's an easy way to ensure that the AST is well formed and understood
and that the AST types properly represent the syntax.

I'll skip adding those here because they're fairly boring, but checkout the code
at the end of the post to look at them. 

## Hooking into the REPL
Now it was as easy as just hooking it into our REPL back in `src/main.rs` to
leverage the input -> lex into `Vec<Token<'a>>` -> parse into AST -> display
"original" input back. The input was not going to use the exact same string
slices as the original and properly capitalized keywords, etc. so it shows that
the AST was successfully parsed. Here's the hook in:

```rust
fn process_command(input: &str) {
    /*
        --- SNIP ---
    */
    println!("Tokens processed: {tokens:?}");
    let parser = Parser::new(tokens);
    match parser.parse() {
        Ok(statements) => {
            println!(
                "Parsed into AST; recreated SQL from the AST: {}",
                statements[0]
            );
        }
        Err(e) => {
            println!("{e}");
        }
    }
}
```

Let's take it for a spin on the same commands given in the [part
2](/foundrydb-part2) post and add a few showing syntax errors! I will spare the
printing of the `Vec<Token<'a>>` but just know it is still in there at this
point. 

```txt
$ cargo run
fdb > create table users (column1 int, column2 varchar(255))
Parsed into AST; recreated SQL from the AST: CREATE TABLE users (column1 INT, column2 VARCHAR(255))
fdb > insert into users (column1, column2) VALUES (2, 'carter')
Parsed into AST; recreated SQL from the AST: INSERT INTO users (column1, column2) VALUES (2, 'carter')
fdb > insert into users (column1, column2) values (3, 'john')
Parsed into AST; recreated SQL from the AST: INSERT INTO users (column1, column2) VALUES (3, 'john')
fdb > select column1 from users
Parsed into AST; recreated SQL from the AST: SELECT column1 FROM users
fdb > select column1, column2 from users where column1 < 3
Parsed into AST; recreated SQL from the AST: SELECT column1, column2 FROM users WHERE column1 < 3
fdb > select column1, column2 from users where
Parsing error near '' (40): Expected Identifier; got Eof
fdb > insert into users values 2, 'carter'
Parsing error near '2' (25): Expected LeftParen; got Number
fdb > create table sessions (column1 int, column2 varchar)
Parsing error near ')' (51): Expected LeftParen; got RightParen
fdb > create table session (column1, column2)
Parsing error near ',' (29): Expected a data type identifier; got Comma
```

As you can see with the last few, the parser successfully caught errors in the
syntax and was able to give ok-ish error messages to help the user figure it
out. It is by no means the best but it works!

For the entire repo after completing parsing, checkout [this commit on
Github](https://github.com/carterburn/foundrydb/tree/ec68592264ccb720d20079e6a2e578c3cded518b)
which is the exact state of the code as I see it.

That's it for now! Next post, I'll add in an 'executor' which will start
allowing for real data and real execution. There will be a schema and an
in-memory table storage. It is just to get our executor working well with
schemas and actually retrieving data. Following that post we'll start to dive
into the nitty gritty details of the on-disk representation! See you next time!

## Series
This section contains the order of the series for easier navigation.

| Previous Article | Next Article |
|:----------------:|:------------:|
| [Part 2 - Writing the Lexer for fdb](/foundrydb-part2) | **Coming Soon!** |
