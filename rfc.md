PHP RFC: Pipe assignment operator

Version: 0.0.1
Date: 2025-06-23
Author: Vadim Dvorovenko, vadimon@mail.ru
Status: Draft
Implementation: -

# Introduction

Pipe operator (or simular) can be used for assignment to reduce congnitive complexity.

```
// Current syntax
$result = "Hello World" |> strlen(...);

// New Syntax
"Hello World" |> strlen(...) |>= $result;
// Or
"Hello World" |> strlen(...) |> $result;
```

# The problem

Pipe operator introduced in RFC (https://wiki.php.net/rfc/pipe-operator-v3) may lead to a complication in the perception of the flow due to mixing of left-to-right and right-to-left action directions. To avoid this, it is proposed to use the pipe operator (or a new similar one) for left-to-right assigning.

Long-standing traditions of procedural code uses two main reading directions: top-to-bottom for procedural flow and right-to-left for function call and assignment flow in single line.
```
// Top to bottom example
$temp = "Hello World";          //   -1-|
                                //      |
$temp = htmlentities($temp);    //   <--|    -2-|
                                //              |
$temp = str_split($temp);       //           <--|   -3-|
                                //                     | 
$temp = array_map(strtoupper(...), $temp); //       <--|    -4-|
                                //                             |
$result = $temp;                //                          <--|

// Left-to-right example.
// 
//   |--4---|     |----------3---------|     |-2-|        |-1-|  
//   V      |     V                    |     V   |        V   |
$result := array_map(strtoupper(...), str_split(htmlentities("Hello World")));
```

The programmer's brain is tuned to perceive exactly these directions. When we try to reverse this directions, we need to to introduce additional rules, such as idents, to make hints to our brain that we should use bottom-to-top direction instead of top-to-bottom.

```
$result = 
//   ^
//   |
    array_map(
    //  ^
    //  |
        strtoupper(...),
        str_split(
        //   ^   
        //   |   
            htmlentities(
            //   ^   
            //   |   
                "Hello World"
            )
    )
);
```

Pipline operator and fluent pattern were invented to use top-to-bottom/left-to-right direction in function calls. Without assignments this gives perfect top-to-bottom and left-to-right code reading direction.

```
// Pipeline
"Hello World"
    |> htmlentities(...)
    |> str_split(...)
    |> fn($x) => array_map(strtoupper(...), $x)

// Example with some fluent helper 
 helper("Hello World")
    .htmlentities()
    .split()
    .map(fn($x) => strtoupper($x));
```

But things are getting worse, when we add asignment operator. We need to use top-to-bottom/left-to-righ reading direction for most parts of instruction, but bottom-to-top/right-to-left for last assignment action.
```
// Single line case
//   |-----------------------------------------------------------4-------------------------------------------------|
//   |            |-----1----|        |-----2-----|     |------3---|                                               |
//   V            |          V        |           V     |          V                                               |
$result := "Hello World" |> htmlentities(...) |> str_split(...) |> fn($x) => array_map(strtoupper(...), $x)  // ---|

// Multiline case
$result :=                          // <---------------------------|
                                    //                             |
    "Hello World"                   //   -1-|                      |
                                    //      |                      |
        |> htmlentities(...)        //   <--|   -2-|               |
                                    //             |               |
        |> str_split(...)           //          <--|   -3-|        |
                                    //                    |        |
        |> fn($x) => array_map(strtoupper(...), $x) // <---     -4-|
```

In the case of multiple lines, the situation becomes more complicated for long functions and pipelines, when everything does not fit to one screen and to read one instruction it is necessary to scroll the screen up and down.

The most natural and convenient reading direction for speakers of all LRT languages ​​(including but not limited to all languages ​​derived from Latin, such as English) is from left-to-right and top-to-bottom. With pipeline operator and pipeline assignment operator it would be posible to write code, that is read and perceived exactly this way, thereby reducing cognitive load.

# Proposal

This RFC introduces of ability to use new pipe assignment operator for assignment to variable.

```
mixed |>= variable
or
mixed |> variable
```

wich is equialent to 
```
$variable = mixed
```

The current bahavior of pipe operator when used with variable is wery poorly described in RFC (but covered with test - https://github.com/php/php-src/blob/master/Zend/tests/pipe_operator/mixed_callable_call.phpt#L73). While 8.5 is not released, we can make exclusion for piping to variable to use it only for assignment. When using for piping to callble, stored in variable, it could be wrapped with arrow function. In this way we even not need to introduce new operator, and can reuse pipe operator for assignment.
If we prefer to use |> as assignment operator, we should change implementation before 8.5 release. In this case if want call closure from variable, it should be wrapped with oter closure.

```
$times29 = function(int $x): int {
    return $x * 29;
};

1 |> $times29 |> $result; // Same as $result = $times29 = 1;
1 |> fn($x) => $times29($x) |> $result; // Same as $result = 29;
```

# In other languages

There have long been known contradictions between mathematicians and programmers regarding the use of the equal sign for the assignment operator and real in the meanings of the expression `a = a + 1`. For example, in Pascal, to resolve this contradiction, `:=` is used for assignment. But only very few languages ​​went further and introduced the left-to-right assignment operator.

* COBOL has has left-to-right assignment operator `MOVE x TO y` (https://www.ibm.com/docs/en/debug-for-zos/15.0.x?topic=commands-move-command-cobol#rcmdmov)
* Casio BASIC has left-to-right assignment operator → (see in https://en.wikipedia.org/wiki/Casio_BASIC#Examples)
* Linux command line has redirection operator > that can be treated as file content assignment (`ls -l | grep ".txt" | wc -l > result`)
* C++ iostream uses >> operator for saving value from stdin to variable


# Further scope

If chosing |>= as operator, we can also create family of pipeline assignment operators.

| New operator  | Equialent   |
|---------------|-------------|
| `$a |>= $b`   | `$b = $a`   |
| `$a |>.= $b`  | `$b .= $a`  |
| `$a |>+= $b`  | `$b += $a`  |
| `$a |>-= $b`  | `$b -= $a`  |
| `$a |>??= $b` | `$b ??= $a` |
