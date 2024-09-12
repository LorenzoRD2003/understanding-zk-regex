# Structures
All this structures derive the traits `Debug` and `Clone`. Besides, they are all serializable (using the `serde` crate) except from the structures that have the prefix `Info`.

In general, every `Map` and `Set` are implemented as `BTreeMap` and `BTreeSet`, while every `Deque` is implemented as `VecDeque`, but for simplicity of functionalities we will refer to them as the basic names in this document.

## Main structure
```rust
pub struct RegexAndDFA {
  pub regex_pattern: String,
  pub dfa: DFAGraph,
  pub has_end_anchor: bool,
  pub substrings: SubstringsDefinitions
}
```
For the creation of the circuit, we need to create the DFA from the regular expression. However, in the main structure we will save both of them. Also, we will have the `SubstringsDefinitions` for all the public parts.

Observe that the `regex_pattern` is a `String`. This means that we will concatenate all of the parts given by the JSON input file to the program into a single string.

The `has_end_anchor` means what we expect. Its purpose is... TODO

## RegexPart
```rust
pub struct RegexPartConfig {
  pub is_public: bool,
  pub regex_def: String
}

pub struct DecomposedRegexConfig {
  pub parts: Deque<RegexPartConfig>
}
```

The input to the program will be a JSON file containing an object with a single attribute `parts`, which is an array of objects with the parameters `is_public` and `regex_def` (those of the `RegexPartConfig` structure), from which we construct a `DecomposedRegexConfig`. An example of a JSON object for input is:
```json
{
  "parts": [
    {
      "regex_def": "m[01]+-",
      "is_public": false
    },
    {
      "regex_def": "[ab]+",
      "is_public": true
    },
    {
      "regex_def": ";",
      "is_public": false
    }
  ]
}
```

In turn, the input to the function `fn get_regex_and_dfa` will be a mutable reference to a `DecomposedRegexConfig`. The idea is for the program to analyze each part of the regular expression separately using the `regex_def` property.

The advantage of having multiple parts of the regular expression is that we can reveal some and keep others hidden (using the `is_public` property). As a limitation, the division into parts must occur after a step where the regular expression can be written as a concatenation of the parts (i.e., the parts must be valid). For example:

- The regular expression given by `[01]*[ab]+` can be separated in parts in multiple ways, for example: `[01]*`, `[ab]` and `[ab]*`.
- The regular expression given by `[[01]*[ab]*]*` cannot be separated in parts in any way. This means it will be completely public or completely private when we generate the code.

However, it is important to address that for nearly every useful regular expression, it will be possible to separate it in parts as we want.

## DFA
```rust
pub struct DFAStateInfo {
  pub typ: String,
  pub source: usize,
  pub edges: Map<String, usize>
}
pub struct DFAStateNode {
  pub state_type: String,
  pub state_id: usize,
  pub transitions: Map<usize, Set<u8>>
}

pub struct DFAGraphInfo {
  pub states: Vec<DFAStateInfo>
}

pub struct DFAGraph {
  pub states: Vec<DFAStateNode>
}
```
Let's assume that each node in the DFA will have an index for identification purposes. As we merge the graphs of each part into a single graph called `net_dfa_graph`, we need to update the set of states, and more importantly, the indices and edges of each node, to avoid index collisions. This renaming must not affect the structure of the DFA.

The `DFAStateInfo` is... TODO

We focus on the `DFAStateNode` structure: the `state_type` property indicates whether a state is final, the `state_id` property indicates the index of the state in the graph, and the `transitions` property is the set of edges, represented as a `Map<usize, Set<u8>>` where each node index has a set of valid characters (i.e., a `Set<u8>`) that allow the transition.

To facilitate the semantic analogy, the code refers to `transitions` as edges in the remaining functions, without considering the case of a multigraph (remember that in a graph that is not a multigraph, there can only be one edge between two given nodes, and if there is more than one character that allows a transition between two states in the DFA, there will be more than one edge).

## Substring Definitions
```rust
pub struct SubstringDefinitions {
  pub substring_ranges: Vec<Set<(usize, usize)>>,
  pub substring_boundaries: Option<Vec<(Set(usize), Set(usize))>>
}

pub struct SubstringDefinitionsJson {
  pub transitions: Vec<Vec<(usize, usize)>>
}
```

The idea is to have a `SubstringDefinitions` structure for each of the public parts of the regular expression.

Consider the context where we are merging a DFA \(N\) (the accumulated network) and \(M\) (corresponding to a public part). This \(M\) will have a range of the type `Set<(usize, usize)>` and a limit of the type `(Set(usize), Set(usize))`.

The limit is called so because when merging the network \(N\) and the public \(M\), where \(m\) is the maximum state index in \(N\), it is understood that \(m\) is the "limit" between \(N\) and \(M\) (as the states of \(M\) are renamed to have indices \( \geq m\)).

The range indicates the set of public edges in \(M\) and is essentially all its edges. That is, a set of pairs of `DFAStateNode.state_id`. We store this for future use, for the circuit generation.

Since \(m\) is an index in both DFAs, and particularly the initial state of \(M\), we have the classic DFA concatenation problem: obtaining \(NM\) (concatenated) where we can only transition from \(N\) to \(M\) via \(m\) and not from any state in \(F_N\), which is a problem regardless of whether \(m \in F_N\). The proposed solution is to replace in \(M\) each edge that includes \(m\) as follows:
$$\begin{aligned}
  m \rightarrow q &\text{ by } \{f \rightarrow q: f \in F_N\}, \quad (q \in Q_M \setminus \{m\})\\
  q \rightarrow m &\text{ by } \{q \rightarrow f: f \in F_N\}, \\
  m \rightarrow m &\text{ by } \{f \rightarrow f': f,f' \in F_N\}
\end{aligned}$$
and the resulting DFA is effectively the concatenation of \(N\) and \(M\).

The two sets of the limit indicate where the substring of the regular expression corresponding to this part begins and ends. The start is the set of `DFAStateNode.state_id` generated by the final states of \(N\), and the end is the set generated by the final states of \(M\).

Thus, a substring matches this part if and only if when we start, we are in an index from the first set, and when we finish, we are in an index from the second set.
