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


### Test example
Domain: e-mail filters 

A user writes rules such as
```
subject contains "invoice" AND from == "billing@example.com"
```
and the mail server decides whether to auto-label a message “Finance”.

```
rule     := term { (AND | OR) term }*
term     := IDENT (contains | ==) LITERAL
```

```python
"""
mini_email_filter.py
────────────────────
Run this file ➜ it evaluates one rule for three mock e-mails.

Output you will see
-------------------
Rule AST: ((subject contains 'invoice') AND (from == 'billing@example.com'))

Email#1  → label = YES
Email#2  → label = NO
Email#3  → label = NO
"""
from __future__ import annotations
from abc import ABC, abstractmethod
from typing import Dict

# ────────────────── 1. Expression hierarchy ──────────────────
Email = Dict[str, str]                 # very small “context” object

class Expr(ABC):
    @abstractmethod
    def interpret(self, msg: Email) -> bool: ...

class Condition(Expr):
    """Single comparison like `subject contains "foo"` or `from == "bob"`."""
    def __init__(self, field, op, value):
        self.field, self.op, self.value = field, op, value.strip('"')

    def interpret(self, msg):
        hay = msg.get(self.field, "").lower()
        needle = self.value.lower()
        if self.op == "contains":
            return needle in hay
        if self.op == "==":
            return hay == needle
        raise ValueError(f"Unknown op {self.op}")

    def __repr__(self):
        return f"({self.field} {self.op} {self.value!r})"

class And(Expr):
    def __init__(self, left, right):
        self.left, self.right = left, right
    def interpret(self, msg): return self.left.interpret(msg) and self.right.interpret(msg)
    def __repr__(self):       return f"({self.left} AND {self.right})"

class Or(Expr):
    def __init__(self, left, right):
        self.left, self.right = left, right
    def interpret(self, msg): return self.left.interpret(msg) or  self.right.interpret(msg)
    def __repr__(self):       return f"({self.left} OR {self.right})"

# ────────────────── 2. Super-light parser ──────────────────
def parse(rule: str) -> Expr:
    """
    Splits on spaces; builds left-associative tree.
    Grammar supports: <id> (contains|==) "literal"  joined by AND/OR
    """
    tokens = rule.split()
    def next_token(): return tokens.pop(0)

    # helper: consume one <id op literal> group
    def parse_term():
        field = next_token()
        op    = next_token()
        lit   = next_token()
        return Condition(field, op, lit)

    expr = parse_term()
    while tokens:
        conj = next_token()            # AND / OR
        rhs  = parse_term()
        expr = And(expr, rhs) if conj == "AND" else Or(expr, rhs)
    return expr

# ────────────────── 3. Demo ──────────────────
if __name__ == "__main__":
    RULE_TXT = 'subject contains "invoice" AND from == "billing@example.com"'

    ast = parse(RULE_TXT)
    print("Rule AST:", ast, "\n")

    emails = [
        {"subject": "March Invoice #4432", "from": "billing@example.com"},
        {"subject": "Hi there!",            "from": "friend@example.com"},
        {"subject": "Invoice reminder",     "from": "alerts@shop.com"},
    ]

    for i, msg in enumerate(emails, 1):
        label = "YES" if ast.interpret(msg) else "NO"
        print(f"Email#{i}  → label = {label}")
```


### Incremental Updates

Adding three useful features:
- NOT operator:	NOT subject contains "promo"
- startsWith comparison: from startsWith "sales@"
- numeric comparisons:	size > 1_000_000 (bytes, here)

