# Understanding ZK Regex
If you are not sure what a regular expression or a DFA are, we strongly recommend you to visit the **Concepts** page.

Use Zero-Knowledge Proofs and regular expressions to validate strings. An arithmetic circuit is generated using Circom.

Reasonable uses for this include verifying that a string (as a piece of information) follows specific patterns without revealing its content. Keep in mind that it is not possible to verify all possibilities for an infinite or excessively large language. Additionally, if we send the string to a smart contract, the "secret" information would be leaked (similar to how it would with IPFS).

In particular, the protocol is programmed in Rust and there are two high-level relevant steps:

- Convert the regular expression into a deterministic finite automaton (DFA), ideally minimal, using techniques of Language Theory. The goal is to preserve the parts indicated as secret in the regular expression as secret in the DFA. Remember that formally a DFA is a \(5\)-uple \(M = \langle Q, \Sigma, \delta, q_0, F \rangle\). In this project, different structures are used to represent (and hide) its data, but broadly speaking, it is thought of as a directed graph.
- Convert the DFA into a Circom circuit. Specifically, from the DFA, a `.circom` file is generated whose constraints represent the DFA.

## Useful links
- [ZK Regex](https://zkregex.com/)
- [ZK Email](https://prove.email/)
- [ZK Regex Graph Visualizer](https://zkregex.com/min_dfa) (this is outdated)
