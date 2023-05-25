---
layout: post
title: Using variables in vim macros
---

I was recently testing an application that allows users to enter a series of integer keys, which are then stored in a B-tree and persisted to a file. 

It appeared to be working when the keys were inserted in ascending order, but that is not a typical case. To test the typical case, I included a simple helper function which simulated user input of all integers between 1 and n inclusive in random order. 

This revealed to me that my implementation was buggy, as insertion of the same n keys was causing unexpected behavior depending on their insertion order.

Although I was able to determine that there was a problem by generating random sequences and throwing them away, I needed to pin one of these random sequences down and reuse it in order to understand the bug it had exposed.

At first, it seemed that dropping this random sequence into my test function would be tricky - my plan was to first explicitly initialize each index of an array with the same value between 1 and n that my random sequence had had at that location. I would then loop over the array, performing an insertion at each step. Unfortunately, the bug was only occurring when n was large enough to make writing out the initialization of each array index by hand impractical.

I knew that my problem could be solved easily by writing a vim macro, but there was a wrinkle - each line edited by the macro would have to put a numeric value representing the index being initialized, and that value would have to be incremented at each step:

```
inputs[0] = 127;
inputs[1] = 35;
...
inputs[89] = 5;
inputs[90] = 94;
```

To review, macros are a feature that vim provides for automating repetitive text editing operations - by typing `q` and then another key which represents the register in which the macro will be stored, you begin a meta-operation during which vim records each key you press. You stop recording by typing `q` again. If you have 100 lines of text and you want to edit all of them in the same way, you can begin recording a macro, edit the first line while recording, save the macro, and then invoke that macro 99 times to edit all 100 lines. If you saved your macro to register `b`, you invoke the macro 99 times by typing `99@b`.

Armed with that knowledge, we have almost everything we need to solve our problem. But how can introduce a local variable if all we are doing is recording key presses and replaying them? As it turns out, you can initialize vim registers directly using the `:let` command, which makes this a very straightforward task - by writing 0 to the `a` register once and incrementing the value in `a` each time the macro is invoked, we can automatically turn a list of numbers into the expressions we wanted with only a few commands:

```
:let @a=0
```

The above command initializes the variable we will be using in our macro. Once you issue this command, you can type `"ap` and 0 will appear after your cursor.

Now for the macro itself, which we will store in register `b`. We will first present it in its entirety, and then dive into each command separately.

```
qb0iinputs[<Esc>"apa] = <Esc>A;<Esc>:let @a=@a+1<Enter>jq
```

As noted above, `qb` means we have started recording a macro that will be saved to register `b`.

The first key we record is `0`, which will move our cursor to the beginning of the current line. This is a common pattern in vim macros, and ensures that our macro will start its execution at the beginning of the current line, no matter where our cursor was when the macro was invoked. 

Next we enter insert mode with `i` and write the text `inputs[` on our current line, returning to normal mode with `<Esc>`.

Now, in order to obtain the index we want to initialize, we access the value that is currently saved to `a` by typing `"ap`. The lowercase `p` will place the value after our opening square bracket.

Once this has value has been placed, we enter insert mode at the next position after the cursor with `a` and write the closing square bracket, a space, the assignment operator, and another space.

We are now at the number we want to include in our array, and we need to put a semicolon immediately after it. We will jump over it to the end of the current line and enter insert mode with `A`, write the semicolon, and return to normal mode again with `<Esc>`.

Now we update the value saved to `a` with the command `:let @a=@a+1`, hitting `<Enter>` to execute the command.

The final command before we close our macro with `q` is a `j`, which moves us down to the next line. This command is what allows us to edit n lines by invoking our macro n times. The end of the last invocation moves us to the next line, setting us up for the next invocation.

Below is a demonstration of our macro in action:

![Demonstration of Vim Macro](/assets/images/2023-05-25-macro.gif)
