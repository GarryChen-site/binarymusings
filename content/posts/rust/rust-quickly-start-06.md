---
title: "Build SQL Query Engine in Rust: CSV Parser with Python Bindings"

description: "Create a powerful SQL query engine in Rust that queries CSV files and URLs. Features Python bindings, AST conversion, and 375 lines of code."

summary: "Build a production-ready SQL query engine in Rust with just 375 lines of code. Learn to parse SQL with custom dialects, convert AST between SQL and DataFrame operations, query CSV files from URLs, implement trait-based architecture for extensibility, and create Python bindings with PyO3. This advanced tutorial covers lifetimes, macros, pattern matching, and cross-language interoperability while building a real-world data processing tool."

date: 2025-08-06
series: ["Rust"]
weight: 1
tags: ["rust", "sql-parser", "python-bindings", "data-processing", "ast-conversion"]
author: ["Garry Chen"]
cover:
  image: images/rust-06-00.webp
  hiddenInList: true
  caption: "SQL"

---


Through the examples of HTTPie and Thumbor, you should have gained a clearer understanding of Rust's capabilities and coding style. We've mentioned before that Rust has a very broad range of applications, though these two examples don't fully showcase this.

Some people want to see what code with extensive lifetime annotations looks like in practice; some are curious about Rust's macros; some are interested in Rust's interoperability with other languages; and some want to know what it's like to use Rust for client-side development. So today, we'll cover all these aspects with a hardcore example.

# SQL

In our work, we often deal with various data sources such as databases, Parquet, CSV, and JSON. The process typically involves fetching, filtering, projecting, and sorting data.

People who work with big data might use tools like Spark SQL to query various heterogeneous data sources. However, our day-to-day SQL usage isn't as powerful. While SQL can be used to query databases (with any DBMS supporting it), querying CSV or JSON with SQL requires a lot of additional handling.

Wouldn't it be great to have a simple tool that **allows you to use SQL to query any data source without introducing something as complex as Spark**?

For example, wouldn't it be fantastic if your shell supported commands like this?

![shell example](images/rust-06-01.webp)

Additionally, our client might fetch a subset of data from a server API. If this subset could be further queried directly on the front end using SQL, it would be incredibly flexible and provide instant responses to users.


