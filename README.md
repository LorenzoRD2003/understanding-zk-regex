# Understanding ZK Regex
Started by Lorenzo Ruiz Díaz as a public contribution for the PSE Core Program 2024.

## Roadmap to 1.0.0 version 🎯
Right now we are like in a 0.4.0 version (??). Before releasing the 1.0.0 version of this documentation, more tasks need to be finished. These are the tasks that remain to do (as 2024-09-13):

- Add an example to the algorithm of conversion from regular expression to DFA. 📌
- Complete the **Circom components** subsection in `preconcepts.md`. 📌
- Explain the use of each data structure used inside the `generate_state_transition_logic` function:
  - `range_checks`. 📌
  - `eq_checks`. 📌
  - `multi_or_checks1`. 📌
  - `multi_or_checks2`. 📌
  - `zero_starting_states`. 📌
  - `zero_starting_and_idxes`. 📌
  - `lines`. ✅
- Add a simple example (with tables) of the algorithm execution for `generate_state_transition_logic`. 📌
- Explain the use of each  of the following functions inside the `generate_state_transition_logic` function:
  - `optimize_char_ranges`. 📌
  - `add_range_check`. 📌
  - `add_eq_check`. 📌
  - `add_state_transition`. 📌
  - `add_state_update`. 📌
  - `add_from_zero_enabled`. 📌
  - `add_zero_starting_state_updates`. 📌
  - `add_state_changed_updates`. 📌
- Complete and enhance the `generate_accept_logic` paragraph. 📝
- Talk about the `is_reveal` signals and the substring logic for the Circom circuit. 📌
- Enhance grammar and spelling. Also check if some concepts could be better explained. 📌

I expect it to take me about two to three more weeks of work. And these are the already finished tasks (not broken down into subtasks):
- Finish the `index.md` file. ✅
- In `preconcepts.md`:
  - Finish the regular expressions section. ✅
  - Finish the DFA section. ✅
  - Finish the explanation of the regular expression to DFA algorithm in. ✅
  - Add useful links section. ✅
- Finish the `structures.md` file. ✅
- Finish the `obtain_dfa.md` file. ✅

## How to use?
Just enter this URL: 
