Solving Data-flow problems in syntax trees

# First slide

- Hi, I'm ...
- just finished my master's thesis at KIT

# Introduction

- 1 minute
- was about merging Call Arity and the relevant parts of the Demand Analyser into a combined Usage Analysis
- usual traversal of syntax tree too rigid
- decouple specification of data-flow problem from computing its solution

# Strictness Analysis

- 2 minutes
- `e` strict in free var `x` if `e` evaluates `x` at least once
- computes, which free variables are used strictly
- `f a b c = if a then b else c*c` strict in `a`, but not in `b` or `c`
- Optimisation: Use Call by value without changing semantics
- (bang patterns)
- further optimisations through worker/wrapper transformation, which unboxes strict arguments

# Strictness signatures/LetDn

# Weakness of LetDn

# Analysis order decoupled from syntactic structure

# Data-flow graphs

- CFGs
- Why not hoopl?

# Example

# Code?

# Related Work

- hoopl

# Conclusion (Pros/Cons)