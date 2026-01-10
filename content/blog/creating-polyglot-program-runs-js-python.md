+++
title = "Creating a Polyglot Program That Runs in Both JS and Python"
date = "2025-12-29"
tags = [
    "javascript",
    "python"
]
+++

Polyglot programs are no new concept. A polyglot is a computer program or script written in a valid form of multiple programming languages or file formats and have been crafted as challenges and curios in hacker culture since at least the early 1990s.

Here's a popular example of a polyglot written in C, PHP, and Bash:

```c
 #define a /*
 #<?php
 echo "\010Hello, world!\n";// 2> /dev/null > /dev/null \ ;
 // 2> /dev/null; x=a;
 $x=5; // 2> /dev/null \ ;
 if (($x))
 // 2> /dev/null; then
 return 0;
 // 2> /dev/null; fi
 #define e ?>
 #define b */
 #include <stdio.h>
 #define main() int main(void)
 #define printf printf(
 #define true )
 #define function
 function main()
 {
 printf "Hello, world!\n"true/* 2> /dev/null | grep -v true*/;
 return 0;
 }
 #define c /*
 main
 #*/
```

But I had yet to come across a polyglot program that runs both in JavaScript and Python so I set out with the idea of creating such a program. The rules were simple:

1.  The code must be syntactically valid in both languages
2.  Each language's executable code must be isolated so the other language ignores it

The type of polyglot I was going for is known as a zipper polyglot. We'll use a 'parasite polyglot' approach where we embed data into comment structures if we don't want it to be run in one language but do want it to be run in another.

At first it felt borderline impossible but after about an hour of experimenting, I finally came up with a program that satisfies the rules of the challenge by making use of multiline comments in both languages.

```js
1; // 1; """
console.log("Hello from JavaScript");

/* """
print("Hello from Python")
"""
"""
# */
```

Despite completing my self assigned challenge, solely using comments to write the polyglot felt a lot like cheating so I did some research and came across a video by Evan Zhou on Youtube. His approach is slightly more intuitive. For the Python subroutine, his solution used a JS comment much like mine however the JavaScript statement utilises a clever trick. By using a JS Labeled Statement, "lambda:" at the start is treated as a label by the JS compiler and is ignored by the python compiler. The eval is then executed like normal and delivers the execution of the JS code in a string.

```py
1 // (lambda: exec("print('Hello from Python')", globals()) or 1)()
lambda: eval("console.log('Hello from JavaScript')")
```

But it gets better. The comment section of Evan's video is fascinating and there are countless talented individuals with a deep understanding of the intricacies in both languages leading to many awesome solutions.

Here is one that I found particularly interesting which capitalises on how bitwise and equality operators bind:

```js
eval(
  ['print("Hello from Python")', 'console.log("Hello from JavaScript")'][
    1 | (0 == 2)
  ]
);
eval(["print(1|0==2)", "console.log(1|0==2)"][1 | (0 == 2)]);
```

Although in hindsight, solving this problem and understanding other solutions seems simple in hindsight, coming up with polyglot programs is a lot harder than you'd think and I would definitely give it a shot in any programming languages of your choice. Writing polyglot programs is definitely an underrated way to become a better software engineer and learn the intricacies of the languages we use on a daily basis.

Thanks for reading :)

### References:

[polyglot-experiments](https://github.com/myl117/polyglot-experiments)

[Polyglot (computing)](<https://en.wikipedia.org/wiki/Polyglot_(computing)>)

[this code runs in both JS and Python.](https://www.youtube.com/watch?v=dbf9e7okjm8)
