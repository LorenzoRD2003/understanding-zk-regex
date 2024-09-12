# Convert from regex to DFA
```rust
fn get_regex_and_dfa(
  decomposed_regex: &mut DecomposedRegexConfig, // regex parts
) -> Result<RegexAndDFA, CompilerError>
```

- Initialize all the (mutable) structures to be used:
  - A `DFAGraph` with an empty vector of states. This is called `net_dfa_graph`, because later we will loop through, creating a new `DFAGraph` for each part of the regular expression and concatenating it to the network.
  - Two empty vectors for the `substring_ranges` and `substring_boundaries` arrays of `SubstringDefinitions`.
- Search for the first `^` that is not within parentheses in the regular expression, if there is one (it is an `Option<usize>` indicating the position of the symbol within the regular expression). If it exists, divide the regular expression into two parts.
- For each part of the regular expression in `decomposed_regex`, do the following:
  - None of the parts of the regular expression should end with `$` except for the last one. Therefore, check that if it is not the last part, it does not end with `$`, and if it is the last part, obtain a `bool` indicating whether it ends with `$` and store this in the `RegexAndDFA` structure under the property `has_end_anchor`.
  - Construct a DFA for the part in question. In particular, it has a default configuration which is understood through the `regex_automata::dfa` crate, and the DFA is built for `^regex$` rather than just `regex`.
  - Transform that DFA into a graph using the `convert_dfa_to_graph` function, obtaining a `DFAGraph` called `dfa_graph`. This function may require its own analysis subsection, but it is not included as it falls outside the ZK scope.
  - If there is a `^` in the first part, modify the `DFAGraph` (to be able to accept the empty string, handling several cases depending on whether the whole part is just a `^` or not).
  - Obtain the highest state index in `net_dfa_graph`, as we want to concatenate `dfa_graph` and avoid index collisions. Based on this index, rename the states of `dfa_graph`.
  - If the part is public, process it (as indicated in the `SubstringDefinitions` subsection) to obtain the data to be stored in the `SubstringDefinitions` structure.
  - Effectively concatenate `dfa_graph` to `net_dfa_graph` (though this merely extends the vector of states).
- Construct the `RegexAndDFA` structure from the obtained parameters. Specifically, the regular expression stored in the structure is the concatenation of the parts, while the logic of which parts to keep secret is incorporated into the `net_dfa_graph`.

