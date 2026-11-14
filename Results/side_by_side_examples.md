# Base vs feedback adapter: side-by-side examples

Feedback from base Qwen3.5-9B and the feedback QLoRA adapter on rows from the 389-row test set, whose questions neither model was trained on. Both models got the same prompt: question, reference answer, student answer, and each key point marked MET / NOT MET. Verdicts are from the Qwen3.5-27B judge in `feedback_teacher_qwen3.5-27B.ipynb`, saved in `Datasets/teacher_feedback/student_test_judgements.jsonl`.

Feedback is quoted verbatim. Markdown characters such as `**` and backticks are escaped, so they show exactly as the model wrote them.

| | Base | Adapter |
|---|---|---|
| Passes all four judge checks | 79.7% | 89.7% |
| Faithful to the student's answer | 86.9% | 95.9% |
| Factually correct | 88.2% | 95.4% |
| Rows where only this model passed | 15 | 54 |

Examples 1–4 are rows the base model failed and the adapter passed. Examples 5–6 passed for both and show the difference in style. Examples 7–8 are rows where the adapter failed and the base model passed.

## 1. Base credits the student with reasoning they never wrote
Row 3050 · partial

**Question:** Q1. State TRUE or FALSE and justify. No correct justification, no marks.
e. If a parent process forks a new child process, the exec system call cannot be used in the parent process until the child process terminates.

**Reference answer:** False, parent and child processes are two independent processes. Parent can perform any operation that it likes including an exec system call.

**Student answer:** \*\*FALSE\*\*

**Key points:**
- MET: Answer states the statement is False
- NOT MET: Answer mentions that parent and child are independent processes
- NOT MET: Answer mentions that the parent can call exec even before the child terminates

**Base Qwen3.5-9B** (98 words). Judge: failed: Feedback falsely claims the student noted process independence, which was not in their answer.

