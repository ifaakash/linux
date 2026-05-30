Here are the answers to your command-line and Git questions.

### Question 2: What is `-iE` used for in `grep`?

The `-iE` flag is actually a combination of two separate flags squished together (`-i` and `-E`). When combined, they allow you to do a **case-insensitive search using extended regular expressions**.

* **`-i` (ignore case):** Tells `grep` to treat uppercase and lowercase letters as the same. Searching for "hello" will match "Hello", "HELLO", and "HeLlO".
* **`-E` (Extended Regular Expressions):** Tells `grep` to understand advanced regex characters like `|` (OR), `+` (one or more), `?` (zero or one), and `()` (grouping) without you having to put a backslash `\` in front of them.

**Example:**
`grep -iE "error|warning" file.txt` will look for both "error" and "warning" in `file.txt`, ignoring whether they are capitalized or not.

---

### Question 3: How to use `find` and the `-R` flag?

**Part 1: Finding a file in any folder**
To find a file when you don't know exactly where it is, use the `find` command followed by the starting directory and the `-name` flag.

```bash
find . -name "my_file.txt"
```

* `.` tells it to start searching in your current directory (and all folders inside it). You could replace `.` with `/` to search your entire hard drive, or `~/Documents` to search just your Documents folder.
* `-name "my_file.txt"` tells it exactly what file to look for. (Use `-iname` if you aren't sure about the capitalization).

**Part 2: Navigating to the file (and a gentle correction about `-R`)**
There is a slight misconception here: **`find` is naturally recursive by default**, meaning it already searches through all subfolders automatically. You don't need a `-R` flag for `find` like you do for `grep`, `cp`, or `ls`.

Furthermore, commands like `find` or `grep` cannot *navigate* you (change your current directory). Only the `cd` (change directory) command can do that.

If `find` tells you your file is located at `./backend/src/utils/my_file.txt`, you **do not** have to type `cd` 3-4 times to get there. You can just copy the path (minus the file name itself) and do it in one single jump:

```bash
cd backend/src/utils/
```

If you want a fancy shortcut to find a file and instantly jump to its directory in one command, you can combine `cd`, `dirname`, and `find` like this:

```bash
cd $(dirname $(find . -name "my_file.txt" | head -n 1))
```

---

### Question 4: What does `git show <commit-hash> --stat` do?

* **`git show <commit-hash>`** shows you the commit metadata (who made it, when, and the message) followed by the **full diff**—every single line of code that was added or removed in that commit. If the commit is huge, this output will be massive.
* **`--stat`** replaces that massive line-by-line code diff with a clean, high-level **summary statistic**.

When you run `git show <commit-hash> --stat`, Git gives you a list of the files that were modified, along with the number of insertions (`+`) and deletions (`-`) for each file. It's the best way to see *which* files were changed in a commit and *how much* they changed, without cluttering your screen with the actual code.

---

### Question 5: How to know who made a particular commit?

If you have the commit hash, there are a couple of ways to see who made it.

**1. The standard way:**
Simply run `git show <commit-hash>`. The very second line of the output will literally say `Author: Name <email@example.com>`.

**2. The exact, targeted way:**
If you want Git to output *only* the author's name and email (useful if you are writing a script or just don't want to see the commit message and stats), you can format the output:

```bash
git show -s --format='%an <%ae>' <commit-hash>
```

* `-s` suppresses the diff/stats so you only get the metadata.
* `%an` stands for Author Name.
* `%ae` stands for Author Email.
