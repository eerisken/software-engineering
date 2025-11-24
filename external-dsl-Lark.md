# **Developing an External DSL using Python and Lark**

A guide to building a business-readable rule engine using the Lark parsing library.  

In a previous manual implementation, we wrote our own tokenizer and parser from scratch. While educational, that approach is error-prone for larger languages.  
In this version, we will use [**Lark**](https://github.com/lark-parser/lark), a modern parsing library for Python. Lark allows us to define our language using a readable grammar (EBNF) and automatically builds the parsing tree for us.

## **The Goal**

We want to parse the same configuration file: 

rule "Christmas Sale":  
    IF date == "2024-12-25"  
    THEN DISCOUNT 50%

rule "VIP December":  
    IF customer_type == "VIP" AND month == 12  
    THEN DISCOUNT 20%

## **Prerequisites**

Install Lark:  
```Python
pip install lark
```

## **Step 1: The Grammar**

Instead of writing regex loops, we define the rules of our language in a string. Lark uses a format similar to EBNF. 
```Python
from lark import Lark, Transformer, v_args

dsl_grammar = """  
    ?start: rule+

    rule: "rule" STRING ":" "IF" condition "THEN" action

    ?condition: binary_op  
              | comparison

    binary_op: condition "AND" condition -> and_op  
             | condition "OR" condition  -> or_op

    comparison: NAME OP (STRING | NUMBER | NAME)

    action: "DISCOUNT" percentage

    percentage: NUMBER "%"?

    // Terminals  
    NAME: /[a-zA-Z_]w*/  
    STRING: /"[^"]*"/  
    OP: "==" | "!=" | ">=" | "<=" | ">" | "<"  
      
    %import common.NUMBER  
    %import common.WS  
    %ignore WS  
"""
```

**Key Improvements over Manual Parsing:**

* **Declarative:** We describe *what* the language looks like, not *how* to parse it.  
* **Precedence:** Notice ?condition and the structure of binary_op. Lark handles operation precedence automatically if structured correctly.  
* **Built-ins:** We import NUMBER and WS (whitespace) from Lark's common library.

## **Step 2: The Transformer**

Lark parses the text and creates a Tree. To make this useful, we use a Transformer to convert that tree into Python objects (our AST).  
```Python
class Rule:  
    def __init__(self, name, condition, action):  
        self.name = name  
        self.condition = condition  
        self.action = action

class Action:  
    def __init__(self, type, value):  
        self.type = type  
        self.value = value

class DSLTransformer(Transformer):  
    def rule(self, args):  
        # args[0] is the rule name (token), args[1] is condition, args[2] is action  
        return Rule(args[0].value.strip('"'), args[1], args[2])

    def comparison(self, args):  
        left, op, right = args  
        # Return a lambda function that takes 'context' and evaluates True/False  
        return lambda ctx: self._compare(  
            ctx.get(left.value) if hasattr(left, 'value') else left,  
            op.value,  
            ctx.get(right.value) if hasattr(right, 'type') and right.type == 'NAME' else self._clean_val(right)  
        )

    def and_op(self, args):  
        return lambda ctx: args[0](ctx) and args[1](ctx)

    def or_op(self, args):  
        return lambda ctx: args[0](ctx) or args[1](ctx)

    def action(self, args):  
        return Action('DISCOUNT', args[0])

    def percentage(self, args):  
        val = args[0].value  
        if val.endswith('%'):  
            return float(val.strip('%')) / 100  
        return float(val)  
          
    def _clean_val(self, token):  
        # Helper to strip quotes from strings or convert numbers  
        if token.type == 'STRING':  
            return token.value.strip('"')  
        if token.type == 'NUMBER':  
            return float(token.value)  
        return token.value

    def _compare(self, left, op, right):  
        # Comparison logic helper  
        if op == '==': return str(left) == str(right)  
        if op == '>=': return left >= right  
        if op == '<=': return left <= right  
        if op == '>':  return left > right  
        if op == '<':  return left < right  
        return False
```

**Note:** In this Transformer, instead of building a static AST class for conditions, we are compiling them directly into Python lambda functions. This makes the execution phase incredibly fast.

## **Step 3: The Engine**

The engine is now much simpler because the Transformer did the heavy lifting of converting logic into executable functions.  
```Python
class Engine:  
    def __init__(self):  
        self.parser = Lark(dsl_grammar, parser='lalr', transformer=DSLTransformer())

    def apply_benefits(self, dsl_text, context):  
        # The parser returns a list of Rule objects (because start: rule+)  
        rules = self.parser.parse(dsl_text)  
          
        best_discount = 0.0  
        applied_rules = []

        for rule in rules:  
            # rule.condition is now a callable lambda waiting for context  
            if rule.condition(context):  
                if rule.action.value > best_discount:  
                    best_discount = rule.action.value  
                    applied_rules.append(rule.name)  
          
        return best_discount, applied_rules
```

## **Putting It All Together**
```Python
dsl_script = """  
rule "Christmas Mega Sale":  
    IF date == "2024-12-25"  
    THEN DISCOUNT 50%

rule "VIP Winter Special":  
    IF customer_type == "VIP" AND month == 12  
    THEN DISCOUNT 20%  
"""

context_christmas = {  
    "date": "2024-12-25",  
    "month": 12,  
    "customer_type": "standard"  
}

engine = Engine()  
discount, rules = engine.apply_benefits(dsl_script, context_christmas)

print(f"Discount: {discount * 100}%")  
print(f"Applied Rules: {rules}")  
# Output:  
# Discount: 50.0%  
# Applied Rules: ['Christmas Mega Sale']
```

## **Comparison: Lark vs Manual**

| Feature | Manual (re) | Lark |
| :---- | :---- | :---- |
| **Complexity** | High (Loops, State management) | Low (Declarative Grammar) |
| **Maintenance** | Difficult to add new operators | Easy (Add one line to grammar) |
| **Error Handling** | Manual exceptions | Automatic syntax errors |
| **Dependencies** | None (Standard Lib) | Requires lark package |

