Solving Data-flow problems in syntax trees

# First slide

- Hi, I'm ...
- just finished my master's thesis (at KIT)
- originally, I planned to talk about that
- instead I'm gonna talk about an interesting byproduct that is more generally applicable

# Introduction

- 2 minutes
- So, what exactly was my thesis about?
- Two analyses in GHC, CA and DA, that overlap in concerns
- I extracted overlaps into a combined Usage Analysis generalising CA and its counterpart in DA
- Surprising result: Precision of CA achievable without co-call graphs
- Now, if those words don't mean anything to you, that's OK
- Otherwise or if you are interested, I encourage you to skim through my master's thesis here
- In order to actually generalise both analyses, the analysis order became quite complex
- In particular, iterative traversal of the syntax tree proved to be too restrictive
- Had to decouple specification of data-flow problem from computing its solution
- We'll reenact this claims with the help of a strictness analysis

# Strictness Analysis

- 1 minute
- provides lower bounds on *evaluation cardinality*
- answers the question of which variables are evaluated at least once
- a property we call strictness
- If an expression uses a variable at least once, then the expression is strict in that expression
- In the cases where the strictness analysis is unsure, we conservatively assume a lazy use
- example: strict in `x` and `y`, but not in `z`
- Optimisation: Use Call by value without changing semantics
- (bang patterns)
- further optimisations through worker/wrapper transformation, which unboxes strict arguments

# GHC's Demand Analyser

- GHC performs strictness analysis in its Demand Analyser, which does quite a lot of things
- Strictness annotations are most important for the worker/wrapper transformation
- Works by backward analysis:  
  - Finds out, for each expression, which free variables are used strictly 
  - and for each function which arguments are used strictly
- Analysis information is captured in the expression's strictness type:  
  - an environment for strictness on free variables
  - a list of strictness indicators for arguments

# Strictness signatures

- 1-2 minutes
- How do we proceed at let bindings?
- Interesting from the perspective of interprocedural data-flow, to put it in imperative terms
- Do it like DA: Annotate function with their strictness signature
- example `const`:  
  - no free vars
  - uses its first argument strictly, its second argument not at all (thus lazily)
- At the call site we can unleash `const`s strictness type to see that `y` is used strictly and the `error`term is not
- In order to have the strictness signature available at call sites, we have to analyse the RHS before the body

# Call Context matters 1

- 1-2 minute
- This restriction is unfortunate in cases like this
- We'll focus on the free variable `z` here
- The whole expression is strict in `z`
- Our strictness analysis fails to recognise this
- In order to have *some* signature available in the body, our analysis only digests `f` for manifest arity 1 (the number of leading lambdas)
- Because of `seq` it may well be that a call to `f` does not evaluate `z` at all
- Thus the strictness type declaring that `f` is not strict in `z`
- in the body, `f` is only ever called with two arguments
- The number of incoming arguments is valuable information!

# Call Context matters 2

- 1-2 minute
- The solution is to postpone analysis of the RHS until incoming arity is known
- In this case:  
  - Go straight into the body
  - handle the call to `f` by analysing its RHS with the incoming arity 2
  - `z` is detected as used strictly
- Formally, we approximate the *strictness transformer* of the expression in finitely many points
- The strictness transformer of an expression maps incoming arity to the appropriate strictness type
- My first ideas to implement this tried to harness laziness in some clever way to cache strictness types

# Recursion

- 1-2 minute
- Turns out that this completely breaks down when handling recursive bindings
- To compute the strictness type of the `fac`torial function called with one argument, we need the strictness type `fac` called with one argument
- So I rediscovered fixed-point iteration, only that iteration order is no longer dictated by syntactic structure
- To reiterate: Typical analyses within GHC would compute a stable approximation of the RHS, then analyse the body or vice versa
- We descend into the body, then jump to the RHS, then jump back into the body 
- Declare analysis dependencies as a data-flow framework, solve it by worklist agorithm

# Data-flow Framework for Strictness Analysis 1

- 1 minute
- How does this look, conceptually?
- What nodes are present in the graphs we model the problem with?
- We need one node denoting the whole expression
- Other than that, at the least we need to allocate nodes where we have to break recursion
- One node per pair of let bound expression and incoming arity is sufficient
- We initialise the worklist to only contain that root node
- and assign all other nodes a bottom value to start with

# Data-flow Framework for Strictness Analysis 2

1. Now we begin with actually computing a solution, by dequeing the first element of the worklist
2. We invoke the transfer function of the root node and realise that it calls `f` with two arguments in its body
3. `f` recurses into itself in one of its branches  
  - Notably with two arguments, as we analyse the branch under the assumption of one additional argument
  - Our 'callstack' already contains `f`, so we assume we don't recurse again
  - Instead we optimistically assume a $\bot$ result
  - which in this case corresponds to the following strictness type
  - which means its approximation might be unstable
3. Under this assumption we can join together the results from all three case branches  
  - The `f 0` case is lazy in the second argument!
  - Put together, we can update `f`s node with this strictness type
  - Compared to the prior $\bot$ value, the strictness type changed
  - We must enqueue every predecessor: root node and `f` itself
  - These nodes depend on possibly unstable approximations
4. With this potentially unstable approximation we continue with analysing the root node  
5. First argument `x` to `f` is used strictly, `y` lazily  
  - We update the strictness type accordingly and remove `root` from the worklist
6. Now we can dequeue the next item from the worklist, `f`, where we needed to break a dependency clycle
7. So we iterate `f` again
8. Which again depends on itself  
  - This time the strictness type doesn't change
  - so we don't need to enqueue any predecessors
