## What is the Interpreter pattern?

Interpreter defines a mini-language (a small grammar) as a set of classes. 
Each class represents one grammar rule and owns an interpret(context) method. 
A “sentence” in that language is built as a tree of those rule objects. 
Giving the tree a context (runtime data) lets every node evaluate itself and combine the results, 
ultimately producing the answer to “What does this sentence mean for this context?”


### Why people use it

When a project needs a simple but frequently changing or customer-facing DSL (Domain Specific Language).
For example, 
- discount rules in e-commerce (“subtotal > 100 AND country == US”),
- message routing selectors
it is often quicker and safer to embed an interpreter than to expose full Python or JavaScript code.
Business analysts can edit rules without touching core code, while developers keep strict control of what the embedded language can do.


### Where you see it in the wild?

- CSS selectors in browsers are evaluated by rule objects.
- SQL “query-planners” turn a parsed statement into an expression tree and interpret it row-by-row.
- Spreadsheet formulas,
- CI/CD declarative pipelines (GitHub Actions’ if:),
- feature-flag engines,
- e-mail filtering rules,
- firewall rules,
- CloudWatch metric alarms
these all embed tiny interpreters rather than general scripting engines.


### Advantages
- the language is tailored to the domain, so non-programmers can read and write it.
- grammar changes are local; add one class, get one new keyword.
- because interpreting walks a tree of objects, you can bolt on optimisers, partial evaluators, or pretty-printers by visiting that same tree.


### Disadvantages
- for anything beyond a toy grammar, hand-rolling an interpreter gets verbose: each new operator means another class. 
- tree walking is slower than code generation, so heavy use may hit performance limits.
- if the DSL keeps growing you eventually re-invent parts of a real compiler.


### Design considerations
- decide whether you need a full parser (tokens + precedence) or a simpler prefix / postfix syntax.
- constrain side-effects: interpreters are often embedded in high-privilege servers.
- if expressions are re-evaluated millions of times, add a cache or compile once to byte-code.
- provide tooling: error messages, formatting, syntax highlighting, so the DSL remains user-friendly.

