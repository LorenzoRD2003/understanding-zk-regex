# Convert from DFA to .circom

## Obtain Circom code (yes, in a String!)
```rust
fn gen_circom_allstr( // every other parameter except for the template comes from the RegexAndDFA
  dfa_graph: &DFAGraph,
  template_name: &str, // name of the Circom template
  regex_str: &str,
  end_anchor: bool,
) -> String
```
- Se construye el AFD en reversa de `dfa_graph`. Esta parte devuelve un `(Map<usize, Map<usize, Vec<u8>>>, Set<usize>)`.
  - El primer elemento de la tupla es el AFD en reversa. La notación quiere decir que el mapeo $\sigma: Q \to \mathcal{P}(Q)$ está dado por
  $$\sigma(q) = $$
  - El segundo elemento de la tupla es el conjunto de estados finales.
