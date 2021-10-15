One of the best ways that SQLFluff users can improve SQLFluff for themselves and others is in contributing dialect changes.

Users will likely know their syntax much better than the regular maintainers and will have access to an instance of that SQL dialect to confirm changes are valid SQL in that dialect.

If you can fix your own issues then that's often the quickest way of unblocking any issues preventing you from using SQLFluff! The maintainers are all volunteers doing this in our spare time and (like you all I'm sure!), we only have so much time to work on this.

## How SQLFluff reads (or parses) SQL

SQLFluff has a lexer and parser which is built in a very modular fashion that is easy to read, understand, and expand on without any core programming skills or deep knowledge of Python or how SQLFluff operates. For more information see the [Architecture documentation](https://docs.sqlfluff.com/en/stable/architecture.html), but will cover that briefly here to give you enough to start contributing.

We also have a robust Continuous Integration pipeline in GitHub where you can gain confidence your changes are correct and will not break other SQLFluff users, even before a regular maintainer reviews the code.

SQLFluff defines the syntax it will used in dialect files (more on this later). If you look at the [`dialect_ansi.py`](https://github.com/sqlfluff/sqlfluff/blob/main/src/sqlfluff/dialects/dialect_ansi.py) file you will see it has syntax like this:

```py
@ansi_dialect.segment()
class SelectClauseSegment(BaseSegment):
    """A group of elements in a select target statement."""

    type = "select_clause"
    match_grammar = StartsWith(
        Sequence("SELECT", Ref("WildcardExpressionSegment", optional=True)),
        terminator=OneOf(
            "FROM",
            "WHERE",
            "ORDER",
            "LIMIT",
            "OVERLAPS",
            Ref("SetOperatorSegment"),
        ),
        enforce_whitespace_preceding_terminator=True,
    )

    parse_grammar = Ref("SelectClauseSegmentGrammar")
```

This says the `SelectClauseSegment` starts with `SELECT` or `SELECT *` and ends when it encouters a `FROM`, `WHERE`, `ORDER`...etc. line.

The `match_grammar` is what is used primarily to try to match and parse the statement. It can be relatively simple (as in this case), to quickly match just the start and terminating clauses. If that is the case, then a `parse_grammar` is needed to actually delve into the statement itself with all the clauses and parts it is made up of. The `match_grammar` is used to quickly identify the start and end of this block, as parsing can be quite intensive and complicated as the parser tries various combinations of clauses (particularly optional ones like the `WildcardExpressionSegment` above, or when there is a choice of statements that could be used).

For some statements a quick match is not needed, and so we can delve straight into the full grammar definition. In that case the `match_grammar` will be sufficient and we don't need the optional `parse_grammar`.

Here's another statement, which only uses the `match_grammar` and doesn't have or need an optional `parse_grammar`:

```py
@ansi_dialect.segment()
class JoinOnConditionSegment(BaseSegment):
    """The `ON` condition within a `JOIN` clause."""

    type = "join_on_condition"
    match_grammar = Sequence(
        "ON",
        Indent,
        OptionallyBracketed(Ref("ExpressionSegment")),
        Dedent,
    )
```

You will have noticed that a segment can refer to another segment, and that is a good way of splitting up a complex SQL expression into it's component parts to manage and handle them separately.

### Segment grammar options

There are a number of options when creating SQL grammar including:

Grammar|Used For|Example
---|---|---
`"KEYWORD"`|Having a raw SQL keyword|`"SELECT"`,
`Sequence`|Having a set of Keywords or Segments|`Sequence("SELECT", Ref("SelectClauseElementSegment"), "FROM"...)`
`AnyNumberOf`|Choose from a set of options which may be repeated|`"SELECT", AnyNumberOf(Ref("WildcardExpressionSegment"), Ref("ColumnReferenceSegment")...)...`
`OneOf`|A more restrictive from a set of `AnyNumberOf` limited to just one option|`OneOf("INNER","OUTER","FULL"), "JOIN"`
`Delimited`|Used for lists (e.g. comma-delimited - which is the default)|`"SELECT", Delimited("SelectClauseElementSegment"), "FROM"...`
`Bracketed`|Used for bracketed options - like function parameters|`Ref("FunctionNameSegment"), Bracketed(Ref("FunctionContentsGrammar")`

Some of the keywords have extra params you can give them, the most commonly used will be `optional=True`. This allows you to further define the make up of a SQL statement. Here's the `DeleteStatementSegment` definition:

```py
    parse_grammar = Sequence(
        "DELETE",
        Ref("FromClauseSegment"),
        Ref("WhereClauseSegment", optional=True),
    )
```
You can see the `WHERE` clause it optional (many's a head has been shaken because of deletes without `WHERE` clauses I'm sure, but that's what SQL syntax allows!).

Using these Grammar options, it's possible to build up complex structures to define SQL syntax.


### Dialects

A lot of SQL is the same no matter which particular type of SQL you are using. The basic `SELECT.. FROM... WHERE` statement is common to them all. However lots of different SQL dialects (Postgres, Snowflake, Oracle...etc.) have sprung up as different companies have implemented SQL, or expanded it, for their own needs.

For this reason, SQLFluff allows creating _dialects_, which can have different grammars from each other.

SQLFluff has all the dialects in the [`src/sqlfluff/dialects`](https://github.com/sqlfluff/sqlfluff/tree/main/src/sqlfluff/dialects) folder. The main dialect file (that every other dialect ultimately inherits from) is the [`dialect_ansi.py`](https://github.com/sqlfluff/sqlfluff/blob/main/src/sqlfluff/dialects/dialect_ansi.py) file.

In SQLFluff, a dialect is basically a copy of the original ANSI dialect, which then adds or overrides parsing segments. If a dialect has the exact same `SELECT`, `FROM` and `WHERE` clauses as ANSI but a different `ORDER BY` syntax, then only the `ORDER BY` clause needs to overridden so the dialect file will be very small. For some of the other dialects where there's lots of differences (T-SQL!) you may be overriding a lot more.

### Lexing

I kind of skipped this part, but before a piece of SQL can be _parsed_, it is _lexed_ - that is split up into symbols, and logical groupings.

An inline comment, for example, is defined as this:

```py
        RegexLexer(
            "inline_comment",
            r"(--|#)[^\n]*",
            CommentSegment,
            segment_kwargs={"trim_start": ("--", "#")},
        ),
```

That is a anything after `--` or `#` to the newline. This allows us to deal with that whole comment as one lexed block and so the don't need to define how to parse it (we even give that a parsing segment name here - `CommentSegment`).

For simple grammar addition, you won't need to to touch the lexing definitions as they usually cover most common one already. But for slightly more complicated ones, you may have to add to this. So if you see lexing errors then you may have to add something here.

Lexing happens in order. So it starts reading the SQL from the start, until it has the longest lexing match, then it chomps that up, files it away as a symbol to deal with later in the parsing, and starts again with the remaining text. So if you have `SELECT * FROM table WHERE col1 = 12345` it will not break that up into `S`, `E`, `L`...etc., but instead into `SELECT`, `*`, `FROM`, `table`...etc.

An example of where we had to override lexing, is in BigQuery we have parameterised variables which are of the form `@variable_name`. The ANSI lexer doesn't recognise the `@` sign, so you could add a lexer for that. But a better solution, since you don't need to know two parts (`@` and `variable_name`) is to just tell the lexer to go ahead and parse the whole thing into one big symbol, that we will then use later in the parser:

```py
bigquery_dialect.insert_lexer_matchers(
    [
        RegexLexer("atsign_literal", r"@[a-zA-Z_][\w]*", CodeSegment),
    ],
    before="equals",
)
```

Note the `before="equals"` which means we tell the lexer the order of preference to try to match this symbol. For example if we'd defined an `at_sign` lexing rule for other, standalone `@` usage, then we'd want this to be considered first, and only fall back to that if we couldn't match this.


### Keywords

Most dialect have a keywords file, listing all the keywords. Some dialects just inherit the ANSI keywords and then add or remove keywords from that. Not quite as accurate as managing the actual keywords, but a lot quicker and easier to manage usually!

Keywords are separated into RESERVED and UNRESERVED lists. RESERVED keywords have extra restrictions meaning they cannot be used as identifiers. If using a keyword in grammar (e.g. `"SELECT"`), then it needs to be in one of the Keywords lists so you may have to add it. And if editing the main ANSI dialect, and adding the the ANSI keyword list, then take care to consider if it needs added to the other dialects if they will inherit this syntax - usually yes unless explicitly overridden in those dialects.


## Example of contributing a syntax fix

So that's a bit of theory but let's got through some actual examples of how to add to the SQLFluff code to address any issues you are seeing. In this I'm not going to explain about how to set up your Python development environment (see the [CONTRIBUTING.md](https://github.com/sqlfluff/sqlfluff/blob/main/CONTRIBUTING.md) file for that), nor how to manage Git (there's a number of resources on this and we use the standard Fork, and open an PR workflow).

So assuming you know how to set up Python environment, and commit via Git, how do you contribute a simple fix to a dialect for syntax you want SQLFluff to support?

### Example 1

If we look at issue [#1520](https://github.com/sqlfluff/sqlfluff/issues/1520) it was raised to say we couldn't parse this:

```sql
CREATE OR REPLACE FUNCTION public.postgres_setof_test()
 RETURNS SETOF text
```

and instead returned this message:
```
Found unparsable section: 'CREATE OR REPLACE FUNCTION crw_public.po...'
```

This was in the `postgres` dialect, so I had a look at (`dialect_postgres.py`)[https://github.com/sqlfluff/sqlfluff/blob/main/src/sqlfluff/dialects/dialect_postgres.py] and found the code in `CreateFunctionStatementSegment` which had the following:

```py
    parse_grammar = Sequence(
        "CREATE",
        Sequence("OR", "REPLACE", optional=True),
        Ref("TemporaryGrammar", optional=True),
        "FUNCTION",
        Sequence("IF", "NOT", "EXISTS", optional=True),
        Ref("FunctionNameSegment"),
        Ref("FunctionParameterListGrammar"),
        Sequence(  # Optional function return type
            "RETURNS",
            OneOf(
                Sequence(
                    "TABLE",
                    Bracketed(
                        Delimited(
                            OneOf(
                                Ref("DatatypeSegment"),
                                Sequence(
                                    Ref("ParameterNameSegment"), Ref("DatatypeSegment")
                                ),
                            ),
                            delimiter=Ref("CommaSegment"),
                        )
                    ),
                    optional=True,
                ),
                Ref("DatatypeSegment"),
            ),
            optional=True,
        ),
        Ref("FunctionDefinitionGrammar"),
    )
```

So it allowed returning a table, or a datatype.

Fixing the issuie was as simple as adding the `SETOF` structure as another return option:

```py
    parse_grammar = Sequence(
        "CREATE",
        Sequence("OR", "REPLACE", optional=True),
        Ref("TemporaryGrammar", optional=True),
        "FUNCTION",
        Sequence("IF", "NOT", "EXISTS", optional=True),
        Ref("FunctionNameSegment"),
        Ref("FunctionParameterListGrammar"),
        Sequence(  # Optional function return type
            "RETURNS",
            OneOf(
                Sequence(
                    "TABLE",
                    Bracketed(
                        Delimited(
                            OneOf(
                                Ref("DatatypeSegment"),
                                Sequence(
                                    Ref("ParameterNameSegment"), Ref("DatatypeSegment")
                                ),
                            ),
                            delimiter=Ref("CommaSegment"),
                        )
                    ),
                    optional=True,
                ),
                Sequence(
                    "SETOF",
                    Ref("DatatypeSegment"),
                ),
                Ref("DatatypeSegment"),
            ),
            optional=True,
        ),
        Ref("FunctionDefinitionGrammar"),
    )
```

With that code the above item could parse.

I added a test case (covered below) and submitted [pull request #1522](https://github.com/sqlfluff/sqlfluff/pull/1522/files) to fix this.

### Example 2

If we look at issue [#1537](https://github.com/sqlfluff/sqlfluff/issues/1537) it was raised to say we couldn't parse this:

```sql
select 1 from group
```

And threw this error:

```
==== parsing violations ====
L:   1 | P:  10 |  PRS | Line 1, Position 10: Found unparsable section: 'from'
L:   1 | P:  14 |  PRS | Line 1, Position 14: Found unparsable section: ' group'
```

The reporter had also helpfully included the parse tree (produced by `sqlfluff parse`):

```
[L:  1, P:  1]      |file:
[L:  1, P:  1]      |    statement:
[L:  1, P:  1]      |        select_statement:
[L:  1, P:  1]      |            select_clause:
[L:  1, P:  1]      |                keyword:                                      'select'
[L:  1, P:  7]      |                [META] indent:
[L:  1, P:  7]      |                whitespace:                                   ' '
[L:  1, P:  8]      |                select_clause_element:
[L:  1, P:  8]      |                    literal:                                  '1'
[L:  1, P:  9]      |            whitespace:                                       ' '
[L:  1, P: 10]      |            [META] dedent:
[L:  1, P: 10]      |            from_clause:
[L:  1, P: 10]      |                unparsable:                                   !! Expected: 'FromClauseSegment'
[L:  1, P: 10]      |                    keyword:                                  'from'
[L:  1, P: 14]      |            unparsable:                                       !! Expected: 'Nothing...'
[L:  1, P: 14]      |                whitespace:                                   ' '
[L:  1, P: 15]      |                raw:                                          'group'
[L:  1, P: 20]      |    newline:                                                  '\n'
```

So the problem was it couldn't parse the `FromClauseSegment`. Looking at that definition showed this:

```py
    FromClauseTerminatorGrammar=OneOf(
        "WHERE",
        "LIMIT",
        "GROUP",
        "ORDER",
        "HAVING",
        "QUALIFY",
        "WINDOW",
        Ref("SetOperatorSegment"),
        Ref("WithNoSchemaBindingClauseSegment"),
    ),
```

So the parser was terminating as soon as it saw the `GROUP` and saying "hey we must have reached the end of the `FROM` clause".

This was a little restrictive so changing that to this solved the problem:

```py
    FromClauseTerminatorGrammar=OneOf(
        "WHERE",
        "LIMIT",
        Sequence("GROUP", "BY"),
        Sequence("ORDER", "BY"),
        "HAVING",
        "QUALIFY",
        "WINDOW",
        Ref("SetOperatorSegment"),
        Ref("WithNoSchemaBindingClauseSegment"),
    ),
```

You can see we simply replaced the `"GROUP"` by a `Sequence("GROUP", "BY")` so it would _only_ match if both words were given. Rechecking the example with this changed code, showed it now parsed. We did the same for `"ORDER"`, and also changed a few other places in the code with similar clauses and added a test case (covered below) and submitted [pull request #1546](https://github.com/sqlfluff/sqlfluff/pull/1546/files) to fix this.

### Testing your changes

So you've made your fix, you've tested it fixed the original problem so just submit that change, and all is good now?

Well, no. You want to do two further things:
- Test your change hasn't broken anything else. To do that you run the test suite.
- Add a test case, so others can check this in future.

To test your changes you'll need to have your environment set up (again see the [CONTRIBUTING.md](https://github.com/sqlfluff/sqlfluff/blob/main/CONTRIBUTING.md) file for how to do that).

#### Adding test cases for your changes

Adding a test case is simple. Just add a SQL file to [`test/fixtures/dialects/`](https://github.com/sqlfluff/sqlfluff/tree/main/test/fixtures/dialects) in the appropriate dialect directory. You can either edit an existing SQL file test case (e.g. if adding something similar to what's in there) or create a new one.

I advise adding the original SQL raised in the issue, and if you have examples from the official syntax, then they are always good test case to add. For example, the Snowflake documentation [has an example section at the bottom of every syntax definition](https://docs.snowflake.com/en/sql-reference/sql/select.html#examples) so just copy all them into your example file too.

#### YML test fixture files

As well as the SQL files, we have YAML equivalents of the statements. This is the parsed version of the SQL, and having these in our source code, allows us to easily see if they change. For most cases (except adding new test cases obviously!) you would not expect unrelated YML files to change so this is a good check.

To regenerate all the YAML files when you add or edit any SQL files run the following command:

```
python test/generate_parse_fixture_yml.py
```

It takes a few mins to run, and should regenerate all the YAML files. You can then do a `git status` to see any differences.

### Running the test suite

There's a few ways of running the test suite.

You could just run the `tox` command, but this will run all the test suites, for various python versions, and with and without dbt, and take a long time. Best to leave that to our CI infrastructure. You just wanna run what you need to have reasonable confidence before submitting.

The run just the dialect tests you can do this:

```bash
pytest test/dialects/dialects_test.py
```

Or, if editing a rule, you can just run the tests for that one rule, for example to test L048 tests:

```bash
pytest -k L048 -v test
```

And finally to run the full test suite (but only on one Python version, and without dbt) use this:

```
tox -e generate-fixture-yml,cov-init,py38,cov-report,linting
```

I like to kick that last one off just before opening a PR but does take a good 10mins to run.

Regardless of what testing you do, GitHub will run the full regression suite when the PR is opened or updated.

#### Black code linting

We use `flake8` to lint our python code (being a linter ourselves we should have high quality code!). Our CI, or the `tox` commands above will run this and flag any errors.

In most cases running `black` on the python file(s) will correct any errors (e.g. line formatting) but for some you'll need to run `flake8` to see the issues and manually correct them.