> You correctly identified the statement as \*\*FALSE\*\*, which is a great start! However, your justification was incomplete. While you noted that parent and child processes are independent, you didn't explicitly state that the parent retains the freedom to execute the \`exec\` system call immediately, regardless of whether the child has terminated. To fully justify your answer, make sure to connect the independence of the processes directly to the parent's ability to perform operations like \`exec\` at any time. Try adding a sentence like, "Since the parent can continue executing independently, it can call \`exec\` before the child finishes."

**Feedback adapter** (86 words). Judge: passed

> This question tests whether you understand that a parent process remains independent of its child after forking. You correctly identified the statement as false, but your answer lacked the necessary justification. Specifically, you did not explain that parent and child are separate entities, nor did you clarify that the parent can execute an exec system call at any time, regardless of the child's status. To improve, explicitly state that the two processes operate independently and that the parent retains full control over its own execution flow.

*What to notice:* The student wrote only "FALSE". The base model says "you noted that parent and child processes are independent", which isn't in the answer. The adapter says the justification is missing and names both NOT MET points.

## 2. Base confirms a wrong answer
Row 890 · none met

**Question:** What does the address (&) operator return?

**Reference answer:** The memory address of its operand.

**Student answer:** the reference to a variable

**Key points:**
- NOT MET: Answer mentions returns memory address of operand

**Base Qwen3.5-9B** (67 words). Judge: failed: Feedback incorrectly validates 'reference' as correct terminology for the & operator in C/C++.

> You are very close! The operator does indeed return a reference to a variable, but in C/C++, the specific term used is the \*\*memory address\*\*. While a reference is a conceptual way to think about it, the \`&\` operator literally yields the numerical address where the operand is stored in memory. Try adjusting your answer to explicitly state "memory address" to match the precise definition. Great start!

**Feedback adapter** (55 words). Judge: passed

> The address-of operator specifically returns the memory address of its operand, not a general reference to a variable. While your answer captures the idea of pointing to a variable, it lacks the precise terminology required for this context. To improve, explicitly state that the operator provides the actual memory location where the operand is stored.

*What to notice:* The base model tells the student the operator "does indeed return a reference" and calls the answer "very close". The adapter says plainly that it returns the memory address, not a reference.

## 3. Base contradicts itself
Row 1850 · none met

**Question:** What is the main difference between a string of characters that is read into a variable of type string versus a variable of type char[]?

**Reference answer:** The char[] will automatically add a null \\0 character at the end of the string.

**Student answer:** the one that is type string has an extra string member. The /0 at the end.

**Key points:**
- NOT MET: Answer mentions char[] adds null terminator

**Base Qwen3.5-9B** (80 words). Judge: failed: Feedback claims student correctly noted string lacks null terminator, but student claimed string has it.

> You correctly identified that the \`string\` type does not include the null terminator, which is a great observation! However, your answer attributed the presence of the null terminator (\`\\0\`) to the \`string\` type. In reality, it is the \`char[]\` array that automatically adds this null character at the end. To improve your response, simply swap the types: the \`char[]\` is the one that includes the extra null terminator, while the \`string\` type manages its own internal memory without needing it.

**Feedback adapter** (63 words). Judge: passed

> Your answer correctly identifies that a null character is involved, but it incorrectly attributes this to the string type. In reality, the char[] variable is the one that automatically adds the null terminator at the end of the string. The string type manages its own internal storage without requiring this explicit addition. What specific responsibility does the char[] type have regarding memory termination?

*What to notice:* The base model first praises the student for saying `string` has no null terminator, then says (correctly) that the student attributed it to `string`. The adapter gives one consistent correction.

## 4. Base adds a factual error and runs long
Row 372 · none met

**Question:** What is the role of a header-file?

**Reference answer:** To store a class interface, including data members and member function prototypes.

**Student answer:** \#include "hat.h"  allows the use of the hat container, so called from the act of pulling names from a hat.  The header file for hat includes functions for storage and retrieval of it's contents, mainly put(), peek(), and pull().

**Key points:**
- NOT MET: Answer mentions storing class interface
- NOT MET: Answer mentions data members and member function prototypes

**Base Qwen3.5-9B** (132 words). Judge: failed: Feedback incorrectly suggests header files contain function definitions instead of prototypes.

> Your explanation of the \`hat.h\` file is creative and clearly illustrates how a header file enables code reuse by exposing specific functions like \`put()\`, \`peek()\`, and \`pull()\`. However, to fully answer the general question, you need to broaden your scope beyond a single example.
>
> A header file's primary role is to define the \*\*interface\*\* for a class or module, not just to list function calls. This means you should explicitly mention that it stores the \*\*data members\*\* (variables) and the \*\*prototypes\*\* (signatures) of member functions. While your example shows the functions, it doesn't explicitly state that the header file declares the underlying data structure or the function signatures themselves. Try to generalize your answer to include both the data and the function definitions to capture the complete role of a header file.

**Feedback adapter** (73 words). Judge: passed

> The question asks for the general purpose of a header file, while your example describes a specific use case for a container class. You correctly identified that header files contain function prototypes like put() and peek(), but you did not explicitly state that they store the class interface or list its data members. A complete answer would define the role as storing the class interface, including both data members and member function prototypes.

*What to notice:* The base model ends by telling the student to include "the function definitions"; header files hold prototypes, not definitions. It is also 132 words in two paragraphs with bold text. The adapter gives the reference definition in 73 words.

## 5. Both pass: style on a fully correct answer
Row 2172 · all met

**Question:** Q1. State TRUE or FALSE and justify. No correct justification, no marks.
e. If a parent process forks a new child process, the exec system call cannot be used in the parent process until the child process terminates.

**Reference answer:** False, parent and child processes are two independent processes. Parent can perform any operation that it likes including an exec system call.

**Student answer:** \*\*False\*\*, after fork is executed the two processes are completely independent of each other and so the parent can call exec system call whenever it wants.

**Key points:**
- MET: Answer states the statement is False
- MET: Answer mentions that parent and child are independent processes
- MET: Answer mentions that the parent can call exec even before the child terminates

**Base Qwen3.5-9B** (72 words). Judge: passed

> Excellent work! You correctly identified the statement as \*\*False\*\* and provided a precise justification.
>
> Your explanation that the parent and child processes are "completely independent" is spot on. This independence is exactly why the parent process retains full control and can execute the \`exec\` system call immediately, regardless of whether the child process has finished running.
>
> You have successfully covered all the necessary aspects of the question. Keep up the great job!

**Feedback adapter** (62 words). Judge: passed

> This question tests your understanding of process independence after a fork system call. You correctly identified the statement as false and explained that the parent and child become completely independent, allowing the parent to execute an exec call at any time. Your answer works because it clearly establishes that the termination of the child is not a prerequisite for the parent's execution.

*What to notice:* Both are correct. The base model uses bold text, three paragraphs and generic praise ("Excellent work!", "Keep up the great job!"). The adapter is one plain paragraph that says what the question tests and why the answer works, the style it was trained on.

## 6. Both pass: style on a one-word answer
Row 88 · all met

**Question:** Where do C++ programs begin to execute?

**Reference answer:** At the main function.

**Student answer:** main method

**Key points:**
- MET: Answer mentions main function

**Base Qwen3.5-9B** (57 words). Judge: passed

> Great job! You correctly identified that C++ programs start execution in the \`main\` function. Just a small note: in C++, we refer to it as the \`main\` \*\*function\*\*, whereas "method" is terminology typically used in Java or C\#. Using the precise term "function" will ensure your answers are technically accurate for C++. Keep up the excellent work!

**Feedback adapter** (47 words). Judge: passed

> Your answer correctly identifies that C++ programs begin execution at the main function. While you used the term "method," which is more common in Java, the core concept here is accurate for C++. This response works because it directly names the specific function where program execution starts.

*What to notice:* Both point out that "method" is Java terminology. The base model uses backticks and bold and closes with "Keep up the excellent work!". The adapter stays plain and explains why the answer works.

# Where the adapter is worse

## 7. Adapter contradicts the marker
Row 362 · partial

**Question:** What is the role of a header-file?

**Reference answer:** To store a class interface, including data members and member function prototypes.

**Student answer:** Header files separate files in which classes are defined and allow the compiler to recognize classes when used elsewhere.

**Key points:**
- MET: Answer mentions storing class interface
- NOT MET: Answer mentions data members and member function prototypes

**Base Qwen3.5-9B** (72 words). Judge: passed

> You correctly identified that header files separate class definitions and enable the compiler to recognize them in other files. However, to fully capture the role of a header file, you should specify that it stores the \*\*interface\*\*, which explicitly includes \*\*data members\*\* and \*\*member function prototypes\*\*. While the definition implies the interface, explicitly listing these components ensures your answer is complete and precise. Try refining your response to highlight these specific elements!

**Feedback adapter** (57 words). Judge: failed: Feedback contradicts MET key point by claiming student missed 'class interface'.

> Your answer correctly identifies that header files help the compiler recognize classes across different files. However, you did not specify that these files store the class interface, which includes both data members and member function prototypes. A complete answer would state that a header file stores a class interface, including its data members and member function prototypes.

*What to notice:* The marker marked "storing class interface" as MET. The adapter says "you did not specify that these files store the class interface", contradicting it. The base model gets this row right.

## 8. Adapter confirms a wrong claim
Row 480 · none met

**Question:** What is the main advantage associated with function arguments that are passed by reference?

**Reference answer:** It avoids making copies of large data structures when calling functions.

**Student answer:** You do not alter the original value of the variable that was passed.

**Key points:**
- NOT MET: Answer mentions avoiding copies of large data structures

**Base Qwen3.5-9B** (69 words). Judge: passed

> Your answer highlights an important distinction: passing by reference does not alter the original variable if the function doesn't explicitly modify it. However, the main advantage you're looking for relates to memory efficiency. When arguments are passed by reference, the program avoids creating duplicate copies of large data structures, which saves significant memory and improves performance. Try focusing your response on how this approach prevents unnecessary duplication of data.

**Feedback adapter** (59 words). Judge: failed: Feedback validates student's incorrect claim about pass-by-reference value preservation.

> The main advantage of passing arguments by reference is that it avoids creating copies of large data structures when calling functions. Your answer correctly notes that the original variable's value remains unchanged, which describes a behavior of pass-by-value or const references rather than the primary efficiency benefit of pass-by-reference. How does avoiding memory duplication improve performance for large datasets?

*What to notice:* The student said pass-by-reference doesn't alter the original value, which is wrong. The adapter opens with the right answer but then says the answer "correctly notes" that the value is unchanged. The base model avoids that.