```python
"""
mini_email_filter_v2.py
────────────────────────
Rule grammar now supports:
  - AND, OR, NOT      (NOT has higher precedence than AND/OR)
  - Comparisons:      ==, contains, startsWith, >, <, >=, <=
  - Literals:         "quoted strings"  or  numbers

No parentheses, still left-associative for AND/OR.

"""
from __future__ import annotations
from abc import ABC, abstractmethod
from typing import Dict

# ────────────────── Expression hierarchy ──────────────────
Email = Dict[str, str | int]          # add 'size' (int) alongside strings

class Expr(ABC):
    @abstractmethod
    def interpret(self, msg: Email) -> bool: ...

class Condition(Expr):
    """field op literal"""
    def __init__(self, field, op, value_token):
        self.field, self.op = field, op
        # drop quotes if present
        self.value_raw = value_token.strip('"')
        # detect numeric literal once
        self.value_num = None
        try:
            self.value_num = float(self.value_raw.replace('_', ''))
        except ValueError:
            pass                         # keep None for non-numeric

    def _num(self, thing):
        try:
            return float(thing)
        except (ValueError, TypeError):
            return None

    def interpret(self, msg):
        hay = msg.get(self.field)

        # Handle numeric > < >= <= if both sides numeric
        if self.op in {">", "<", ">=", "<="}:
            left = self._num(hay)
            right = self.value_num
            if left is None or right is None:
                return False
            if self.op == ">":  return left >  right
            if self.op == "<":  return left <  right
            if self.op == ">=": return left >= right
            if self.op == "<=": return left <= right

        # String ops (case-insensitive)
        if not isinstance(hay, str):
            return False
        hay = hay.lower()
        needle = self.value_raw.lower()

        if self.op == "contains":    return needle in hay
        if self.op == "startsWith":  return hay.startswith(needle)
        if self.op == "==":          return hay == needle
        return False                 # unknown op

    def __repr__(self):
        return f"({self.field} {self.op} {self.value_raw!r})"

class And(Expr):
    def __init__(self, l, r): self.l, self.r = l, r
    def interpret(self, msg): return self.l.interpret(msg) and self.r.interpret(msg)
    def __repr__(self):       return f"({self.l} AND {self.r})"

class Or(Expr):
    def __init__(self, l, r): self.l, self.r = l, r
    def interpret(self, msg): return self.l.interpret(msg) or self.r.interpret(msg)
    def __repr__(self):       return f"({self.l} OR {self.r})"

class Not(Expr):
    def __init__(self, e): self.e = e
    def interpret(self, msg): return not self.e.interpret(msg)
    def __repr__(self):       return f"(NOT {self.e})"

# ──────────────────  Super-light parser  ──────────────────
def parse(rule: str) -> Expr:
    """
    Grammar (EBNF-ish, left-associative):
        expr   := term { (AND | OR) term }*
        term   := [NOT] atom
        atom   := ID op LITERAL
    """
    tokens = rule.split()
    def next_tok(): return tokens.pop(0)

    def parse_atom():
        field = next_tok()
        op    = next_tok()
        lit   = next_tok()
        return Condition(field, op, lit)

    def parse_term():
        if tokens and tokens[0] == "NOT":
            next_tok()
            return Not(parse_atom())
        return parse_atom()

    expr = parse_term()
    while tokens:
        conj = next_tok()              # AND / OR
        rhs  = parse_term()
        expr = And(expr, rhs) if conj == "AND" else Or(expr, rhs)
    return expr

# ────────────────── Demo  ──────────────────
if __name__ == "__main__":
    RULE = ('NOT subject contains "promo" AND '
            'from startsWith "billing@" OR size > 1_000_000')

    ast = parse(RULE)
    print("Rule AST:", ast, "\n")

    mails = [
        # passes: meets size rule even though subject contains "promo"
        {"subject": "great PROMO inside", "from": "sales@shop.com", "size": 2_500_000},
        # fails: "promo" plus small size, wrong sender
        {"subject": "promo deal",         "from": "marketing@shop.com", "size": 400_000},
        # passes: from billing@, subject ok
        {"subject": "Invoice April",      "from": "billing@shop.com", "size": 120_000},
    ]

    for i, m in enumerate(mails, 1):
        res = "YES" if ast.interpret(m) else "NO"
        print(f"Mail #{i} → label = {res}")

"""
Expected output
---------------
Rule AST: (((NOT (subject contains 'promo')) AND (from startsWith 'billing@')) OR (size > '1000000'))

Mail #1 → label = YES
Mail #2 → label = NO
Mail #3 → label = YES
"""
```


[COMPLEX]
### Real-world-style example – a feature-flag rule engine

A SaaS platform wants product managers to target new features at subsets of users without deploying code. 
Rules look like:

