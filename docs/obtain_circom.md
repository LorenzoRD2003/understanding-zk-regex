# Convert from DFA to a Circom circuit

The first function called is
```rust
fn generate_outputs(
  regex_and_dfa: &RegexAndDFA,
  halo2_dir_path: Option<&str>,
  circom_file_path: Option<&str>,
  circom_template_name: Option<&str>,
  num_public_parts: usize,
  gen_substrs: bool,
) -> Result<(), CompilerError>
``` 
As we will use Circom for this document, we assume that the `halo2_dir_path` input is `None`. The important parameter here is `regex_and_dfa`, which is a reference to the `RegexAndDFA` structure generated in the process described in **Obtaining the DFA**.

The other inputs are for us to specify the name of the file path to the `.circom` output file, the template name of the circuit in Circom, the number of public parts of the regular expression for which we generated the `RegexAndDFA` structure (although this is only used to generate multiple output file paths in the Halo2 version) and the `gen_substrs` bool, which purpose is to add substring constraints to the code generated by the following, and in fact most important function, `gen_circom_allstr`.

```rust
fn gen_circom_allstr( // every other parameter except for the template comes from the RegexAndDFA
  dfa_graph: &DFAGraph,
  template_name: &str, // name of the Circom template
  regex_str: &str,
  end_anchor: bool,
) -> String
```
(yes, all the Circom code will be in a very big string that can be saved as a `.circom` file)

The `dfa_graph`, `regex_str` and `end_anchor` parameters are references to parts of the `RegexAndDFA` structure generated in the the process described in **Obtaining the DFA**.

## Generate the reverse graph of `dfa_graph`
Given a directed graph \(G = (V, E)\), the reverse graph \(G^r = (V, E^r)\) is the directed graph with its edges reversed. This means that there exists an edge \(u \to v \in E\) if and only if there exists an edge \(v \to u \in E^r\).

In a DFA \(M\) with graph representation, it is not so simple to construct a reverse DFA \(M^r\), because we also need to assert there is only one starting state in \(M^r\), and there may be multiple accepting states in \(M\). It is possible to solve this, by creating an extra node with \(\lambda\)-transitions to each of the accepting states in \(M\) and setting this at the starting node in \(M^r\). However, the obtained structure is a NFA, and we have to determinize it.

We have explained this because it important to address that this is what the `build_reverse_graph` function DOES NOT DO, as it only reverts the edges of a (multi)graph and collects the accepting states of the `dfa_graph`.

The `build_reverse_graph` function recieves both the number of states and the `dfa_graph` and returns both the reverse graph (as a `Map<usize, Map<usize, Vec<u8>>>`) and the accepting states (as a `Set<usize>`). It is important to understand what this structure represents, as it maps the ID of a state \(q \in Q\) to the IDs of the states \(Q' \subseteq Q\) from which it recieved a transition in the `dfa_graph`, and it represents each of these states \(q' \in Q'\) as a map from their ID to a vector with the characters that allowed the transition \(q' \to q\) in the `dfa_graph`. Finally, the function also checks there exists an accepting state in the graph, panicking if not.

## Generate the state transition logic
Given the reverse graph `rev_graph`, the `generate_state_transition_logic` function creates the core logic for state transitions in the DFA, including range checks, equality checks, and multi-OR operations. In particular, it creates multiple lines of Circom code with calls to the basic components used. These components can be found in the **Concepts** section.

In fact, this function returns lots of things in a tuple:
- The number of `IsEqual` checks used.
- The number of `LessEqThan` checks used.
- The number of `AND` gates used.
- The number of `MultiOR` gates used.
- A `Vec<String>` containing the generated Circom code lines. This is the most important output of the function.

This function is very complex and its explanation will be divided in many subsections.

### Initialize counters and data structures for different types of checks
It defines mutable variables for the values that will be returned as output of the function, which are counters for the amount of gates of each type: 

- The number of `IsEqual` checks used.
- The number of `LessEqThan` checks used.
- The number of `AND` gates used.
- The number of `MultiOR` gates used.

It also defines the following mutable data structures:

