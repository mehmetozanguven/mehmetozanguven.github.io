---
layout: post
title: "Regex in one blog"
date: 2020-10-05 19:45:31 +0530
categories: "java"
author: "mehmetozanguven"
newUrl: "https://mehmetozanguven.com/java/regex-in-one-blog/"
---

In this post, we are going to learn Regex expression in one blog

Let's start with defining what is Regex or Regex expression.

# What is Regex?

Regex or Regex expression is a special text string for describing a search pattern. Regex can be used to find specific word in the text file or we can use to determine whether text is matched with specific pattern or not.

And also the regex is applied on the text from left to right.

Before diving into regex examples, we should know the specific characters/symbols that has special meaning in regex.

## Regex Symbols

- `.` => Any character except new line (please be careful, **not a word, character**)
- `\d` => search for any digit `[0-9]`
- `\D` => search for anything not a digit `[^0-9]`
- `\w` => search for a word character `[a-zA-Z0-9-]` (Note: `-` is also a word character)
- `\W` => search for a not word character
- `\s` => search for any whitespace (includes space, tab, newline)
- `\S` => search for anything not whitespace
- `\b` => search for a word boundary. Matches positions where one side is a word character and other side is not a word character.
- `\B` => search for not a word boundary
- `[]` => matches a characters in brackets
- `[^]` => negates the condition in the brackets
- `^` => Beginning of a string (`^a` => string starts with `a` , be careful this command takes care of whitespace for example string starting with whitespace and a character like ` a` does not match)
- `$` => end of a string (`a$` => string ends with `a`, takes care of whitespace like)
- `*` => 0 or more
- `+` => 1 or more
- `?` => 0 or 1
- `{3}` => exact number
- `{3,5}` => range of number
- `(a | b | c)` => matches the group, (match with a, b or c)

Looking at the table is not enough to understand regex. We should do some basic examples before implementing in any programming language. That's why first of all, I am going to use sublime text, then search via regex expression.

## Examples in Sublime Text

It would be a good to start with a simple one (at least for me :D ), **word boundary**

### Word Boundary

Just open the sublime text and write two sentences line by line.

```wiki
Java is a programming language.
JavaIsAProgrammingLanguage.
```

And open the search tool (`ctrl + f`), and please enable **Regular Expression and Case Sensitive** options.

Let's write a few examples:

- `\bJava` => will match string Java where before first character `J` must not followed by any word character. In that case you are going see that both first Java strings will be highlighted.

> If you put any space before character `J`, it will also match. Because **space is not a word character**

- `\bJava\b` => will match string Java where before first character `J` and after last character `a` must not followed by any word character. In that case, only the first line will match. Because in the second line `JavaIsAProgrammingLanguage` there is a word character after the last character `a`.
- `\BJava` => will match string Java where before first character `J` must be followed by any word character. In that case, there will be no matched. If you want to see, just update the first line like this:

```wiki
aJava is a programming language.
```

- `Java\B` => will only highlight the second Java character.

### Special Characters `-, *, +, ?`

- If we put `-` (dash) outside of the brackets `[]`, regex will try to match just the symbol itself.
- If we want to match with internal, then we use `-` in the brackets `[]` , like:
  - `[a-z]` => matches only a single character if it is `a` or `b` or `c` ... or `z`

Here is the examples:

- `[a-z]` => matches the lower character between a-z. When you run this in the search toolbox, here is the result screen:

<img src="/assets/java/regex/special_character_1.png" alt="Special Character -" title="Special Character Example" />

**Don't confuse yourself, regex matching the individual character not all sentences. Look at carefully**.

**You may ask how the match with the string not a character**. In this situation, you should use `*, + ,?`

- `[a-z]*` => matches the 0 or more lower characters between a-z. Now difference between the above is clear, look at the screenshot:

<img src="/assets/java/regex/special_character_2.png" alt="Special Character -" title="Special Character Example 2" />

Regex matching with `ava` because character `a` is followed by `v` and v followed by `a` (not: **space is not included in the list**)

- `[a-z]{4}` => matches exactly 4 lower characters between a-z.

<img src="/assets/java/regex/special_character_3.png" alt="Special Character -" title="Special Character Example 3" />

- `j{3}` => matched with three j's.
- `[jav]{3}` => three characters, each of which can be a `j,a` or `v`. Matching examples: `jav`, `jjjav` (only `jjj` matches), `jjaaav`(two matches `jja` `aav`)

### Begin and End `^, $`

If `^` used in `[^]` bracket, it just negates the condition.

- `[a-z]` => any lower character between a-z
- `[^a-z]` => any character, number .. but not lower character between a-z

Examples (these are not related to the our Java sentences):

- `.*P$` => matches 0 or more any new character except new line and ends with P.
- ^J[0-9]+a$ => starts with J, then followed by one or more numbers, then and with a. `J1a, J213a`

## Difference between `ja` and `[ja]`

- If you write `ja` in the search box: it will just the search the string `ja`

<img src="/assets/java/regex/difference_example_1.png" alt="Difference Example" title="Difference Example 1" />

- If you write `[ja]` in the search box: it will match the character `j` or `a`

<img src="/assets/java/regex/difference_example_2.png" alt="Difference Example 2" title="Difference Example 2" />

-

## Random Examples

Here is the list of the some example, our aim is to find a regex pattern for the texts (matched or skipped)

- Example 1:

```wiki
match: abc123xyz
match: define "123"
match: var g = 123;
```

Solution: `[\w\d\s\W]+` => matches _one or more word character_, _number_, _whitespace_ and _not a word character_.

- Example 2:

```wiki
match 	cat.
match 	896.
match 	?=+.
skip 	abc1
```

Looks hard, but one point of all matching texts is that they are ending with `.` dot. Therefore we can match anything that ends with `.` dot. Solution: `...\.` => (`\` => escaped character)

- Example 3:

```wiki
match 	can
match 	man
match 	fan
skip 	dan
skip 	ran
skip 	pan
```

Solution: `[cmf]an` => matches single character if it is `c`, or `m` or `f` and followed by `an` strings.

- Example 4:

```wiki
match 	hog
match 	dog
skip 	bog
```

Solution: You may use the same approach from the previous one => `[hd]og`. But there is elegant way and more flexible `[^b]og`

For final section, let's do some real examples:

### Match with specific filenames

Assume that you are searching image files in the linux terminal, and you also want to group the image name and image extension. For example if the filename is the **sample.jpg**, you should match the filename and the groups **sample** and **jpg**.

Last thing, you want to match also png and gif files.

You should match the files: `img09.img, favicon.gif, updated_img12.png`

> To create a group use `()` parentheses

Solution: `(.*).(jpg|png|gif)$` => match one or more any characters except new line and this matching characters must be followed by `.(dot)`. After `.(dot)` must be followed by strings `jpg` or `png` or `gif`.

### Matching with HTML tags

You are trying capturing HTML tags let's say in the file.

You should match the following tags:

```wiki
<a>This is a link</a>			 			capture a
<a href='https://regexone.com'>Link</a>		capture a
<div class='test_style'>Test</div>			capture div
<div>Hello <span>world</span></div>			capture div
```

Solution: `<(\w+)`

You can find more and more examples in this [link](https://regexone.com/)

Last but not least, wait for the next post…
