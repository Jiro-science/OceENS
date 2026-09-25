# LLM Utilities for OcéEns

Side tools around OcéEns's LLM providers.

## Cost tracking moved into the application

This folder used to contain `token-counting/`, which estimated the number of
tokens of the **repository's source code** (`*.py`) and multiplied it by a
hard-coded price. That measurement said nothing about what the application
actually spends: free-text summaries consume *prompt and response* tokens, not
Python files, and the "4 characters = 1 token" approximation matches no
provider's tokenizer.

Costs are now measured from the tokens each provider reports, inside the
application: see the "Summary costs" section of the root README.

---

## Upcoming utilities

This folder remains meant for LLM tools outside the application:

- prompt management and versioning;
- comparative model evaluation;
- provider switch-over scripts.
