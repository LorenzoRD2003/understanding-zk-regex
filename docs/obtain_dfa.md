# Convert from regex to DFA
```rust
fn get_regex_and_dfa(
  decomposed_regex: &mut DecomposedRegexConfig, // regex parts
) -> Result<RegexAndDFA, CompilerError>
```
The purpose of this function if to obtain the main structure than the function which will convert from DFA to Circom code will use. Refer to the **Structures** page for understanding its design.

## Initialize all the (mutable) structures to be used
- A `DFAGraph` with an empty vector of states. This is called `net_dfa_graph`, because later we will loop through the `decomposed_regex`, creating a new `DFAGraph` for each part of the regular expression and concatenating it to the network.
- Two empty vectors for the `substring_ranges` and `substring_boundaries` arrays of `SubstringDefinitions`.
- A default configuration for the DFA (`create_dfa_config` function), it is a `Config` object as defined in the `regex_automata::dfa` Rust crate. Particularly, it states that the generated DFA should be minimized, that it should start anchored (i.e. with a _starting anchor_ `^`), that it will not have byte classes, and that it will have acceleration enabled.

### Search for the first caret
This is achieved by the `process_caret_in_regex` function. Search for the first caret `^` that is not within parentheses in the regular expression, if there is one (it is an `Option<usize>` indicating the position of the symbol within the regular expression, so if there does not exist, it will return `None`).

If it exists, divide the regular expression into two parts, before (non-inclusive) and after (inclusive) the caret (and updating the `Deque` in the `decomposed_regex` accordingly). The regular expression part before the caret will be set to private, so it will be guaranteed that the second element of the `Deque` is a regular expression part which starts with a caret.

Remember from the **Preconcepts** page that a caret in a regular expression matches the start of a line (or in the usual case, the beginning of a regular expression). This is why it is also called a _starting anchor_, and it means that we should be looking for a match at the beginning of a string.

## Main cycle
For each part of the regular expression in `decomposed_regex`, do the following:

### Validate end anchor
The important thing to address here is that none of the parts of the `decomposed_regex` should end with `$` except for the last one. Therefore, the `validate_end_anchor` function checks that if we are not processing the last part in the `decomposed_regex`, it does not end with `$`, and if it is the last part, obtain a `bool` indicating whether it ends with `$` and store this in the `RegexAndDFA` structure under the property `has_end_anchor`.

### Construct the DFA for each part
The algorithm constructs a DFA for the part in question. This is done by the `DFA::builder()` function of the `regex_automata::dfa` Rust crate, with the particular configuration indicated in a previous subsection. This returns... TODO

### Transform the DFA into a `DFAGraph`
The algorithm transforms that DFA (represented as a `DFA<Vec<u32>>` structure of the `regex_automata::dfa` Rust crate) into a graph using the `convert_dfa_to_graph` function, obtaining a `DFAGraph` structured called `dfa_graph`. In particular, the function does these things:

- Converts the DFA to a string representation. This is with the `format!` Rust macro.
- Creates an empty `DFAGraphInfo` structure (refer to **Structures** for more information), which will be populated with the next step.
- Parses states from the string representation with the `parse_states` function, which will recieve as parameters mutable references to both the string representation and the `DFAGraphInfo` structure. This function will be better explained in an update of this documet, but as a resume it uses regular expressions to match state definitions and transitions in the input string, iterating over state matches in order to create `DFAStateInfo` objects for each state (each node in the graph), then it parses transitions for each state (with the `parse_transition`) function to populate the edges of that state, and finally it populates the `states` parameter of the `DFAGraphInfo` with the parsed states.
- Handles End of Input (EOI) transitions. In particular, if an state \(q\) has a transition to another through the EOI symbol, it will be removed from the edges, and \(q\) will be marked as an accepting state (with the `typ` property in the `DFAStateInfo` structure).
- Finds the start state of the string representation of the DFA using `find_start_state`, obtaining its ID.
- Sorts and renames states accordingly given the `DFAGraphInfo` structure and the start identifier. This is achieved by performing a Performs a BFS (with a queue implemented as a `VecDeque`) in the graph from the start state, creating a mapping of old states IDs to new ones, renaming states and updating their edges according to the mapping. This means each `DFAStateInfo` is modified, but the obtained `DFAGraphInfo` is equivalent to the original (in the sense that the formal DFA they represent is the same, or at least recognizes the same language).
- Before converting the `DFAGraphInfo` structure into a `DFAGraph`, it has to call the function `process_state_edges` because the properties for the edges in the `DFAGraphInfo` and `DFAGraph` structures are different. It converts from `Map<String, usize>` to `Map<usize, Set<u8>>`. In particular, remember that for efficiency, there are two types of edges in `DFAGraphInfo`, single character edges and range edges, so it has to address these different cases in the `process_edge` function. 
- With the transitions of the expected type, it converts the `DFAGraphInfo` structure into a `DFAGraph` that which will put inside the `RegexAndDFA` and be used by the Circom code generator. We obtain a `dfa_graph` (remember that an important property of this structure is that it only has one accepting state).

### Handle a caret in the first part
If there is a caret `^` in the first part of the `decomposed_regex`, the `dfa_graph` returned by the previous function is not able to accept the empty string, but it should! Therefore, there is a `handle_regex_caret` function for this. It addresses two cases:

- The regex is `^`. In this case the graph generated will be very simple as it only contains two states and one transition from the starting state to the accepting state for the character of byte value \(255\).
- The regex is not `^`. In this case it modifies the `dfa_graph` by clearing the the state type of its initial state, finding its accepting state and adding a transition from the start state to the accept state for the character of byte value \(255\).

### Rename the states
We want to concatenate the new `dfa_graph` for the current `decomposed_regex` into the `net_dfa_graph`, but we want to avoid ID collisions. To achieve this, we have to obtain the highest state ID in the `net_dfa_graph`. Remember that each ID is just a number, so if the maximum index for `net_dfa_graph` is \(m\), it is possible to rename the `dfa_graph` states as \([m, m + 1, \ldots, m + k - 1]\) if it has \(k\) states.

### If the `decomposed_regex` part is public...
We have to process the regular expression and both the `net_dfa_graph` and `dfa_graph` before the concatenation, in a function called `process_public_regex`. This is to be able to obtain the data needed to be stored by the `SubstringDefinitions` structure. It is indicated how this works and why it is necessary in the (`SubstringDefinitions` subsection in **Structures**).

### Concatenate the graph to the network
Effectively concatenate `dfa_graph` to `net_dfa_graph`, returning a new `DFAGraph` which will be saved as the new `net_dfa_graph` (even though this `add_dfa` function merely extends the vector of states).

## Construct the main structure
From the parameters obtained, we are able to construct the `RegexAndDFA` structure (refer to **Structures** to remember which they are).

Specifically, the regular expression stored in the structure is the string concatenation of the parts of the `decomposed_regex`, while the logic of which parts to keep secret is incorporated into the `net_dfa_graph`.