In software development, there's a famous rule known as [Greenspun's Tenth Rule](https://en.wikipedia.org/wiki/Greenspun%27s_tenth_rule):
> Any sufficiently complicated C or Fortran program contains an ad hoc, informally-specified, bug-ridden, slow implementation of half of Common Lisp.

Let's create our own Programmer's Forty-Second Rule:
> Any sufficiently complex API interface will contain an ad hoc, informally-specified, bug-ridden, slow implementation of half of SQL.

So, how about we design a library that allows SQL queries on any data source and retrieves the results? Of course, as a Minimum Viable Product (MVP), we'll start by supporting SQL queries on CSV files. Moreover, we want this library to be usable from Python3 and Node.js.

Guess how many lines of code this library will take? Today's challenge is quite difficult, so let's set a benchmark of 500 lines of code.

# Design Analysis

First, we need a SQL parser. Writing a parser in Rust isn't difficult. We can use [`serde`](https://github.com/serde-rs/serde), any [parser combinator](https://en.wikipedia.org/wiki/Parser_combinator), or a [PEG parser](https://en.wikipedia.org/wiki/Parsing_expression_grammar) like [`nom`](https://github.com/rust-bakery/nom) or [`pest`](https://github.com/pest-parser/pest). However, SQL parsing is common enough that the Rust community already has a solution: [`sqlparser-rs`](https://github.com/sqlparser-rs/sqlparser-rs).

Next, we need to load CSV or other data sources into a DataFrame.

Anyone familiar with data processing or who has used [`pandas`](https://pandas.pydata.org/pandas-docs/stable/index.html) should be familiar with DataFrames. It's a matrix data structure where each column can contain different types. You can perform filtering, projection, and sorting on a DataFrame.

In Rust, we can use [`polars`](https://github.com/pola-rs/polars) to load data from CSV into a DataFrame and perform subsequent operations.

With these two libraries chosen, the next step is to map the abstract syntax tree ([AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree)) parsed by `sqlparser` to operations on a `polars` DataFrame.

An abstract syntax tree is a tool for describing complex syntax rules, from SQL or a specific DSL to an entire programming language. Its structure can be described using an AST, as shown in the following diagram.

![AST](images/rust-06-02.webp)

How do we map SQL syntax to DataFrame operations? For example, selecting three columns from the data with `SELECT a, b, c` should map to selecting columns `a`, `b`, and `c` from the DataFrame.

`polars` internally has its own AST to aggregate various operations and execute them together. For example, the expression `WHERE a > 10 AND b < 5` in `polars` is represented as `col("a").gt(lit(10)).and(col("b").lt(lit(5)))`. `col` represents a column, `gt/lt` stands for greater than/less than, and `lit` stands for a literal value.

Understanding this, the core problem of "performing SQL queries on CSV and other sources" becomes **converting one AST (SQL AST) into another AST (DataFrame AST)**.

Wait, isn't this what macro programming (specifically, procedural macros in Rust) is for? Upon further analysis of the data structures, we can derive the following correspondence:

![AST mapping](images/rust-06-03.webp)

See, our main task is converting between two data structures. So, after writing today's code, you'll definitely have confidence in macros.

Macro programming isn't as daunting as it seems. Putting aside quote/unquote, its main job is converting one syntax tree into another. This conversion process boils down to converting one data structure into another. In short, **the main flow of macro programming is implementing several `From` and `TryFrom` traits**. Sounds simple, right?

Of course, the conversion process is quite tedious. Without good pattern matching capabilities in the language, macro programming would be an inhumane ordeal.

Fortunately, Rust has excellent pattern matching support. While not as powerful as Erlang/Elixir, it surpasses most programming languages. You'll feel this firsthand when you start coding.

# Creating a SQL Dialect

With our analysis complete, let's proceed step-by-step with the coding.

First, we use `cargo new queryer --lib` to generate a library. Open the generated directory in VSCode, create an `examples` directory at the same level as `src`, and add the following to `Cargo.toml`:

```toml
[[example]]
name = "dialect"

[dependencies]
anyhow = "1" # Error handling; ideally we should use `thiserror` for libraries, but let's keep it simple.
async-trait = "0.1" # Allows async functions in traits.
sqlparser = "0.10" # SQL parser.
polars = { version = "0.15", features = ["json", "lazy"] } # DataFrame library.
reqwest = { version = "0.11", default-features = false, features = ["rustls-tls"] } # Our old friend, the HTTP client.
tokio = { version = "1", features = ["fs"]} # Our old friend, the async library, here we need async file handling.
tracing = "0.1" # Logging.

[dev-dependencies]
tracing-subscriber = "0.2" # Logging.
tokio = { version = "1", features = ["full"]} # In examples, we need more `tokio` features.
```

With dependencies sorted out, let's try an example with `sqlparser` since we're not very familiar with its functionality. This example will look for [`dialect.rs`](https://github.com/GarryChen-site/medium-repo/blob/main/Rust/06_queryer/queryer/examples/dialect.rs) in the `examples` directory.

Create the `examples/dialect.rs` file and write some code to test `sqlparser`:

```rust
use sqlparser::{dialect::GenericDialect, parser::Parser};

fn main() {
    tracing_subscriber::fmt::init();

    let sql = "SELECT a a1, b, 123, myfunc(b), * \
    FROM data_source \
    WHERE a > b AND b < 100 AND c BETWEEN 10 AND 20 \
    ORDER BY a DESC, b \
    LIMIT 50 OFFSET 10";

    let ast = Parser::parse_sql(&GenericDialect::default(), sql);
    println!("{:#?}", ast);
}
```

This code uses a SQL statement to test what structure `Parser::parse_sql` will output. When writing library code, if you're unclear about a third-party library, you can try it out by writing an example like this.

Let's run `cargo run --example dialect` to see the results:

```text
Ok([Query(
    Query {
        with: None,
        body: Select(
            Select {
                distinct: false,
                top: None,
                projection: [ ... ],
                from: [ TableWithJoins { ... } ],
                selection: Some(BinaryOp { ... }),
                ...
            }
        ),
        order_by: [ OrderByExpr { ... } ],
        limit: Some(Value( ... )),
        offset: Some(Offset { ... })
    }
])
```

I've simplified the structure here. What you'll see in the command line will be much more complex.

Reading up to line 9, have you ever wondered how cool it would be **if the `FROM` clause in SQL could accept a URL or a filename**? This way, we could read data from that URL or file. Just like the example at the beginning, `SELECT * FROM ps`, using the `ps` command as a data source, we could easily extract data from its output.

Standard SQL doesn't support this kind of syntax, but `sqlparser` allows you to create your own SQL dialect. So, let's give it a try.

Create a `src/dialect.rs` file and add the following code:

```rust
use sqlparser::dialect::Dialect;

#[derive(Debug, Default)]
pub struct TyrDialect;

// Create our own SQL dialect. TyrDialect supports identifiers that can be simple URLs
impl Dialect for TyrDialect {
    fn is_identifier_start(&self, ch: char) -> bool {
        ('a'..='z').contains(&ch) || ('A'..='Z').contains(&ch) || ch == '_'
    }

    // Identifiers can include ':', '/', '?', '&', '='
    fn is_identifier_part(&self, ch: char) -> bool {
        ('a'..='z').contains(&ch)
            || ('A'..='Z').contains(&ch)
            || ('0'..='9').contains(&ch)
            || [':', '/', '?', '&', '=', '-', '_', '.'].contains(&ch)
    }
}

/// Test helper function
pub fn example_sql() -> String {
    let url = "https://raw.githubusercontent.com/owid/covid-19-data/master/public/data/latest/owid-covid-latest.csv";

    let sql = format!(
        "SELECT location name, total_cases, new_cases, total_deaths, new_deaths \
        FROM {} WHERE new_deaths >= 500 ORDER BY new_cases DESC LIMIT 6 OFFSET 5",
        url
    );

    sql
}

#[cfg(test)]
mod tests {
    use super::*;
    use sqlparser::parser::Parser;

    #[test]
    fn it_works() {
        assert!(Parser::parse_sql(&TyrDialect::default(), &example_sql()).is_ok());
    }
}
```

This code mainly implements the `Dialect` trait from `sqlparser`, allowing us to overload the SQL parser's identifier recognition methods. Next, we need to add the following to `src/lib.rs`:

```rust
mod dialect;
```

Import this file, and finally, I wrote a test. You can run `cargo test` to check it out. The test passed! Now we can properly parse SQL like this:

```sql
SELECT * FROM https://abc.xyz/covid-cases.csv WHERE new_deaths >= 500
```

Cool! You see, with about 10 lines of code (from line 7 to line 19), by adding characters that make URLs valid, we implemented our own SQL dialect that supports URLs.

Why is this so powerful? Because with traits, you can easily achieve [Inversion of Control (IoC)](https://en.wikipedia.org/wiki/Inversion_of_control). In Rust development, this is a very common practice.

# Implementing AST Conversion

We have completed SQL parsing, and now it's time to use Polars for AST conversion. 

Since we are not very familiar with the Polars library, let's first test how to use it. Create `examples/covid.rs` (remember to add it to `Cargo.toml`), and manually implement DataFrame loading and querying:

```rust
use anyhow::Result;
use polars::prelude::*;
use std::io::Cursor;

#[tokio::main]
async fn main() -> Result<()> {
    tracing_subscriber::fmt::init();

    let url = "https://raw.githubusercontent.com/owid/covid-19-data/master/public/data/latest/owid-covid-latest.csv";
    let data = reqwest::get(url).await?.text().await?;

    // Using Polars to directly request
    let df = CsvReader::new(Cursor::new(data))
        .infer_schema(Some(16))
        .finish()?;

    let filtered = df.filter(&df["new_deaths"].gt(500))?;
    println!(
        "{:?}",
        filtered.select((
            "location",
            "total_cases",
            "new_cases",
            "total_deaths",
            "new_deaths"
        ))
    );

    Ok(())
}
```

If we run this example, we get a nicely printed table, reading and querying countries and regions with new deaths greater than 500 from the [`owid-covid-latest.csv`](https://raw.githubusercontent.com/owid/covid-19-data/master/public/data/latest/owid-covid-latest.csv) file on GitHub:

![data](images/rust-06-04.webp)

The ultimate goal is to achieve this effect by parsing an SQL query for similar data queries. How do we do it?

As analyzed at the beginning today, **the main task is to convert the AST parsed by `sqlparser` into the AST defined by Polars**. Let's review the SQL AST output again:

```text
Ok([Query(
    Query {
        with: None,
        body: Select(
            Select {
                distinct: false,
                top: None,
                projection: [ ... ],
                from: [ TableWithJoins { ... } ],
                selection: Some(BinaryOp { ... }),
                ...
            }
        ),
        order_by: [ OrderByExpr { ... } ],
        limit: Some(Value( ... )),
        offset: Some(Offset { ... })
    }
])
```

Here, `Query` is one of the structures of the `Statement` enum. Besides querying, SQL statements include data insertion, data deletion, table creation, etc. Today, we only care about `Query`.

So, we can create a file `src/convert.rs`, **first defining a data structure `Sql` to describe the mapping between the two, and then implementing the `TryFrom` trait for `Sql`**:

```rust
/// Parsed SQL
pub struct Sql<'a> {
    pub(crate) selection: Vec<Expr>,
    pub(crate) condition: Option<Expr>,
    pub(crate) source: &'a str,
    pub(crate) order_by: Vec<(String, bool)>,
    pub(crate) offset: Option<i64>,
    pub(crate) limit: Option<usize>,
}

impl<'a> TryFrom<&'a Statement> for Sql<'a> {
    type Error = anyhow::Error;
    fn try_from(sql: &'a Statement) -> Result<Self, Self::Error> {
        match sql {
            // Currently, we only care about query (select ... from ... where ...)
            Statement::Query(q) => {
              ...
            }
        }
    }
}
```

The framework is set; let's continue with the conversion. Looking at the structure of `Query`, it has a `body`, which is of type `Select`, containing `projection`, `from`, and `selection`. In Rust, we can use an assignment statement along with pattern matching and data destructuring to extract them all:

```rust
let Select {
    from: table_with_joins,
    selection: where_clause,
    projection,

    group_by: _,
    ..
} = match &q.body {
    SetExpr::Select(statement) => statement.as_ref(),
    _ => return Err(anyhow!("We only support Select Query at the moment")),
};
```

In one line, from matching to taking references, to assigning the references' internal fields to variables, all done comfortably! How can you not love a language that can greatly improve productivity?

Let's look at an example of handling `Offset`. We need to convert the `Offset` from `sqlparser` to `i64`, and we can implement a `TryFrom` trait for it. This time, it's data destructuring in a match branch.

```rust
use sqlparser::ast::Offset as SqlOffset;

// Due to Rust's orphan rule, if we want to implement a trait for an existing type,
// we need to wrap it slightly.

pub struct Offset<'a>(pub(crate) &'a SqlOffset);

/// Convert SqlParser's offset expr to i64
impl<'a> From<Offset<'a>> for i64 {
    fn from(offset: Offset<'a>) -> Self {
        match offset.0 {
            SqlOffset {
                value: SqlExpr::Value(SqlValue::Number(v, _b)),
                ..
            } => v.parse().unwrap_or(0),
            _ => 0,
        }
    }
}
```

Yes, data destructuring can also occur in branches. If you remember the `if let` / `while let` discussed in the third lecture, it's the same usage. This comprehensive support for pattern matching makes you increasingly grateful to Rust's authors, especially when developing procedural macros.

From this code, you can also see that the defined data structure `Offset` uses a lifetime annotation `< 'a >` because it uses a reference to `SqlOffset` internally. We will soon talk about lifetime knowledge, and for now, you don't need to understand why this is done.

The entire `src/convert.rs` mainly uses pattern matching to convert between different subtypes. 

In the future, when writing procedural macros in Rust, the main task is this kind of work. However, instead of outputting the transformed AST as code using `quote`, Polars' lazy interface can directly handle the AST in this example.


I've been talking at length about the process of data conversion because it is a crucial part of our programming activities. Think about it—what do we mainly do when we write code? **Most of the logic involves converting data from one interface to another**.

Take the user registration process we are familiar with as an example:

1. After user input is validated on the frontend, it is converted into a `CreateUser` object and then converted into an HTTP POST request.
2. When this request reaches the server, the server reads it and converts it into the server's `CreateUser` object. After validation and normalization, this object is converted into an ORM object (if using ORM). The ORM object is then converted into SQL and sent to the database server.
3. The database server wraps the SQL request into a Write-Ahead Logging ([WAL](https://en.wikipedia.org/wiki/Write-ahead_logging)), which is then updated into the database file.

The entire data conversion process is shown in the diagram below:

![process](images/rust-06-05.webp)

Such a processing flow is often written in a very coupled manner due to its strong business binding, and over time, it becomes unmaintainable spaghetti code. **Good code should make each main process clear and concise, with code appearing exactly where it should be, making it understandable without comments**.

This means we need to encapsulate those unimportant details in a separate place, with the granularity of encapsulation being such that it is written once and rarely needs to change. Even if it changes, the impact should be very localized.

Such code is easy to read, easy to test, simple to maintain, and a pleasure to work with. Rust's standard library's `From`/`TryFrom` trait is designed for this purpose and is well worth using.

# Fetching Data from the Source

Having completed the AST conversion, the next step is to fetch data from the source.

By processing and populating the `Sql` structure, we can obtain the data source in the SQL `FROM` clause, which we stipulate must be a string starting with `http(s)://` or `file://`. Because if it starts with `http`, we can fetch content via URL, and if it starts with `file`, we can open a local file to fetch content.

So, once we have this string describing the data source, it's easy to write the following code:

```rust
/// Fetch data from a file source or an HTTP source
async fn retrieve_data(source: impl AsRef<str>) -> Result<String> {
    let name = source.as_ref();
    match &name[..4] {
        // Including http / https
        "http" => Ok(reqwest::get(name).await?.text().await?),
        // Handle file://<filename>
        "file" => Ok(fs::read_to_string(&name[7..]).await?),
        _ => Err(anyhow!("We only support http/https/file at the moment")),
    }
}
```

The code looks simple but will not be easy to maintain in the future. If your HTTP request results need some subsequent processing, this function will quickly become complex. So what should we do?

If you recall the code we wrote in the first two lectures, you should have an answer in mind: **we can use traits to abstract the fetch logic, define the interface, and then change the implementation of `retrieve_data`**.

Below is the complete code for `src/fetcher.rs`:

```rust
use anyhow::{anyhow, Result};
use async_trait::async_trait;
use tokio::fs;

// Rust's async trait is not yet stable, so we can use the async_trait macro
#[async_trait]
pub trait Fetch {
    type Error;
    async fn fetch(&self) -> Result<String, Self::Error>;
}

/// Fetch data from a file source or an HTTP source and form a data frame
pub async fn retrieve_data(source: impl AsRef<str>) -> Result<String> {
    let name = source.as_ref();
    match &name[..4] {
        // Including http / https
        "http" => UrlFetcher(name).fetch().await,
        // Handle file://<filename>
        "file" => FileFetcher(name).fetch().await,
        _ => return Err(anyhow!("We only support http/https/file at the moment")),
    }
}

struct UrlFetcher<'a>(pub(crate) &'a str);
struct FileFetcher<'a>(pub(crate) &'a str);

#[async_trait]
impl<'a> Fetch for UrlFetcher<'a> {
    type Error = anyhow::Error;

    async fn fetch(&self) -> Result<String, Self::Error> {
        Ok(reqwest::get(self.0).await?.text().await?)
    }
}

#[async_trait]
impl<'a> Fetch for FileFetcher<'a> {
    type Error = anyhow::Error;

    async fn fetch(&self) -> Result<String, Self::Error> {
        Ok(fs::read_to_string(&self.0[7..]).await?)
    }
}
```

It seems that this approach yields more code, but it separates `retrieve_data` from the specific handling of each type, following the open/closed principle to construct low-coupling, high-cohesion code. This way, when we modify `UrlFetcher` or `FileFetcher`, or add a new `Fetcher`, the impact on `retrieve_data` is minimal.

Now we have completed the SQL parsing, implemented the AST conversion from SQL to DataFrame, and fetched data from the source. The challenge is more than halfway done, leaving only the main process logic.

# Main Process

When we create a library, we typically don't expose the internal data structures directly; instead, we wrap them in our own data structures.

However, this approach has a drawback: **if we want to expose methods of the original data structure, we need to implement each interface again**, even if it's just a simple proxy. This is a common pain point in many languages.

In the `queryer` library, we encounter the same issue: the results of SQL queries are stored in a `polars` DataFrame, but we don't want to expose this DataFrame directly. If we did, it would be difficult to add additional metadata in the future.

So, I defined a `DataSet` to wrap around the DataFrame. However, I also wanted to expose the DataFrame’s methods via the `DataSet`. With so many functions, it's impractical to proxy them one by one.

Fortunately, Rust provides the `Deref` and `DerefMut` traits for this purpose, which allow a type to dereference to another type. We'll cover these traits in detail later when discussing common Rust traits, but for now, let's see how `DataSet` handles it:

```rust
#[derive(Debug)]
pub struct DataSet(DataFrame);

/// Allow DataSet to behave like a DataFrame
impl Deref for DataSet {
    type Target = DataFrame;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

/// Allow DataSet to behave like a DataFrame
impl DerefMut for DataSet {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}

// DataSet's own methods
impl DataSet {
    /// Convert DataSet to CSV
    pub fn to_csv(&self) -> Result<String> {
        let mut buf = Vec::new();
        let writer = CsvWriter::new(&mut buf);
        writer.finish(self)?;
        Ok(String::from_utf8(buf)?)
    }
}
```

As you can see, the `DataSet` dereferences to a `DataFrame`, so when users use `DataSet`, it behaves just like a `DataFrame`. Additionally, we implemented a `to_csv` method for `DataSet`, allowing us to generate CSV from query results.

With `DataSet` defined, implementing the core function `query` becomes straightforward: we first parse the SQL structure, then read a `DataSet` from the source, apply filter/order_by/offset/limit/select operations, and finally return the `DataSet`.

Both the `DataSet` definition and the `query` function are in `src/lib.rs`. Here is the complete code:

```rust
use anyhow::{anyhow, Result};
use polars::prelude::*;
use sqlparser::parser::Parser;
use std::convert::TryInto;
use std::ops::{Deref, DerefMut};
use tracing::info;

mod convert;
mod dialect;
mod loader;
mod fetcher;
use convert::Sql;
use loader::detect_content;
use fetcher::retrieve_data;

pub use dialect::example_sql;
pub use dialect::TyrDialect;

#[derive(Debug)]
pub struct DataSet(DataFrame);

/// Allow DataSet to behave like a DataFrame
impl Deref for DataSet {
    type Target = DataFrame;

    fn deref(&self) -> &Self::Target {
        &self.0
    }
}

/// Allow DataSet to behave like a DataFrame
impl DerefMut for DataSet {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.0
    }
}

impl DataSet {
    /// Convert DataSet to CSV
    pub fn to_csv(&self) -> Result<String> {
        let mut buf = Vec::new();
        let writer = CsvWriter::new(&mut buf);
        writer.finish(self)?;
        Ok(String::from_utf8(buf)?)
    }
}

/// Query data from the source, filter by conditions, and select the required columns
pub async fn query<T: AsRef<str>>(sql: T) -> Result<DataSet> {
    let ast = Parser::parse_sql(&TyrDialect::default(), sql.as_ref())?;

    if ast.len() != 1 {
        return Err(anyhow!("Only support single SQL at the moment"));
    }

    let sql = &ast[0];

    // The details of converting the entire SQL AST to our defined Sql structure are buried in try_into().
    // We only need to focus on using the data structure, and conversion details can be addressed later if needed.
    // This separation of concerns helps us manage software complexity.
    let Sql {
        source,
        condition,
        selection,
        offset,
        limit,
        order_by,
    } = sql.try_into()?;

    info!("retrieving data from source: {}", source);

    // Read a DataSet from the source
    // How detect_content works isn't important; what's important is that it returns a DataSet based on the content
    let ds = detect_content(retrieve_data(source).await?).load()?;

    let mut filtered = match condition {
        Some(expr) => ds.0.lazy().filter(expr),
        None => ds.0.lazy(),
    };

    filtered = order_by
        .into_iter()
        .fold(filtered, |acc, (col, desc)| acc.sort(&col, desc));

    if offset.is_some() || limit.is_some() {
        filtered = filtered.slice(offset.unwrap_or(0), limit.unwrap_or(usize::MAX));
    }

    Ok(DataSet(filtered.select(selection).collect()?))
}
```

In the main process of the `query` function, the entire SQL AST is converted into our defined `Sql` structure, with the details hidden in `try_into()`. We only need to focus on using the `Sql` data structure, and conversion details can be addressed later. 

This is the essence of [Separation of Concerns](https://en.wikipedia.org/wiki/Separation_of_concerns), a key principle for managing software complexity. The traits in Rust's standard library, refined through extensive use, help us write better, less complex code.

The main process involves a `detect_content` function, which identifies the content and chooses the appropriate loader to load the text as a `DataSet`. Currently, it only supports CSV but can be extended to support JSON and other formats in the future. This function is defined in `src/loader.rs`. Let's create this file and add the following code:

```rust
use crate::DataSet;
use anyhow::Result;
use polars::prelude::*;
use std::io::Cursor;

pub trait Load {
    type Error;
    fn load(self) -> Result<DataSet, Self::Error>;
}

#[derive(Debug)]
#[non_exhaustive]
pub enum Loader {
    Csv(CsvLoader),
}

#[derive(Default, Debug)]
pub struct CsvLoader(pub(crate) String);

impl Loader {
    pub fn load(self) -> Result<DataSet> {
        match self {
            Loader::Csv(csv) => csv.load(),
        }
    }
}

pub fn detect_content(data: String) -> Loader {
    // TODO: Content detection
    Loader::Csv(CsvLoader(data))
}

impl Load for CsvLoader {
    type Error = anyhow::Error;

    fn load(self) -> Result<DataSet, Self::Error> {
        let df = CsvReader::new(Cursor::new(self.0))
            .infer_schema(Some(16))
            .finish()?;
        Ok(DataSet(df))
    }
}
```

By using traits, we ensure that while we currently only support `CsvLoader`, we have left the interface open for adding more loaders in the future.

Now the [library](https://github.com/GarryChen-site/medium-repo/tree/main/Rust/06_queryer/queryer) is fully implemented. Try compiling it.

If the code compiles successfully, you can modify the previous `examples/covid.rs` to test using SQL for querying:

```rust
use anyhow::Result;
use queryer::query;

#[tokio::main]
async fn main() -> Result<()> {
    tracing_subscriber::fmt::init();

    let url = "https://raw.githubusercontent.com/owid/covid-19-data/master/public/data/latest/owid-covid-latest.csv";

    // Use SQL to fetch data from the URL
    let sql = format!(
        "SELECT location name, total_cases, new_cases, total_deaths, new_deaths \
        FROM {} WHERE new_deaths >= 500 ORDER BY new_cases DESC",
        url
    );
    let df1 = query(sql).await?;
    println!("{:?}", df1);

    Ok(())
}
```

Bingo! Everything works as expected. We successfully requested a CSV from the web using SQL, queried, and sorted it, and the results are correct!

![result](images/rust-06-06.webp)

Using `tokei` to check the lines of code, you can see it’s 375 lines, well below the 500-line target!

![lines](images/rust-06-07.webp)

With such a small amount of code, we’ve done a lot of decoupling work: the architecture is split into four parts—Sql Parser, Fetcher, Loader, and Query.

![architecture](images/rust-06-08.webp)

Potentially changing Fetchers and Loaders can be easily extended in the future. For instance, the `select * from ps` example mentioned initially can be handled with a `StdoutFetcher` and `TsvLoader`.


# Support for Other Languages

Now that we have completed the core code, don't you feel a sense of accomplishment? The queryer tool we implemented can be used as a library in Rust, available for other Rust programs, which is wonderful.

But our story doesn't end here. It would be a waste if only Rust developers could enjoy such a powerful feature. After all, sharing joy is better than enjoying it alone. So, let's try to **integrate it into other languages, like the commonly used Python**.

There is a lot of high-performance code in Node.js and Python written in C/C++, but cross-language calls often involve complex interface conversion code, which makes using C/C++ painful. Let's see if using Rust can avoid these tedious processes. We have high expectations for using Rust to provide high-performance code for other languages. If this process is also complex, then how can it be utilized?

For the queryer library, the main interface we want to expose is: `query`, where the user inputs a SQL string and an output type string, and returns a string processed according to the SQL query and matching the output type. For Python, this is the interface:

```python
def query(sql, output='csv')
```

Alright, let's give it a try.

First, create a new directory called `queryer` as a workspace, move the existing queryer into it as a subdirectory. Then, create a `Cargo.toml` with the following content:

```toml
[workspace]

members = [
  "queryer",
  "queryer-py"
]
```

# Python

In the root directory of the workspace, create a new crate with `cargo new queryer-py --lib`. In the `queryer-py` directory, edit `Cargo.toml`:

```toml
[package]
name = "queryer_py" # Python module needs to use underscores
version = "0.1.0"
edition = "2018"

[lib]
crate-type = ["cdylib"] # Use cdylib type

[dependencies]
queryer = { path = "../queryer" } # Import queryer
tokio = { version = "1", features = ["full"] }

[dependencies.pyo3] # Import pyo3
version = "0.14"
features = ["extension-module"]

[build-dependencies]
pyo3-build-config = "0.14"
```

The library for Rust and Python interaction is pyo3. You can check its documentation later if you're interested. In `src/lib.rs`, add the following code:

```rust
use pyo3::{exceptions, prelude::*};

#[pyfunction]
pub fn example_sql() -> PyResult<String> {
    Ok(queryer::example_sql())
}

#[pyfunction]
pub fn query(sql: &str, output: Option<&str>) -> PyResult<String> {
    let rt = tokio::runtime::Runtime::new().unwrap();
    let data = rt.block_on(async { queryer::query(sql).await.unwrap() });
    match output {
        Some("csv") | None => Ok(data.to_csv().unwrap()),
        Some(v) => Err(exceptions::PyTypeError::new_err(format!(
            "Output type {} not supported",
            v
        ))),
    }
}

#[pymodule]
fn queryer_py(_py: Python, m: &PyModule) -> PyResult<()> {
    m.add_function(wrap_pyfunction!(query, m)?)?;
    m.add_function(wrap_pyfunction!(example_sql, m)?)?;
    Ok(())
}
```

Even if I don't explain the code, you can basically understand what it's doing. We provide two interfaces for the Python module: `example_sql` and `query`.

Next, in the `queryer-py` directory, create a virtual environment and then use `maturin develop` to build the Python module:

```sh
python3 -m venv .env
source .env/bin/activate
pip install maturin ipython
maturin develop
```

After building, you can test it with `ipython`:

```python
In [1]: import queryer_py

In [2]: sql = queryer_py.example_sql()

In [3]: print(queryer_py.query(sql, 'csv'))
name,total_cases,new_cases,total_deaths,new_deaths
India,32649947.0,46759.0,437370.0,509.0
Iran,4869414.0,36279.0,105287.0,571.0
Africa,7695475.0,33957.0,193394.0,764.0
South America,36768062.0,33853.0,1126593.0,1019.0
Brazil,20703906.0,27345.0,578326.0,761.0
Mexico,3311317.0,19556.0,257150.0,863.0

In [4]: print(queryer_py.query(sql, 'json'))
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-4-7082f1ffe46a> in <module>
----> 1 print(queryer_py.query(sql, 'json'))

TypeError: Output type json not supported
```

Cool! With just 20 lines of code, we made our module callable by Python, and error handling works as expected. As you can see, by writing a bit of auxiliary code on top of the Rust library, we can integrate it with different languages. I think this is a very promising direction for Rust.

For many companies, it is very costly to fully migrate their existing codebase to Rust, but by integrating Rust with various languages, they can migrate parts of the code that require high performance to Rust, enjoy the benefits, and gradually promote it. This way, Rust can be applied more effectively.

# Summary

Looking back on our Rust coding journey this week, we first created an HTTPie, a simple flow with a bronze-level difficulty that you can write after learning ownership and understanding basic traits.

Next, we created Thumbor, introducing async, generics, and more traits, with a silver-level difficulty. After understanding the type system and getting a bit familiar with async, you should be able to handle it.

Today's Queryer used a lot of traits to ensure the code structure adheres to the open-closed principle and separation of concerns. It used several lifetime annotations to reduce unnecessary memory copies and employed complex pattern matching to retrieve data. This is a gold-level difficulty, and after studying the advanced sections of this course, you should be able to understand this code.

Many people find Rust code difficult to write, especially when generic data structures and lifetimes are involved. However, in the previous examples, lifetime annotations appeared only once. **So, most of the time, your code doesn't need complex lifetime annotations**.

As long as you understand ownership and lifetimes well, if you find yourself endlessly battling with lifetime annotations and struggling with the compiler, you might need to stop and think:

If the compiler dislikes my approach so much, could there be a problem with my design? Should I use a better data structure? Should I redesign it? Is my code overly coupled?

Just like there are four ways to write the Chinese character "hui" in fennel beans, different people will have different ways to write the same requirement using the same language. However, **excellent design must produce simple and readable code, not the opposite**.

---