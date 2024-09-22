java c
CSSE4630 Assignment One: Rust-Inspired Analyses
2024 version 1.0
1 Introduction
This assignment is focused on several kinds of analysis inspired by the Rust programming language. Rust is a strongly typed language that uses a sophisticated type system to prevent many kinds of programming errors, including memory safety errors. It provides a powerful ownership system that allows the programmer to control the lifetimes of objects and ensure that they are not used after they have been deallocated. This assignment will focus on two kinds of analysis that are used in Rust.
The goal of the assignment is to give you experience with implementing static analyses, and at the same time develop a better understanding of the techniques that Rust uses to guarantee type safety and memory safety.
You will first implement type checking for TIP, so that we can distinguish pointers from inte-gers.
Then you will design a new analysis for TIP that corresponds to the ownership checker of Rust. Optionally, you can extend this analysis to be interprocedural.
This assignment is worth 20% of your final mark for this course. It is due on Friday 3-Oct-2024 before 4:00pm. You can submit your solution multiple times to Gradescope before this deadline, to check that various aspects are working correctly.
2 School Policy on Student Misconduct
This assignment is to be completed individually. You are required to read and understand the School Statement on Misconduct, available on the School’s website at: https://eecs.uq.edu.au/current-students/guidelines-and-policies-students/student-conduct
You may appropriately use AI as a programming aid while completing these assessment tasks, but you must clearly reference any AI-generated code (following the usual UQ AI guidelines: https://guides.library.uq.edu.au/referencing/generative-ai-tools-assignments), just as you should acknowledge any code fragments adapted from other internet sources. Include these acknowledgements as comments in your Scala source code. A failure to reference genera-tive AI or machine translation (MT) use may constitute student misconduct under the Student Code of Conduct.
3 Implementing Type Checking [40%]
The TIP system does not fully implement static type checking yet. To see this, run the following command and read the exception it generates:
tip -types examples/ptr7.tip
Implement the missing cases in the visit method in TypeAnalysis.scala. See Section 3.2 (Type Constraints) of the textbook for the rules you should implement.
Hints:
1. The solver is already created for you, so you can just call its unify method.
2. The address-of constructor in the textbook (X) corresponds to PointerRef( ) in the Scala code.
3. To allocate a fresh variable (called α in the textbook) use the FreshVariable() function.
4. It is important that each occurrence of the same TIP variable must be mapped to the same type variable. You could implement this yourself, by defining a mapping from identifiers to type variables, but this gets quite tricky, because you really need to take into account the scope of each variable. For example, two different functions might both use a variable with the same name, but we should really treat them as separate variables, possibly with different types! Fortunately, TIP makes our task easier by providing the declData(id) method, which maps each variable to its correct declaration. So rather than implementing your own mapping, you can use this method to get a common handle (the declaration object) for all occurrences of a variable in that scope, then map that declaration object to a unique type variable for all occurrences of the identifier.
3.1 Checking your results
Use your type checking algorithm to analyse the tests/main1.tip program. You can download several example test files like this from Blackboard. This example checks that you are unifying type variables correctly, and also that your type checking algorithm knows that the main method is special, because it always has integer inputs and outputs.
You should see standard output being printed that includes the following results:
[info] Inferred types:
[main(a[2 : 6], b[2 : 9])... : 1 : 1 K = (int,int) -> int[a[2 : 6] K = int[b[2 : 9] K = int[c[3 : 9] K = int
You can submit your solution (multiple times) to Gradescope to get feedback from several tests like this. After the deadline, your final typechecker will be tested using additional test programs and marked for correctness and elegance.
4 Implementing a Simple Ownership Algorithm [40%]
Next we want to implement a simple ‘ownership’ algorithm for heap variables, to track a unique owner for each heap-allocated memory location. When this owner goes out of scope, we can deallocate that heap variable. Variables that store non-heap values, such as integers, we can ignore.
For our analysis, we will map each variable to an ownership status, which will be one of the following three options:
• HeapOwner: the variable points to a heap-allocated value and is the unique owner of that value.
• HeapNotOwner: the variable points to a heap-allocated value but is NOT the owner of that value. It is illegal to read the heap value via this variable.
• NonHeap: the variable does not point to the heap. For example the variable contains integer values.
Our analysis will be a forward analysis. Initially, we will start with a simple intraprocedural analysis and will restrict our input language so that pointers only point directly to heap allocated values, not to local variables, and not to other pointers in the heap. In other words, you can assume that:
1. the address-of (x) operator will not appear in our input programs.
2. the value stored inside a heap allocated memory location will always be an integer.
The rules for ownership analysis of a CFG node v are as follows:
[vars v1, v2, . . .] = JOIN(v)[v1 → ⊥, v2 → ⊥, . . .]                               [x = y] = JOIN(v)[x → eval(y, JOIN(v)), y 7→ yo], where
                                              yo = let o = eval(y, JOIN(v)) in (if (o = HeapOwner) o else HeapNotOwner)                               [x = E] = JOIN(v)[x → eval(y, JOIN(v))], if E is not an identifier
For the eval function, alloc e should evaluate to HeapOwner obviously. All non-pointer values such as integers will evaluate to NonHeap. So all binary operators and constants will evaluat代 写CSSE4630 Assignment One: Rust-Inspired AnalysesR
代做程序编程语言e to NonHeap. The dereference operator x must check that x is a HeapOwner, and if so, return NonHeap (because the dereferenced result should be an integer), otherwise give either an error or a warning:
• Reason.OwnershipError if x is definitely not a HeapOwner.
       Error message should be: illegal dereference when not owner: x.
• Reason.OwnershipWarning if x might not be a HeapOwner.
       Warning message should be: dereference when may not be owner: x.
Hints:
1. A good strategy when developing a new feature in an unfamiliar system is to start by copying and modifying one of the existing features. In this case, you might start off by copying SimpleSignAnalysis.scala and renaming everything to OwnershipAnalysis. For the simplest approach, I suggest starting from SimpleSignAnalysis, because it also uses a nice small flat lattice, is a forward analysis, and is written in an easy-to-understand self-contained style. (rather than using inheritance to reuse worklist solvers, etc. which is a better approach in the long term, but requires more knowledge of all the various traits and classes available in TIP). However, if you plan to do the advanced interprocedural analysis (Section 5) part of this assignment, you might want to start by copying the more complex SignAnalysis.scala, because it illustrates how to support the interprocedural analysis that you will need for that part of the assignment.
2. To integrate your new analysis into the tip command line, you will need to define a new command line flag (-ownership) and implement this in at least Tip.scala and the master controller superclass FlowSensitiveAnalysis. Again, follow a similar example: SimpleSignAnalysis or SignAnalysis.
4.1 Reporting Ownership Errors and Warnings
Generate an error whenever the program tries to reference a heap-allocated value that is defi-nitely not owned by the current variable. Generate a warning whenever a variable is derefer-enced that may not be a HeapOwner (e.g. Top). For the automated testing and marking of your solution you must report these errors and warnings in exactly the way described in this section.
You must use the supplied MessageHandler class to record and print all your error and warning message. Download this Scala class from Blackboard and put it into the src/tip/util folder of your TIP project. Create a new instance of the MessageHandler class as a field of your analysis class.
Then in your analysis code call the message(Level, Location, Msg) method of your message handler whenever you want to record an error or warning. The Level parameter should be OwnershipError or OwnershipWarning, or None if there is no error at that location. The Location parameter is the location in the program where the error or warning occurs (you can get this from the loc field of the statement or expression you are analysing), and the Msg parameter should be a string that describes the error or warning.
You should also clear any previous error messages for the same location before adding a new one, so that you do not get multiple error messages for the same location. For example, you should have code like this in your analysis for each of your ownership error or warning checks:
if (some condition ) {
// no error here, so clear any previous error message for this location
msgs.message(msgs.Reason.None, loc)
. . .
} else {
val reason = msgs.Reason.OwnershipError
msgs.message(reason, loc, s"illegal dereference when not owner: $expr")
. . .
}
5 Advanced Ownership - Interprocedural [20%]
This is a more difficult part of the assignment that you should attempt only after completing all previous parts. The goal is to improve your ownership checker so that it can also analyse TIP programs that use function calls.
Follow the general interprocedural rules given in Section 8.1 of the textbook. You should im-plement at least the iwlr version of your interprocedural analysis, so that when you run:
• tip -ownership tests/ownership1.tip it runs the intraprocedural version of your anal-ysis.
• tip -ownership iwlr tests/ownership1.tip it runs the interprocedural version of your analysis.
In practice, you should include the following options in your TIP command line to ensure that programs are normalised into a standard form.
• -normalizereturns normalize return statements
• -normalizecalls normalize function calls
• -normalizepointers normalize pointer usages
We are not concerned with the full Rust ownership system, so you can ignore the borrow checker and the lifetime annotations. In other words, you can still assume that the address-of (x) operator will not appear in our input programs and that heap locations will only store integers.
Hints:
1. If you want to implement the interprocedural analysis, I recommend you use the SignAnalysis class as a starting point, because it already has the necessary infrastructure for interpro-cedural analysis, such as using an interprocedural CFG.
2. One of the challenges with interprocedural analysis is finding the connections between caller and callee functions. Fortunately, the InterproceduralProgramCfg pre-calculates these connections for you. A CfgFunEntryNode node has a callers field that gives you the set of all functions that call this function and a CfgCallNode has a callees field that gives the set of possible functions being called. Also, a CfgAfterCallNode node has a calleeExits field that gives you all the function exit nodes that might be returning to that after-call node. So you can use these fields to find the connections between caller and callee functions.
6 Submission
Submit your assignment via Gradescope, as a single zip file containing your whole tip directory (with subdirectories called src, tests, out etc.). The src subdirectory will include your mod-ified Scala source files that implement your pointer analysis algorithms. The out subdirectory should include your output files for each *.tip test file in the tests directory.
You can submit multiple times before the deadline to check that your submission has the correct structure and that the provided tests work correctly. It is your last submission before the deadline that will count.
After the deadline, your final submission will be tested using additional test programs and marked for correctness and elegance.





         
加QQ：99515681  WX：codinghelp