9. The worklist is empty and all nodes are stable, done!  
  - `x` is used strictly, `y` possibly lazily


# Implementation 1

- 1-2 minutes
- That's how we want to look things conceptually
- How do we model this elegantly in Haskell?
- My approach was to hide the iteration strategy behind an abstract `TransferFunction` monad  
  - Parameterized by `k`, the type of nodes, and `v`, the denoting lattice
- There's a impure single primitive in that monad: `dependOn`  
  - Expresses a dependency on the node given as the first parameter
  - Returns just the current value of the node
  - Or `Nothing`, denoting $\bot$, if there is none and the iteration strategy has to break a cycle
  - In absence of any cycles, the current iteration strategy just performs a depth-first traversal
- So that's how we specify a single transfer function

# Implementation 2

- < 1 minute
- A data-flow framework contains a transfer function for every node
- as well as a `ChangeDetector` which captures the notion of efficient tests for increasing values  
  - increasing wrt. the lattice
  - Takes the node, the old value and the new value
- Every analysis problem is to be specified as a value of `DataFlowFramework`
- We can further separate wiring up the dependencies between nodes and specifying the analysis logic

# Implementation 3

- 1-2 minutes
- The final step will be to pass this specification to `runFramework`
- `runFramework` is an interpreter for `DataFlowFramework`s which computes a solution
- It captures a certain iteration strategy
- In our specific case:
- Depth-first traversal to discover all relevant nodes (e.g. the call graph)
- Iterate all nodes in the work list which end up there because of broken cycles, until they are stable
- The `DataFlowFramework` as well as the initial worklist is passed to the `runFramework` function
- The result is a map from nodes to stable denotations

# Applied to Strictness Analysis

- When applied to strictness analysis, we want to denote expressions by their strictness transformer  
  - that was the map from incoming arity to strictness type
- As we saw in the earlier example, we model the strictness transformer point-wise  
  - so there is a strictness type per actual `ExprNode` and `Arity`
- Since `CoreExpr`s/`Id`s themselves are not totally ordered, we need a separate `ExprNode` type for that  
  - The order is important not only for indexing nodes in the graph, but also determines the priorities within the worklist
  - Need to be allocated as part of building the data-flow framework
  - Don't need to leak into the analysis logic
  - Priorities should represent ordering of Occurence Analyser for good analysis performance

# Comparison to hoopl

- Everything that does data-flow analysis in a Haskell should compare to hoopl  
  - For anyone unaware, hoopl stands for 'higher-order optimisation library'
  - is used in GHC's Cmm backend  
- hoopl works on ordinary CFGs  
  - For the better or the worse, our approach is much less restrictive
  - Could allocate a node per subexpression or just the minimal number of nodes to break cycles
  - Edges are implicit in the DSL through calls to `dependOn`
  - Number of nodes more or less transparent to the logic
  - Allocation of more nodes results in higher degrees of caching but also more book-keeping
- designed to handle imperative languages, hence CFG  
  - whereas our approach is explicitly targeted at expression languages
- In the same veign: hoopl is a rather operational model  
  - Inputs flow into the begin of the basic block, through all instructions and out of the basic block
  - Our approach is more of a denotational, giving meaning to a composite expression from its parts in some domain  
- Also, hoopl makes the join-semilattice explicit  
  - which is probably a good thing to do in order to share more analysis logic
- Finally, hoopl is also concerned with a solution for transformations  
  - something we currently ignore
  - messes with allocated nodes and their priorities

# Discussion

- Now for some pros and cons of my approach
- From the plain standpoint of software engineering, separating analysis logic from iteration logic seems like the right thing to do
- Although on the other hand, the coupling currently present in GHC's analysis is not as bad as when whipping up imperative analyses from scratch
- The interaction of different abstraction layers still obscures intent  
  - 'Is this relevant to analysis logic or just part of fixed-point iteration?'
- As we saw earlier with the call context example, expressing analysis logic and iteration logic as a single iterated tree traversal also makes some ideas impossible to pursue
- Also, optimisations to fixed-point iteration and other 'hacks', such as caching of analysis results between iterations, are encoded in the very nature of data-flow analysis  
  - Iterating bindings in a certain order is abstracted in the worklist
- It is unclear how compiler performance will be affected  
  - Needs quite some book-keeping to update graph nodes and check for changes
  - On the other hand, it promises even better caching and better asymptotics
  - We can't really tell until we measure
- Of course, extracting the shared concerns can only really shine if a number of analyses use this model

# Conclusion

- To wrap up
- I pitched what I found to be an interesting idea that came out of my thesis
- The theme was to separate specification of analysis logic from computing the solution of the underlying data-flow problem
- We saw an example for how the solver would compute a stable solution by iterating the data-flow graph
- Apologies if this was all a bit vague, but I have very concrete plans for the future:  
  1. First, recall that for strictness analysis, we denoted expressions by their strictness transformer, which is a monotone function. Currently, we don't model that at all. What is needed for that to happen, is a map data-structure that is indexed by a partial order, so that we can have arbitrary lattices as keys. You can track my work in this repository
  2. If we have the appropriate data-structure, we need to find a proper way to integrate it into the API. We also need ways to express abortion and other frequent analysis hacks. After some polishing, I plan to actually publish it on hackage
  3. Finally, we can testdrive it in different analyses within GHC and measure how it affects compiler performance
- Even if performance turns out to be bad, I think the approach is probably worth to be considered in hobby projects or when prototyping