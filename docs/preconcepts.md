# Preconcepts
## Regular Expressions
A regular expression \(r\) is a sequence of characters that defines a search pattern. We say that, given an alphabet \(\Sigma\) a regular expression _describes_ a language \(\mathcal{L} \subseteq \Sigma\). Regular expressions are used primarily for string pattern matching and manipulation. Outside of the Zero-Knowledge world, they have worked as powerful tools for:

- Searching for specific patterns within a text. For example, you might use a regex to find all occurrences of email addresses in a document.
- Substituting one pattern with another. For instance, you could use a regex to replace all instances of a specific word with a new one.
- Checking if a string matches a predefined format, such as ensuring an input is a valid phone number or email address.
- Breaking a string into substrings based on a pattern. For example, splitting a CSV string into its component values.

The standards for regular expression notation (and in fact, the notation recognized by this converter) are determined by literals, metacharacters, quantifiers, anchors, character classes and groups. In particular, they are defined by:

- A `\` followed by a character that is not a newline matches that character. This includes all special characters.
- A caret `^` matches the start of a line. It is also called a _starting anchor_
- A dollar sign `$` matches the end of a line. It is also called an _ending anchor_.
- A dot `.` matches any character.
- A single character that has no special meaning matches that character.
- A string enclosed in square brackets `[]` matches any character in the string. ASCII characters can also be abbreviated as `a-z0-9`. If a `^` is the first character inside the brackets, it matches any character that is NOT inside the brackets.
- A regular expression followed by an asterisk `*` matches a sequence of zero or more occurrences of the regular expression.
- A regular expression followed by a plus sign `+` matches a sequence of one or more occurrences of the regular expression.
- A regular expression followed by a question mark `?` matches a sequence of zero or one occurrence of the regular expression.
- Two concatenated regular expressions match an occurrence of the first followed by one of the second.
- Two regular expressions separated by a pipe `|` match an occurrence of the first or the second.
- Parentheses are used to group a regular expression (and then apply special characters as needed).
- Using `@` makes the regular expressions case-sensitive.
- Using `~` makes the regular expressions case-insensitive.

### Example of a regular expression
Consider a regular expression that recognizes strings containing an even number of \(1\)s:
$$r = 0^*(10^*10^*)^*$$

## Deterministic Finite Automaton (DFA)
It is a theoretical model used in computer science to represent and recognize patterns or sequences of symbols. It can be formalized as a \(5\)-uple \(M = \langle Q, \Sigma, \delta, q_0, F \rangle\) where:

- \(Q\) is the (finite) set of *states*.
- \(Sigma\) is the *input alphabet*, i.e. set of symbols that the DFA can read. For example, in a DFA designed to recognize binary numbers, the input alphabet would be \(\Sigma = \{0, 1\}\).
- \(\delta: Q \times \Sigma \to Q\) is the *transition function*. It maps a state and an input symbol to the next state. This function is defined for every combination of states and input symbols. The important thing it that it is *deterministic*, this means the automaton has a single and well-defined path to follow for each input string.
- \(q_0\) is the *initial state*. Every time \(M\) starts processing a string, it begins from this state. 
- \(F \subseteq Q\) is the set of *accepting states*. This states define that if the automaton ends up after processing the input string, the string is considered accepted.

A DFA \(M\) recieves an input string \(s \in \Sigma^*\) (where \(\Sigma\)^* is the _Kleene clause_ of \(Sigma\)), and processes it from its initial state using its transition function. If at the end, we are at one of the accepting states, the string is considered _recognized_ by the DFA. Therefore, we can formally define the _recognized language_ of \(M\) as the set of strings that recognizes, by
$$\mathcal{L}(M) = \{s \in \Sigma^*: \delta(q_0, s) \in F\}$$
where we are abusing notation to allow \(\delta\) recognize strings and not only characters.

Remember that a DFA is a theoretical model. When we implement it, we can allow more freedom or restrictions according to out neccesities. For example, the DFAs generated by ZK Regex have the additional restriction of only having one accepting state (\(|F| = 1\)). We also encourage to think the transition function of a DFA as a graph, where the states are nodes and reading a character represents an edge.

Another useful algorithm is DFA minimization. For every DFA \(M\), there exists another DFA \(M'\) such that \(\mathcal{L}(M) = \mathcal{L}(M')\) and \(M'\) has the least possible amount of states. ZK Regex just uses the same algorithm provided by the `DFA` Rust crate.

### Example of a DFA
Consider a simple DFA that recognizes binary strings containing an even number of \(1\)s:
$$M = \langle \{q_0, q_1\}, \{0, 1\}, \delta, q_0, \{q_0\}\rangle$$
The automaton is in the state \(q_0\) when it has recieved an even number of \(1\)s (that is why it is an accepting state), and it is in the state \(q_1\) when it has recieved an odd number of \(1\)s. Of course, its transition function is given by:

- \(\delta(q_0, 0) = q_0\)
- \(\delta(q_0, 1) = q_1\)
- \(\delta(q_1, 0) = q_1\)
- \(\delta(q_1, 1) = q_0\)

## Relation between regular expressions and DFA
The main result is that regular expressions and DFA are equivalent. This means that given any regular expression, you can construct a DFA that accepts the same language described by that regular expression. And conversely, for every DFA, there exists a regular expression that describes the language accepted by that DFA. This means that the set of strings accepted by a DFA can be expressed by some regular expression.

And in fact, this result is proven by construction! This means we know algorithms that can do this. In particular, the first part of ZK Regex works by converting a regular expression to a DFA. The difference is that there is more data, as described in the **Structures** page, that describes which sections of a regular expression must be public parameters or private parameters in the final circuit. Usually, the basic algorithm involves several steps:

- Converting Regex to NFA (Non-Deterministic Finite Automaton).
- Converting NFA to DFA.

There does exist crates for the basic algorithm and structures in Rust, but the algorithm needs to be modified because of the cryptographic requirements of ZK Regex. Refer to the **Obtaining DFA** page for more information about this implementation.

## Circom Circuits
TODO

### IsEqual

### LessEqualThan

### AND

### MultiOR

## Useful links
- [DFA](https://en.wikipedia.org/wiki/Deterministic_finite_automaton)
- [Regular Expressions](https://en.wikipedia.org/wiki/Regular_expression)
- [Experiment with regex!](https://regexr.com/)
- [PDF about theory of Automatons and Regular Expressions](https://www-2.dc.uba.ar/staff/becher/Hopcroft-Motwani-Ullman-2001.pdf)