```nginx
country == "US" AND (age >= 21 OR premium == true) AND NOT beta_opt_out
```

When an HTTP request arrives, the server loads the rule tree once and asks: “given this user context, is the flag ON?”

Below is a trimmed but fully working interpreter that:
- tokenises the rule string,
- parses it into a tree of Expression objects,
- interprets the tree against a user dict,
- supports AND, OR, NOT, ==, !=, >=, <=, >, <, parentheses, and booleans/strings/numbers.

```python
"""
feature_flag_interpreter.py
───────────────────────────
A minimal Interpreter-pattern rule engine for “Is this feature ON for user X?”

Rule grammar supported
----------------------
  • comparison:   country == "US", age >= 21, premium != true, …
  • boolean ops:  AND, OR, NOT
  • grouping:     parentheses
  • literals:     strings "…", numbers 12.3, booleans true/false
"""

# 1 ─────────────────────────────────────────────────────────────
# LEXER → turn raw text into tokens
# --------------------------------------------------------------

import re
from abc import ABC, abstractmethod
from typing import Any, List, Dict

# regex patterns + token type names
TOKEN_PATTERNS = [
    (r"\s+",              None),      # skip whitespace
    (r"==|!=|>=|<=|>|<",  "OP"),
    (r"\bAND\b",          "AND"),
    (r"\bOR\b",           "OR"),
    (r"\bNOT\b",          "NOT"),
    (r"\(",               "LPAR"),
    (r"\)",               "RPAR"),
    (r"true|false",       "BOOL"),
    (r'"[^"]*"',          "STRING"),
    (r"\d+(?:\.\d+)?",    "NUMBER"),
    (r"[A-Za-z_]\w*",     "ID"),
]

Token = tuple[str, str]               # e.g. ("ID", "country")

def lex(code: str) -> List[Token]:
    """
    >>> lex('country == "US" AND age >= 18')
    [('ID', 'country'), ('OP', '=='), ('STRING', '"US"'), ('AND', 'AND'),
     ('ID', 'age'), ('OP', '>='), ('NUMBER', '18')]
    """
    regex = "|".join(f"(?P<T{i}>{p})" for i, (p, _) in enumerate(TOKEN_PATTERNS))
    token_map = {f"T{i}": typ for i, (_, typ) in enumerate(TOKEN_PATTERNS) if typ}
    out = []
    for m in re.finditer(regex, code):
        group = m.lastgroup
        typ = token_map.get(group)
        if typ:                         # ignore whitespace
            out.append((typ, m.group().strip()))
    return out

# 2 ─────────────────────────────────────────────────────────────
# PARSER → convert tokens → AST (tree of Expression objects)
# (recursive-descent with precedence: OR < AND < NOT < comparisons)
# --------------------------------------------------------------

class Parser:
    def __init__(self, tokens: List[Token]):
        self.tokens = tokens
        self.pos = 0

    # helpers ---------------------------------------------------
    def _peek(self, typ): 
        return self.pos < len(self.tokens) and self.tokens[self.pos][0] == typ

    def _eat(self, typ) -> str:
        if not self._peek(typ):
            raise SyntaxError(f"Expected {typ}, found {self.tokens[self.pos]}")
        tok = self.tokens[self.pos][1]
        self.pos += 1
        return tok

    # public entry ---------------------------------------------
    def parse(self) -> "Expression":
        expr = self._parse_or()
        if self.pos != len(self.tokens):
            raise SyntaxError("Unexpected token after expression")
        return expr

    # grammar rules --------------------------------------------
    def _parse_or(self):
        expr = self._parse_and()
        while self._peek("OR"):
            self._eat("OR")
            expr = Or(expr, self._parse_and())
        return expr

    def _parse_and(self):
        expr = self._parse_not()
        while self._peek("AND"):
            self._eat("AND")
            expr = And(expr, self._parse_not())
        return expr

    def _parse_not(self):
        if self._peek("NOT"):
            self._eat("NOT")
            return Not(self._parse_not())
        return self._parse_comp()

    def _parse_comp(self):
        # parentheses first
        if self._peek("LPAR"):
            self._eat("LPAR")
            expr = self._parse_or()
            self._eat("RPAR")
            return expr
        # atom [op atom]?
        left = self._parse_atom()
        if self._peek("OP"):
            op = self._eat("OP")
            right = self._parse_atom()
            return Compare(left, op, right)
        return left

    def _parse_atom(self):
        if self._peek("ID"):
            return Var(self._eat("ID"))
        if self._peek("STRING"):
            return Literal(self._eat("STRING").strip('"'))
        if self._peek("NUMBER"):
            num = self._eat("NUMBER")
            return Literal(float(num) if "." in num else int(num))
        if self._peek("BOOL"):
            return Literal(self._eat("BOOL") == "true")
        raise SyntaxError("Unexpected token")

# 3 ─────────────────────────────────────────────────────────────
# AST node classes  (Interpreter pattern core)
# --------------------------------------------------------------

Context = Dict[str, Any]              # runtime values for one user

class Expression(ABC):
    @abstractmethod
    def interpret(self, ctx: Context) -> Any: ...

# leaf nodes ----------------------------------------------------
class Literal(Expression):
    def __init__(self, value): self.value = value
    def interpret(self, ctx):   return self.value
    def __repr__(self):        return f"{self.value!r}"

class Var(Expression):
    def __init__(self, name): self.name = name
    def interpret(self, ctx):  return ctx.get(self.name)
    def __repr__(self):       return self.name

# internal nodes ------------------------------------------------
class Compare(Expression):
    OPS = {
        "==": lambda a, b: a == b,
        "!=": lambda a, b: a != b,
        ">":  lambda a, b: a >  b,
        "<":  lambda a, b: a <  b,
        ">=": lambda a, b: a >= b,
        "<=": lambda a, b: a <= b,
    }
    def __init__(self, left, op, right):
        self.left, self.op, self.right = left, op, right
    def interpret(self, ctx):
        return self.OPS[self.op](self.left.interpret(ctx), self.right.interpret(ctx))
    def __repr__(self): return f"({self.left} {self.op} {self.right})"

class And(Expression):
    def __init__(self, l, r): self.l, self.r = l, r
    def interpret(self, ctx):  return self.l.interpret(ctx) and self.r.interpret(ctx)
    def __repr__(self):       return f"({self.l} AND {self.r})"

class Or(Expression):
    def __init__(self, l, r): self.l, self.r = l, r
    def interpret(self, ctx):  return self.l.interpret(ctx) or self.r.interpret(ctx)
    def __repr__(self):       return f"({self.l} OR {self.r})"

class Not(Expression):
    def __init__(self, e): self.e = e
    def interpret(self, ctx):  return not self.e.interpret(ctx)
    def __repr__(self):       return f"(NOT {self.e})"

# 4 ─────────────────────────────────────────────────────────────
# Convenience function: compile rule text → Expression tree
# --------------------------------------------------------------

def compile_rule(rule: str) -> Expression:
    tokens = lex(rule)
    return Parser(tokens).parse()

# 5 ─────────────────────────────────────────────────────────────
# Demo run
# --------------------------------------------------------------

if __name__ == "__main__":
    RULE_TXT = '''
        country == "US" AND
        (age >= 21 OR premium == true) AND
        NOT beta_opt_out
    '''.strip()

    # 1 Compile once at startup
    rule_ast = compile_rule(RULE_TXT)
    print("Parsed rule AST:", rule_ast)
    # ⇒ Parsed rule AST: ((country == 'US' AND ((age >= 21) OR (premium == True))) AND (NOT beta_opt_out))

    # 2 Evaluate for three user contexts
    users = [
        {"id": 1, "country": "US", "age": 25, "premium": False, "beta_opt_out": False},
        {"id": 2, "country": "CA", "age": 30, "premium": True,  "beta_opt_out": False},
        {"id": 3, "country": "US", "age": 17, "premium": True,  "beta_opt_out": True},
    ]

    for u in users:
        on = rule_ast.interpret(u)
        print(f"User#{u['id']:>2} → feature is {'ON ' if on else 'OFF'}")
    # ⇒ User# 1 → feature is ON 
    # ⇒ User# 2 → feature is OFF
    # ⇒ User# 3 → feature is OFF

```