- `range_checks` as a `Vec<Vec<usize; 256>; 256>` (in fact, with `Option`)
- `eq_checks` as a `Vec<usize; 256>` (in fact, with `Option`)
- `multi_or_checks1` as a `Map<String, usize>`
- `multi_or_checks2` as `Map<String, usize>`
- `zero_starting_states` as a `Vec<usize>`
- `zero_starting_and_idxes` as a `Map<usize, Vec<usize>>`
- `lines` as a `Vec<String>`. It just has every line of the `.circom` main component generated by this function.

### A bit about how the Circom circuit will have
We consider that there are \(n\) states in the DFA and that the message to validate with the regular expression is a string \(s\) of length \(k\) (with starting anchor `^`). Then, we will have the following signals

- `in[k]` as the input signals. We will have one for each character, where its value will be from \(0\) to \(255\).
- `states[k + 1][n]`, which represent the state transitions based on the input. In fact, at the beginning it will be `states[0][j] = 0` \(\forall 0 \leq j < n\), and we will calculate `states[i][j] = 1` if  and only if \(\delta(s_i, q_{j-1}) = q_j\) \(\forall 0 \leq i \leq k, \), which will translate to different code depending on the transition function context.
- `states_tmp[k + 1][n]`, which is a helper signal to handle complicated conditions. We will address where it is used.
- `state_changed[k]` that will have a boolean in the position \(i\) if there was progress when processing the \(i\)-th character of the string. This means we changed the state correctly. For this to happen, we need to have success at using the comparison components explained in **Concepts** to check if an input matches a character, a set of characters, or a range.
- `from_zero_enabled[k + 1]`, which checks that after processing the \(i\)-th character we are in the initial (zero) state. For example, one of its uses is that if we have `from_zero_enabled[i] = 1` and we are able to transition from state `0` to `1`, we have to go to state `1` directly.
- `is_consecutive[k + 1][n - 1]`, which address the question of keeping track of whether the matches are consecutive across the input, ensuring that patterns requiring consecutive elements are correctly validated.
- `final_state_result`, which is the output signal that is `0` or `1` depending if at some moment we managet to get to the UNIQUE final state of the DFA.

One example that usually appears in the code is that
$$\text{state_changed}[i] = \bigwedge_{i=1}^{n} \text{states}[i + 1][j]$$
is implemented as an `AND(n - 1)` component, which means that we change the state in the \(i\)-th step if we transition when we are reading the \(i\)-th element, something that is addressed in the \((i+1)\)-th row of `states`.

TODO: Add a simple example (regular expression `a[bc][de]*f` and input `abcdedef`).

Remember that not every DFA `M` will have a topologically sorted graph (i.e. it may have cycles), so it is not as sequential as we want it to be.

### Generate the Circom code for each state in the DFA
We will use the following auxiliar functions for each state in the DFA (going sequentially from \(0\) up to \(n - 1\)):

- Not an auxiliar function, but it updates the `zero_starting_states` and `zero_starting_and_idxes` data structures for the current state in the DFA.
- `optimize_char_ranges`:
- `add_range_check`:
- `add_eq_check`:
- `add_state_transition`:
- `add_state_update`:
- `add_from_zero_enabled`:
- `add_zero_starting_state_updates`:
- `add_state_changed_updates`:

## Generate the bureaucratic Circom code
The function `generate_declarations` adds everything that every `.circom` circuit should have, for example the `pragma`, the `include` declarations and the circuit template. However, there will have to be many parameters inside the template (for numerating the amount of components for each type of gates), and there is where we use the parameters recieved by the `generate_state_transition_logic` function.

The function `generate_init_code` generates the initialization code for the Circom circuit, by creating the code to initialize all states except the first one to 0.

The function `generate_accept_logic` recieves the set of accepting nodes and the boolean indicating if there is an ending anchor, and generates the acceptance logic for the Circom circuit. In particular, this function creates the code to check if the DFA has reached an accepting state, and handles the ending anchor logic in the case it is present. Unless there is exactly one accepting node, it will panic (this is supposed to be enhanced in future ZK Regex versions).

## Combine all the generated Circom code and return.
The final code is a `String` generated by the concatenation of the following vectors of strings:

- `declarations`, returned by the `generate_declarations` function.
- `init_code`, returned by the `generate_init_code` function.
- `lines`, returned by the `generate_state_transition_logic` function.
- `accept_lines`, returned by the `generate_accept_logic` function.

And it will be saved as a `.circom` file so everyone can use it!
