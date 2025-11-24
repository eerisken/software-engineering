# **Developing an External DSL using Python**

A guide to building a business-readable rule engine for temporal benefits.  
Domain Specific Languages (DSLs) allow us to decouple complex business logic from our application code. While Python itself is readable, allowing business analysts or non-developers to define rules in a language that looks like English can be a game-changer for systems requiring frequent updates—like dynamic pricing or holiday events.  
In this article, we will build a simple **External DSL** to apply temporal benefits (discounts) based on customer attributes and dates (e.g., Christmas or VIP status).

## **The Goal**

We want to parse and execute a configuration file that looks like this:  

rule "Christmas Sale":  
    IF date == "2024-12-25"  
    THEN DISCOUNT 50%

rule "VIP December":  
    IF customer_type == "VIP" AND month == 12  
    THEN DISCOUNT 20%

## **The Architecture**

To build this, we need three distinct components:

1. **Lexer (Tokenizer):** Breaks the raw text into meaningful chunks (tokens).  
2. **Parser:** Organizes those tokens into a structured tree (Abstract Syntax Tree).  
3. **Interpreter:** Traverses the tree and executes logic against a data context.

## **Step 1: The Tokenizer**

We'll use Python's built-in re module. We need to define regex patterns for keywords, operators, strings, and numbers.  
```Python
import re  
from typing import NamedTuple, List, Any

class Token(NamedTuple):  
    type: str  
    value: str

def tokenize(code: str) -> List[Token]:  
    token_specification = [  
        ('NUMBER',   r'd+(.d*)?%?'),  # Integer or decimal, optional %  
        ('STRING',   r'"[^"]*"'),        # Double-quoted string  
        ('ID',       r'[A-Za-z_][A-Za-z0-9_]*'), # Identifiers  
        ('OP',       r'[=!<>]=?|AND|OR'), # Operators  
        ('COLON',    r':'),  
        ('NEWLINE',  r'n'),  
        ('SKIP',     r'[ t]+'),         # Skip over spaces  
        ('MISMATCH', r'.'),              # Any other character  
    ]  
      
    tok_regex = '|'.join('(?P<%s>%s)' % pair for pair in token_specification)  
    tokens = []  
      
    for mo in re.finditer(tok_regex, code):  
        kind = mo.lastgroup  
        value = mo.group()  
        if kind == 'NUMBER':  
            if value.endswith('%'):  
                value = float(value.strip('%')) / 100  
            else:  
                value = float(value)  
        elif kind == 'STRING':  
            value = value.strip('"')  
        elif kind == 'SKIP':  
            continue  
        elif kind == 'MISMATCH':  
            raise RuntimeError(f'Unexpected value: {value}')  
          
        tokens.append(Token(kind, value))  
    return tokens
```

## **Step 2: The Parser**

The parser takes the list of tokens and ensures they follow our grammar. We will build a simple **Abstract Syntax Tree (AST)**.  
```Python
# AST Nodes  
class Rule:  
    def __init__(self, name, condition, action):  
        self.name = name  
        self.condition = condition  
        self.action = action

class BinaryOp:  
    def __init__(self, left, op, right):  
        self.left = left  
        self.op = op  
        self.right = right

class Action:  
    def __init__(self, type, value):  
        self.type = type  
        self.value = value

class Parser:  
    def __init__(self, tokens):  
        self.tokens = tokens  
        self.pos = 0

    def consume(self, expected_type=None):  
        if self.pos < len(self.tokens):  
            token = self.tokens[self.pos]  
            if expected_type and token.type != expected_type:  
                raise SyntaxError(f"Expected {expected_type}, got {token.type}")  
            self.pos += 1  
            return token  
        return None

    def parse(self):  
        rules = []  
        while self.pos < len(self.tokens):  
            # Skip empty lines  
            while self.pos < len(self.tokens) and self.tokens[self.pos].type == 'NEWLINE':  
                self.consume()  
            if self.pos >= len(self.tokens): break  
              
            rules.append(self.parse_rule())  
        return rules

    def parse_rule(self):  
        # Grammar: rule "Name": IF <condition> THEN <action>  
        if self.consume('ID').value != 'rule': raise SyntaxError("Expected 'rule'")  
        rule_name = self.consume('STRING').value  
        self.consume('COLON')  
        self.consume('NEWLINE')  
          
        if self.consume('ID').value != 'IF': raise SyntaxError("Expected 'IF'")  
        condition = self.parse_condition()  
        self.consume('NEWLINE')  
          
        if self.consume('ID').value != 'THEN': raise SyntaxError("Expected 'THEN'")  
        action = self.parse_action()  
          
        return Rule(rule_name, condition, action)

    def parse_condition(self):  
        # Simple Left OP Right parsing  
        left = self.consume() # ID (e.g., customer_type)  
        op = self.consume('OP')  
        right = self.consume() # Value (String/Num) or ID  
          
        # Look ahead for AND/OR  
        if self.pos < len(self.tokens) and self.tokens[self.pos].value in ('AND', 'OR'):  
            logic_op = self.consume('OP')  
            next_condition = self.parse_condition()  
            return BinaryOp(BinaryOp(left, op.value, right), logic_op.value, next_condition)  
              
        return BinaryOp(left, op.value, right)

    def parse_action(self):  
        # Grammar: DISCOUNT <number>  
        action_type = self.consume('ID').value # DISCOUNT  
        value = self.consume('NUMBER').value  
        return Action(action_type, value)
```

