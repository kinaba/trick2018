### Remarks

Run it with some JSON input on STDIN or as file arguments.
`-rpp` is needed except on MRI 2.5.

    ruby -rpp entry.rb example.json

I confirmed the following implementations/platforms:

* ruby 2.5.0p0 (2017-12-25 revision 61468) [x86_64-linux]
* truffleruby 0.33, like ruby 2.3.6 <native build with Graal> [x86_64-linux]
* jruby 9.1.16.0 (2.3.3) 2018-02-21 8f3f95a OpenJDK 64-Bit Server VM 25.151-b12 on 1.8.0_151-b12 +jit [linux-x86_64]
* rubinius 3.100 (2.3.1 250dab17 2018-03-02 4.0.1) [x86_64-linux-gnu]

### Description

The program parses JSON input to Ruby using almost exclusively regular expressions.
The program supports JSON input conforming to the grammar at https://www.json.org/.
Notably, it can parse the main test file from the `json` standard library
(`test/json/fixtures/pass1.json`), provided as `example.json` for convenience.


### Internals

#### Internals and Algorithm of the Program

The input is first preprocessed in strings and non-strings tokens to remove
whitespaces in non-strings. The author tried to use a single `gsub` without
splitting into tokens but this took too long with a backtracking Regexp engine.

The grammar is taken directly from https://www.json.org/ and translated to a Regexp.
The grammar is arguably simpler in Regexp form since it is shorter to express
repetitions with Regexps instead of EBNF.
Each grammar rule is a named capture group, suffixed with `{0}` so the initial
definitions do not capture any text. The different rules can refer to each other
arbitrarily thanks to the `\g<name>` subexpression calls feature of Regexps.
From this grammar Regexp, we build a set of initial Regexps for each rule with
`#{grammar}\G\g<#{rule}>`.

While the initial `\g<val>` Regexp expresses the full JSON grammar, which enables
to check easily that the input is valid and rely on it with only one Regexp match,
this program illustrates that it is difficult to get values out of the initial Regexp match.
If only we could have a callback on each match of a capture group!
Indeed, with nested definitions, only one of the many captures (nested in the input)
is reported to the user. So a Regexp never provides more groups to the user than
the number of literal capture groups in the Regexp. `String#scan` can be used to
handle repeated sequences but not recursive groups.

To circumvent this problem, we use a *meta-Regexp*, which parses the contents of
the initial grammar Regexp, to find the next rules to match after each match
against the initial Regexp. If we find that all rules to match are the same, we
consider they are repeated and collect all the results in an Array, taking in account
separators around the rules. If there are different rules in the definition
and there is no repetition, we return a tuple with each value matched.
The only difference between matching an array or an object is expressed in a
single line:

```ruby
rule == "obj" ? values.to_h : values
```

The program should be generalizable to other input languages, although the
*meta-Regexp* is tuned for the JSON rules for simplicity.

#### Formatting

The text is originally generated by FIGlet with the `doh` font.
The result is then tweaked to fit the numbers of characters of the code.
Then care must be taken to avoid `\` not being contiguous with what comes after,
and to be able to distinguish real spaces from formatting spaces.

### Limitation

The program is rather lazy for parsing terminal values such as strings and
numbers and uses `eval` since the syntax is essentially compatible with Ruby.
