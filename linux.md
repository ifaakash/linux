Question 2: What is -iE used for in grep?
The -iE flag is actually a combination of two separate flags squished together (-i and -E). When combined, they allow you to do a case-insensitive search using extended regular expressions.

-i (ignore case): Tells grep to treat uppercase and lowercase letters as the same. Searching for "hello" will match "Hello", "HELLO", and "HeLlO".

-E (Extended Regular Expressions): Tells grep to understand advanced regex characters like | (OR), + (one or more), ? (zero or one), and () (grouping) without you having to put a backslash \ in front of them.

Example:
grep -iE "error|warning" file.txt will look for both "error" and "warning" in file.txt, ignoring whether they are capitalized or not.



