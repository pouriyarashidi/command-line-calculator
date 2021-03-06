# Command Line Calculator in Swift

![command line calculator](images/command-line.png)

## Overview

This repository contains a calculator program written in Swift 4.2 that can be used from the command line, along with this article which explains how the program works. There are suggested programming exercises at the bottom of this page, in case you are interested in adding new features to the calculator.

## Disclaimer 

The calculator program in this repository was written as a programming exercise, and is not intended for general usage. Using the calculator outside of its didactic capacity is strongly discouraged. The author is not to be held responsible for problems caused by or related to incorrect values produced by the program.

## Motivation
 
Sure, macOS comes with the built-in Calculator app and `bc` bash program. And there are plenty of free online calculators. The world doesn't need another way to crunch numbers on their Macs. But I didn't let that stop me from solving a very interesting problem!

This calculator accepts input like `"1 + 2 * 3 + -(4 - 5)"`, crunches the numbers according to the standard [order of operations](https://en.wikipedia.org/wiki/Order_of_operations), and prints out `8`. Creating a simple solution for this problem was a fun, tricky challenge.

## Data Transformations

At the highest level, this program turns a string into either a number or an error message. Assuming the user provides valid input, the next level down involves turning a string into a tree of arithmetic expressions. A properly constructed expression tree can be recursively evaluated to calculate the correct answer. An expression tree can be visualized like this:

![expression tree](images/expression-tree.png)

This program does not leap directly from user input to an expression tree. There are multiple data transformations involved, each of which analyzes incoming data and emits different data morphed one step closer to the final result. These data transformations are piped together, as seen in this method from [main.swift](calc/main.swift):

```swift
func calculate(_ input: String) -> CalcResult<Number> {
    return InputParser.createGlyphs(from: input)
        .then(Tokenizer.createTokens(fromGlyphs:))
        .then(Operationizer.createOperations(fromTokens:))
        .then(Expressionizer.createExpression(fromOperations:))
        .then(Calculator.evaluate(expression:))
}
```

The following sections describe each step in this data transformation pipeline.

### Text -> Glyphs

The user input is a string which may or may not contain valid arithmetic expressions. Each character in the string is  categorized by trying to create a `Glyph` value to represent it.

```swift
enum Glyph {

    case add
    case decimalSeparator
    case digit(UnicodeScalar)
    case divide
    case multiply
    case parenthesisLeft
    case parenthesisRight
    case subtractOrNegate
    case whitespace

}
```

Note: The `digit` case has an associated `UnicodeScalar` value. Since Swift strings are Unicode-compliant each character technically might consist of multiple scalars, but for the set of supported symbols in this program we don't need to worry about that. When you see `UnicodeScalar` in this program, just think of it as a character in a string.

Analysis of each user input character occurs in [input-parser.swift](calc/text%20-%3E%20glyphs/input-parser.swift). It uses `CharacterSet` from the `Foundation` framework to categorize a character. If it encounters an unrecognized symbol, an error is returned.

For example, `"45 + -3"` yields these `Glyph` values:

```swift
[
    .digit("4"),
    .digit("5"),
    .whitespace,
    .add,
    .whitespace,
    .subtractOrNegate,
    .digit("3")
]
```

At this point we know very little about what the user input means. The presence of a `-` character might mean subtraction or negation (hence the `Glyph` case named `subtractOrNegate`). Numbers are still broken up into individual digit values. This ambiguity can be reduced, thanks to the metadata provided by `Glyph`, to create more meaningful _tokens_.

### Glyphs -> Tokens

A `Token` is a symbol created from one or more `Glyph` values. 

```swift
enum Token {

    case add
    case divide
    case multiply
    case number(Number)
    case parenthesisLeft(negated: Bool)
    case parenthesisRight
    case subtract
    
}
```

Tokens are created from glyphs in [tokenizer.swift](calc/glyphs%20-%3E%20tokens/tokenizer.swift). This involves determining if `-` means to subtract or negate something, getting rid of whitespace, and combining various glyphs to create a `.number` token's associated `Number` value.

```swift
enum Number {

    case int(Int)
    case double(Double)
    
}
```

For example, `"10 * (5 - 1) + 2.0"` is represented by these `Token` values:

```swift
[
    .number(.int(10)),
    .multiply,
    .parenthesisLeft(negated: false),
    .number(.int(5)),
    .subtract,
    .number(.int(1)),
    .parenthesisRight,
    .add,
    .number(.double(2.0))
]
```

Having tokens brings us much closer to being able to create an expression tree and calculate the answer, but we're not quite there yet. A token array is a flat list while an expression tree is a highly nested binary tree. It would be easier to transform a token array to an expression tree via an intermediate representation which begins the nesting process, by shifting focus to binary operators and their operands, called _operations_.

### Tokens -> Operations

An `Operation` represents either an operator or operand in a calculation.

```swift
indirect enum Operation {

    case binaryOperator(BinaryOperator)
    enum BinaryOperator {
        case add, divide, multiply, subtract
    }

    case operand(Operand)
    enum Operand {
        case number(Number)
        case parenthesizedOperations([Operation], negated: Bool)
    }
    
}
```

This data representation categorizes a token into only two possible values: `.binaryOperator` or `.operand`. An operand can either be a numeric value or a nested `Operation` array. The code in [operationizer.swift](calc/tokens%20-%3E%20operations/operationizer.swift) creates operations from tokens.

For example, `"(7 + 3) * 3 + 12"` is represented by these operations:

```swift
[
    .operand(.parenthesizedOperations([
        .operand(.number(.int(7))),
        .binaryOperator(.add),
        .operand(.number(.int(3)))
        ], negated: false)),
    .binaryOperator(.multiply),
    .operand(.number(.int(3))),
    .binaryOperator(.add),
    .operand(.number(.int(12)))
]
```

At this point in the data transformation pipeline we have a hierarchical data format with all of the raw material needed to perform an arithmetic calculation. However, it does not include any notion of arithmetic's [order of operations](https://en.wikipedia.org/wiki/Order_of_operations) (remember BEDMAS or PEMDAS from grade school?). Having an `Operation` array does not help us calculate `1 + 2 * 3` and get `7` instead of `9`. Calculating each operation in the correct order can occur after transforming operations into an _expression_.

### Operations -> Expression

An `Expression` is the highest abstraction level in the calculator's data transformation pipeline.

```swift
indirect enum Expression {

    // Initial value
    case empty

    // Leaf node
    case number(Number)

    // Branch nodes
    case add(Expression, Expression)
    case divide(Expression, Expression)
    case multiply(Expression, Expression)
    case subtract(Expression, Expression)
    
}
```

Valid user input becomes an `Expression` value, which might be a single expression or a tree of expressions.

![expression tree](images/expression-tree.png)

Transforming an `Operation` array into an `Expression` tree involves two steps.
1. The `Operation` array is mapped to all possible `Expression` values (see [expressionizer.swift](calc/operations%20-%3E%20expression/expressionizer.swift)).
2. The array of all possible expressions is combined into an expression tree (see [expression-combining.swift](calc/operations%20-%3E%20expression/expression-combining.swift)).

Applying the order of operations occurs during the second step. I found this a particularly tricky problem, it's worth a look.

All that's left to do is evaluate the expression and calculate an answer.

### Expression -> Calculation

Calculating an answer involves recursing through the expression tree and accumulating a number. Recursing the nodes bottom-up and left-to-right ensures the expressions are evaluated according to the order of operations, as seen in [calculator.swift](calc/expression%20-%3E%20calculation/calculator.swift).

## But…how does it work?

This article describes the data transformations involved with calculating an answer, but it doesn't show how they work. Each section above links to relevant Swift file(s) that implement the transformations. I leave exploring those files up to you, if you're interested.

## Programming Exercises

Do you find this stuff as interesting as I do? Are you looking to get your hands dirty? If so, clone or download this repository and give these challenges a go!

1. Add support for `^` as an exponentiation operator. For example, `2 ^ 3` would evaluate to `8`.
2. Add support for the (unnecessary) unary `+` operator, which does not alter the value of its operand. It should be able to precede either a number or a left parenthesis. For example, `2 + +(4 - +5)`.
3. Add support for a locale-dependent decimal separator. Currently the program is hard-coded to use a decimal point, like `3.14`, but some cultures use a different separator, such as `3,14`. Check [NSLocale.decimalSeparator](https://developer.apple.com/documentation/foundation/nslocale/1643064-decimalseparator) for the appropriate separator to use.