## **Step 3: The Interpreter**

The interpreter runs the logic. It takes a **Context** (a dictionary containing the current customer and date data) and applies the AST rules against it.  
import datetime
```Python
class Engine:  
    def __init__(self):  
        self.rules = []

    def load_rules(self, dsl_text):  
        tokens = tokenize(dsl_text)  
        parser = Parser(tokens)  
        self.rules = parser.parse()

    def evaluate_condition(self, node, context):  
        if isinstance(node, BinaryOp):  
            # Handle Logical AND/OR  
            if node.op == 'AND':  
                return self.evaluate_condition(node.left, context) and   
                       self.evaluate_condition(node.right, context)  
            if node.op == 'OR':  
                return self.evaluate_condition(node.left, context) or   
                       self.evaluate_condition(node.right, context)

            # Handle Comparisons  
            # Resolve left side (usually a variable from context)  
            left_val = context.get(node.left.value) if node.left.type == 'ID' else node.left.value  
            right_val = context.get(node.right.value) if node.right.type == 'ID' else node.right.value

            # Simple comparison logic  
            if node.op == '==': return str(left_val) == str(right_val)  
            if node.op == '>=': return left_val >= right_val  
            if node.op == '<=': return left_val <= right_val  
            if node.op == '>':  return left_val > right_val  
            if node.op == '<':  return left_val < right_val  
              
        return False

    def apply_benefits(self, context):  
        best_discount = 0.0  
        applied_rules = []

        for rule in self.rules:  
            if self.evaluate_condition(rule.condition, context):  
                if rule.action.type == 'DISCOUNT':  
                    # We simply take the max discount available in this example  
                    if rule.action.value > best_discount:  
                        best_discount = rule.action.value  
                        applied_rules.append(rule.name)  
          
        return best_discount, applied_rules
```

## **Putting It All Together**

Let's test our engine with a specific date and customer type.  
```Python
# 1. Define the Rules  
dsl_script = """  
rule "Christmas Mega Sale":  
    IF date == "2024-12-25"  
    THEN DISCOUNT 50%

rule "VIP Winter Special":  
    IF customer_type == "VIP" AND month == 12  
    THEN DISCOUNT 20%  
      
rule "Standard Member":  
    IF customer_type == "standard"  
    THEN DISCOUNT 5%  
"""

# 2. Initialize Engine  
engine = Engine()  
engine.load_rules(dsl_script)

# 3. Create Contexts (Simulation)  
context_christmas = {  
    "date": "2024-12-25",  
    "month": 12,  
    "customer_type": "standard"  
}

context_vip_early = {  
    "date": "2024-12-10",  
    "month": 12,  
    "customer_type": "VIP"  
}

# 4. Run  
discount, rules = engine.apply_benefits(context_christmas)  
print(f"Christmas Context: {discount*100}% off via {rules}")  
# Output: Christmas Context: 50.0% off via ['Standard Member', 'Christmas Mega Sale']

discount, rules = engine.apply_benefits(context_vip_early)  
print(f"VIP Context:       {discount*100}% off via {rules}")  
# Output: VIP Context:       20.0% off via ['VIP Winter Special']
```

## **Conclusion**

By implementing a Tokenizer, Parser, and Interpreter, we've created a safe, sandboxed environment for defining business rules. This approach prevents the risks of using Python's eval() and allows us to define a language that perfectly fits the business domain.
