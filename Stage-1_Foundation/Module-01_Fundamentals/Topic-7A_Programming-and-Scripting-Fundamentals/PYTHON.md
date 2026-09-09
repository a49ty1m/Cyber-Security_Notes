# Python, Data Analysis, Networking & Security — Practice Questions

Use this as a learning checklist. Solve each question yourself, test edge cases, and keep solutions in clearly named files. For networking and security exercises, use only systems, accounts, and packet captures you own or are explicitly authorized to test.

---

## 1. Core Python Fundamentals

### 1.1 Variables, Data Types, and Operators

- [ ] Create variables for a student's name, age, room number, and CGPA. Print both their values and data types.
- [ ] Take two numbers as input and display their addition, subtraction, multiplication, division, floor division, modulus, and exponentiation.
- [ ] Explain with code when to use `int`, `float`, `str`, `bool`, and `None`.
- [ ] Write a program that converts a user-entered age from a string to an integer safely (handle `ValueError`).
- [ ] Check whether a student is eligible for placement when CGPA is at least 7 and there are no backlogs.
- [ ] Predict the result of comparison and logical expressions using `and`, `or`, and `not`.
- [ ] Swap two variables with and without using a temporary variable.
- [ ] Build a simple bill calculator that applies a percentage discount and tax in the correct order.
- [ ] Use the walrus operator (`:=`) to assign and check a value inside a `while` loop condition.
- [ ] Explain the difference between `==` and `is`; give one example where they produce different results.
- [ ] Demonstrate bitwise operators (`&`, `|`, `^`, `~`, `<<`, `>>`) on small integers and explain each result.
- [ ] Use augmented assignment operators (`+=`, `-=`, `*=`, `//=`, `**=`, `%=`) inside a loop.

### 1.2 Input, Output, Conditions, and Loops

- [ ] Write a program that checks whether an integer is even or odd.
- [ ] Write a voting-eligibility checker for a user aged 18 or older.
- [ ] Take three numbers and print the largest, including the case where values are equal.
- [ ] Create a grade calculator: 90–100 → A+, 80–89 → A, 70–79 → B, 60–69 → C, below 60 → Fail.
- [ ] Print the multiplication table for a number from 1 to 10.
- [ ] Find the sum of the first `N` natural numbers using a loop.
- [ ] Calculate a factorial using a loop. Handle zero and reject negative input.
- [ ] Accept 10 numbers and display how many are even and how many are odd.
- [ ] Keep asking for the password `future123` until the correct password is entered (use `while`).
- [ ] Build a number-guessing game that reports "Too High" or "Too Low" until the answer is correct.
- [ ] Compare `for`, `while`, `break`, `continue`, and `pass` by writing one short example of each.
- [ ] Validate numeric input so that non-numeric values do not crash your program.
- [ ] Use nested loops to print a right-angled triangle of stars with `N` rows.
- [ ] Use `enumerate()` to print each item in a list along with its index — avoid a manual counter variable.
- [ ] Use `zip()` to pair two lists of equal length and print each pair on one line.

### 1.3 Strings and Built-in Functions

- [ ] Print the first, last, and middle character of a string using positive and negative indexing.
- [ ] Reverse a string with slicing and check whether it is a palindrome.
- [ ] Given a sentence, demonstrate `upper`, `lower`, `title`, `replace`, `split`, `join`, `find`, `count`, `startswith`, `endswith`, and `strip`.
- [ ] Count the vowels, consonants, digits, spaces, and special characters in a user-entered string.
- [ ] Write a program that normalizes a full name by removing extra spaces and applying title case.
- [ ] Use `len`, `max`, `min`, `sum`, `type`, `round`, `abs`, `sorted`, and `reversed` in meaningful examples.
- [ ] Find the longest word in a sentence after removing punctuation.
- [ ] Mask all but the final four characters of an account identifier without changing short identifiers.
- [ ] Use advanced f-string formatting: align text, pad numbers with leading zeros, show two decimal places, and format a large number with a thousands separator.
- [ ] Explain raw strings (`r"..."`) and when they are necessary (e.g., Windows file paths, regex patterns).

### 1.4 Lambda Functions

- [ ] Write a `lambda` that squares a number and use it directly in an expression.
- [ ] Use `lambda` with `sorted()` to sort a list of tuples by the second element.
- [ ] Use `map()` with a `lambda` to convert a list of Celsius temperatures to Fahrenheit.
- [ ] Use `filter()` with a `lambda` to keep only even numbers from a list.
- [ ] Use `reduce()` from `functools` with a `lambda` to compute the product of all numbers in a list.
- [ ] Explain when a named function is preferable to a `lambda` and when a `lambda` is the right choice.

### 1.5 Collections

#### Lists

- [ ] Create a list of five programming languages, add one language, remove another, and explain the difference between `remove`, `pop`, and `del`.
- [ ] Find the length, sum, minimum, maximum, and average of a list of 10 numbers.
- [ ] Print only the even numbers from a list of 10 numbers.
- [ ] Read five numbers from the user, store them in a list, and print the list in sorted and reversed order.
- [ ] Use a list comprehension to create squares of even numbers from 1 to 20.
- [ ] Remove duplicate values from a list while preserving the original order.
- [ ] Use `extend`, `insert`, `index`, `count`, `copy`, and `clear` on a list and describe what each does.
- [ ] Create a 2D list (matrix) and print its elements row by row using nested loops.
- [ ] Flatten a nested list using a list comprehension with two `for` clauses.

#### Tuples

- [ ] Create a tuple containing the seven days of the week and print the third item and the length.
- [ ] Check whether `"Python"` is in `("Java", "Python", "C", "JavaScript")`.
- [ ] Try changing a tuple item and explain the resulting `TypeError`.
- [ ] Count the occurrences of `10` in `(10, 20, 10, 30, 40, 10)` and find the index of the first occurrence.
- [ ] Unpack a tuple of student details into meaningful variable names.
- [ ] Convert a list to a tuple and explain a practical reason to choose an immutable sequence.
- [ ] Use a tuple as a dictionary key and explain why lists cannot be used as dictionary keys.

#### Sets

- [ ] Create a set with duplicate numbers and explain why duplicates disappear automatically.
- [ ] Add and remove student names from a set; use both `discard` and `remove` and explain their different behavior for missing values.
- [ ] Find the union, intersection, difference, and symmetric difference of two sets.
- [ ] Determine whether one set is a subset, superset, or disjoint set of another.
- [ ] Find the common skills shared by two lists of technologies using sets.
- [ ] Take five numbers from the user and display only unique values using a set.

#### Dictionaries

- [ ] Create a dictionary with `name`, `age`, and `city`, then update a value and add a new key.
- [ ] Print all dictionary keys, values, and key-value pairs using `.keys()`, `.values()`, and `.items()`.
- [ ] Store marks for five students and print each name with its mark.
- [ ] Check whether the `age` key exists before accessing it; use `get` and `setdefault` safely.
- [ ] Count word frequency in a sentence using a dictionary.
- [ ] Sort a dictionary of student marks by name and separately by marks (descending).
- [ ] Use a dictionary comprehension to invert a dictionary (swap keys and values).
- [ ] Merge two dictionaries using the `|` operator (Python 3.9+) and explain what happens when keys clash.

### 1.6 Functions, Modules, and Project Structure

- [ ] Write reusable functions for `is_even`, `calculate_grade`, and `calculate_factorial` with docstrings.
- [ ] Explain positional arguments, keyword arguments, default arguments, `*args`, and `**kwargs` with one example each.
- [ ] Return multiple values from a function and unpack them safely.
- [ ] Explain local, global, and `nonlocal` scope by correcting a scope-related bug.
- [ ] Create a module named `calculations.py`, import it into `main.py`, and use `if __name__ == "__main__":` correctly.
- [ ] Create a small package with `__init__.py` and explain the difference between absolute and relative imports.
- [ ] Organize a command-line project into `src/`, `tests/`, `data/`, `README.md`, and `requirements.txt`.
- [ ] Write a function that accepts a callback function as an argument and calls it with a result.

### 1.7 Advanced Python Features

- [ ] Write a decorator that logs a function's name, arguments, and return value before and after it runs.
- [ ] Use `functools.wraps` inside a decorator and explain why it preserves the wrapped function's `__name__` and `__doc__`.
- [ ] Write a parameterized decorator (decorator factory) that measures execution time without changing the wrapped function.
- [ ] Write a decorator that retries a function up to `N` times after a temporary failure, with a configurable retry limit.
- [ ] Write a generator function that yields Fibonacci numbers up to a user-defined limit.
- [ ] Explain generator expressions versus list comprehensions and compare their memory usage with `sys.getsizeof`.
- [ ] Use `yield from` to delegate to a sub-generator and combine two generators into one.
- [ ] Write an infinite counter generator safely, then consume only the first 10 values using `itertools.islice`.
- [ ] Write a context manager using `with` for safe file access; then write a custom context manager class using `__enter__` and `__exit__`.
- [ ] Create a `@contextmanager`-based context manager (using `contextlib`) that reports whether a block completed successfully.
- [ ] Implement an iterator class with `__iter__` and `__next__` that yields a fixed number of countdown values.

### 1.8 Object-Oriented Programming

- [ ] Create a `Student` class with name, roll number, course, a constructor (`__init__`), and a `display` method.
- [ ] Create an `Employee` class that raises salary by 10% and displays the updated data.
- [ ] Build a `BankAccount` class with `deposit`, `withdraw`, and `display_balance`; reject invalid withdrawals using a custom exception.
- [ ] Explain instance attributes versus class attributes and show how they interact when modified on an instance.
- [ ] Demonstrate `@classmethod` and `@staticmethod` using one class — explain when to use each.
- [ ] Use `@property`, `@<name>.setter`, and `@<name>.deleter` to control read/write/delete access to a private attribute.
- [ ] Create a `Person` parent class and a `Student` child class that adds roll number and course (single inheritance).
- [ ] Demonstrate multiple inheritance: create two parent classes and one child class that inherits from both; call `super()` correctly.
- [ ] Explain the Method Resolution Order (MRO) and print it with `ClassName.__mro__`.
- [ ] Demonstrate method overriding: create an `Animal` parent class with a `speak` method; override it in `Dog` and `Cat` child classes.
- [ ] Use composition: give a `Student` object an `Address` object instead of inheriting unnecessarily.
- [ ] Implement dunder/magic methods: `__str__`, `__repr__`, `__len__`, `__eq__`, `__lt__`, `__add__`, and `__contains__` in one class.
- [ ] Use `__slots__` in a class and explain how it reduces memory usage compared to the default `__dict__`.
- [ ] Use `@dataclass` to define a data-holding class and compare it with an equivalent regular class that has `__init__`, `__repr__`, and `__eq__`.
- [ ] Create an Abstract Base Class using `ABC` and `@abstractmethod`; show that it cannot be instantiated directly.

### 1.9 Files, Exceptions, and Environments

- [ ] Create `student.txt`, write your name, read it back, then append your city — use `with open(...)` throughout.
- [ ] Write multiple student records to a file and read them back; explain file modes `r`, `w`, `a`, `x`, `b`, and `+`.
- [ ] Handle `ValueError`, `ZeroDivisionError`, `FileNotFoundError`, and a custom validation exception with `try`/`except`/`else`/`finally`.
- [ ] Create a custom exception class that inherits from `Exception` and carries a descriptive error message.
- [ ] Use a custom exception to reject invalid bank-account transactions.
- [ ] Demonstrate the difference between `raise` and `raise ... from ...` (exception chaining).
- [ ] Create and activate a virtual environment named `.venv`.
- [ ] Install a package with `pip`, generate `requirements.txt` with pinned versions, and reproduce the environment in a new virtual environment.
- [ ] Explain why package versions should be pinned and how to check them for known vulnerabilities.

### 1.10 Modern Python Syntax (3.8 – 3.12)

- [ ] Use the walrus operator (`:=`) in a `while` loop to read and process input without repeating the call.
- [ ] Use `match`/`case` (Python 3.10+) to handle HTTP status codes cleanly; include a wildcard `case _:` fallback.
- [ ] Use `match`/`case` with a sequence pattern to destructure a list and with a mapping pattern to destructure a dictionary.
- [ ] Merge two dictionaries with `|` and update one in-place with `|=`; explain what happens when keys overlap.
- [ ] Show positional-only parameters (`/`) and keyword-only parameters (`*`) in a function signature.

---

## 2. Standard Library Deep Dive

### 2.1 `collections` Module

- [ ] Use `Counter` to count word frequency in a sentence and print the three most common words.
- [ ] Use `defaultdict(list)` to group students by their grade without raising a `KeyError`.
- [ ] Use `defaultdict(int)` to count character frequencies without manually initializing keys.
- [ ] Create a `namedtuple` called `Point` with `x` and `y` fields; access fields by name and by index.
- [ ] Use `deque` to implement a fixed-size sliding-window buffer with `maxlen`.
- [ ] Use `deque` to efficiently rotate a sequence left and right without slicing.
- [ ] Use `OrderedDict` to preserve insertion order and compare it with a regular `dict` (Python 3.7+).
- [ ] Use `ChainMap` to merge two configuration dictionaries and explain lookup priority.

### 2.2 `datetime` Module

- [ ] Get today's date and the current date-time; print each in at least two different formats.
- [ ] Calculate a person's age in years from their birth date using `date.today()`.
- [ ] Add and subtract `timedelta` objects (days, hours, minutes) from a date-time.
- [ ] Parse a date string (`"2025-12-31"`) into a `date` object using `strptime` and format it back with `strftime`.
- [ ] Find the number of days remaining until a future event (e.g., New Year's Day next year).
- [ ] Explain the difference between `date`, `time`, `datetime`, and `timedelta`.

### 2.3 `pathlib` Module

- [ ] Use `Path` to build file paths that work on both Windows and Linux without string concatenation.
- [ ] Check whether a path exists, is a file, or is a directory using `Path` methods.
- [ ] List all `.py` files in a directory recursively using `Path.rglob("*.py")`.
- [ ] Read a file's full contents and write new contents using `Path.read_text()` and `Path.write_text()`.
- [ ] Extract the file name, stem (name without extension), suffix (extension), and parent directory from a path.
- [ ] Create a new directory and all necessary parent directories using `Path.mkdir(parents=True, exist_ok=True)`.
- [ ] Use `pathlib` to ensure a user-supplied output file stays inside an allowed project directory (prevent path traversal).

### 2.4 `os` and `sys` Modules

- [ ] Use `os.getcwd()`, `os.listdir()`, `os.path.join()`, `os.path.exists()`, and `os.path.getsize()` in one script.
- [ ] Read environment variables with `os.environ.get()` and provide a safe default when a variable is missing.
- [ ] Use `os.walk()` to print all files and directories under a given folder recursively.
- [ ] Use `sys.argv` to read command-line arguments and print a usage message when required arguments are missing.
- [ ] Use `sys.exit()` with a non-zero exit code to signal an error from a script.
- [ ] Explain the difference between `os.path` and `pathlib.Path` and when to prefer each.

### 2.5 `itertools` Module

- [ ] Use `itertools.chain()` to iterate over multiple lists as if they were one sequence.
- [ ] Use `itertools.islice()` to safely consume only the first `N` values from an infinite generator.
- [ ] Use `itertools.count()`, `itertools.cycle()`, and `itertools.repeat()` and explain a use case for each.
- [ ] Use `itertools.combinations()` and `itertools.permutations()` to generate all possible selections from a list.
- [ ] Use `itertools.product()` to generate a Cartesian product (e.g., all suit–rank pairs for a deck of cards).
- [ ] Use `itertools.groupby()` to group a sorted list of records by a key field.
- [ ] Use `itertools.accumulate()` to compute running totals of a list of values.
- [ ] Use `itertools.takewhile()` and `itertools.dropwhile()` to filter a sequence based on a predicate.

### 2.6 `functools` Module

- [ ] Use `functools.reduce()` to compute the product of a list without an explicit loop.
- [ ] Use `functools.partial()` to create a specialized version of `pow` that always cubes its argument.
- [ ] Use `functools.lru_cache()` to memoize a recursive Fibonacci function and measure the speed difference.
- [ ] Use `functools.wraps` in a decorator and explain what is lost if you omit it.
- [ ] Use `functools.total_ordering` to define only `__eq__` and `__lt__` on a class and automatically get all other comparison methods.

### 2.7 `argparse` Module

- [ ] Write a command-line script that accepts `--input FILE`, `--output FILE`, and `--verbose` flags using `argparse`.
- [ ] Add `--help` text, a `--version` flag, and a required positional argument to an `argparse` parser.
- [ ] Use `type=int` and `choices=[...]` in `argparse` to validate numeric and enumerated arguments automatically.
- [ ] Use subcommands (`add_subparsers`) to build a CLI tool with `create`, `read`, and `delete` sub-commands.
- [ ] Parse a list of values with `nargs="+"` and explain the difference from `nargs="*"`.

### 2.8 `logging` Module

- [ ] Configure the `logging` module with `DEBUG`, `INFO`, `WARNING`, `ERROR`, and `CRITICAL` levels; show which messages appear at each level.
- [ ] Write logs to a rotating file with `logging.handlers.RotatingFileHandler` that caps log size and keeps three backups.
- [ ] Use a custom log formatter that includes timestamp, level, module name, and line number.
- [ ] Log a caught exception with full traceback using `logging.exception()` inside an `except` block.
- [ ] Demonstrate how to redact a sensitive field (e.g., an API token) from log output using a custom `logging.Filter`.
- [ ] Use separate loggers per module with `logging.getLogger(__name__)` and explain why this is better than the root logger.

### 2.9 `subprocess` Module

- [ ] Run a simple shell command using `subprocess.run()` with `capture_output=True` and print its stdout.
- [ ] Compare `subprocess.run(shell=True, ...)` with a list-based invocation; explain the security risk of `shell=True`.
- [ ] Check the return code of a subprocess and raise an error if it is non-zero using `check=True`.
- [ ] Pass environment variables to a subprocess with the `env` parameter without leaking parent-process secrets.
- [ ] Use `subprocess.Popen` to start a long-running process, read its output line by line, and terminate it cleanly.

### 2.10 `re` — Regular Expressions

- [ ] Write patterns to validate an email-like string, an IPv4 address, a date (`YYYY-MM-DD`), and a phone number.
- [ ] Use `re.search`, `re.match`, `re.findall`, `re.finditer`, `re.sub`, and `re.split` and describe how each differs.
- [ ] Explain raw strings (`r"..."`), groups `()`, named groups `(?P<name>...)`, quantifiers (`*`, `+`, `?`, `{n,m}`), and anchors (`^`, `$`).
- [ ] Extract log entries using a named-group pattern for timestamp, severity, and message.
- [ ] Write tests that distinguish valid IPv4 addresses from look-alike invalid strings (e.g., `256.0.0.1`).
- [ ] Redact an API token or password from a log line using `re.sub`.
- [ ] Explain catastrophic backtracking and rewrite a vulnerable pattern safely.

### 2.11 `json`, `csv`, `xml`, and `yaml`

- [ ] Use `json.load`, `json.dump`, `json.loads`, and `json.dumps` to read, modify, and write a JSON configuration file.
- [ ] Validate that a JSON configuration has all required keys and correct value types before using it.
- [ ] Read a CSV with `csv.DictReader`, validate required columns, and write a cleaned output CSV with `csv.DictWriter`.
- [ ] Detect malformed CSV rows, report their line numbers, and continue processing valid rows without crashing.
- [ ] Convert validated CSV records to a structured JSON file; report rows that could not be converted.
- [ ] Parse a local XML document with `xml.etree.ElementTree`, extract selected elements, and turn records into dictionaries.
- [ ] Explain why untrusted XML should be parsed with defenses against XML bomb and XXE attacks.
- [ ] Load YAML configuration with `yaml.safe_load`, validate expected fields, and explain why `yaml.load` without a `Loader` is dangerous.
- [ ] Build a file-normalization utility that reads CSV, JSON, XML, or YAML and produces one validated JSON output format.

### 2.12 `typing` Module

- [ ] Add type hints to a function using `int`, `str`, `float`, `bool`, `-> None`, and `list[int]` (Python 3.9+).
- [ ] Use `Optional[str]` and `Union[int, str]` to annotate parameters that may be `None` or multiple types.
- [ ] Use `Callable[[int, int], int]` to annotate a function that accepts another function as a parameter.
- [ ] Use `TypeVar` to write a generic `identity` function that preserves the return type.
- [ ] Use `TypedDict` to define the expected shape of a dictionary and annotate a function that accepts it.
- [ ] Run `mypy` on a small module, interpret all reported type errors, and fix them.
- [ ] Explain `Any` and when its use defeats the purpose of type checking.

### 2.13 `urllib.parse`

- [ ] Split a URL into scheme, host, path, and query components using `urlparse`.
- [ ] Build an API URL safely from a base URL and user-supplied query values using `urlencode`.
- [ ] Encode and decode URL-percent-encoded strings with `quote` and `unquote`.
- [ ] Identify and reject URL schemes other than `https` in a downloader configuration.
- [ ] Use `urljoin` to resolve relative links found in parsed HTML.

---

## 3. Mini Projects

- [ ] Build a menu-driven ATM: check balance, deposit, withdraw, and exit — prevent overdrafts and validate all input.
- [ ] Build a student result manager: accept several students' names, marks, and attendance; print pass/fail, grade, percentage, and an attendance warning. Validate marks (0–100), attendance (0–100), and duplicate student names.
- [ ] Save the student result manager's records to a file and reload them when the program starts.
- [ ] Refactor the result manager into a package with separate modules for models, services, utilities, and the CLI entry point.
- [ ] Add a search feature that finds a student by name or roll number.
- [ ] Add an edit/delete menu option, confirm the action, and save updated data safely.
- [ ] Write automated `pytest` tests for the grade, attendance-warning, deposit, and withdrawal rules.
- [ ] Write a menu-driven contact book using functions, lists, dictionaries, file storage, and input validation.
- [ ] Create a command-line expense tracker that calculates category totals and monthly spending.
- [ ] Read a text file and report its line count, word count, character count, and most frequent word.
- [ ] Build the Hangman game: random word selection, masked display, limited guesses, and a replay option.

---

## 4. NumPy Practice Questions

### 4.1 Arrays and Attributes

- [ ] Import NumPy and create 1D arrays of integers, decimal values, employee IDs, even numbers, and odd numbers.
- [ ] Compare multiplying a Python list by 2 with multiplying a NumPy array by 2 — explain the difference.
- [ ] Create a 3×3 matrix, a 2×4 matrix, marks for three students across three subjects, and a 3D array containing two 2×3 matrices.
- [ ] For a 2D array, print `ndim`, `shape`, `size`, `dtype`, `itemsize`, `nbytes`, and `T` (transpose).
- [ ] Create arrays with `zeros`, `ones`, `eye`, `arange`, and `linspace`; explain a real use case for each.
- [ ] Convert a Python list to a NumPy array and explicitly change its data type with `astype`.
- [ ] Create a reproducible array of random integers using `np.random.default_rng(seed=42)`.
- [ ] Reshape a 1D array of 12 elements into shapes `(3, 4)`, `(2, 6)`, and `(2, 2, 3)` using `reshape`.

### 4.2 Operations and Broadcasting

- [ ] Perform element-wise addition, subtraction, multiplication, division, floor division, exponentiation, and modulus on two equal-shaped arrays.
- [ ] Add 100 to every value, multiply every value by 3, and square and cube each value in an array.
- [ ] Add a column vector to a row vector; explain the broadcasting rule that makes the shapes compatible.
- [ ] Intentionally trigger a broadcasting shape-mismatch error; explain how to reshape one array to fix it.
- [ ] Use aggregation functions to find `sum`, `mean`, `min`, and `max` along each axis of a 2D array.
- [ ] Round decimal array values and explain the difference between element-wise operations and matrix multiplication (`@` or `np.dot`).
- [ ] Increase employee salaries by ₹5,000, give a 20% increment, and reduce electricity bills by ₹200 using vectorized operations.

### 4.3 Indexing, Slicing, and Filtering

- [ ] Print the first, third, last, and second-last elements of an array; then change the second and last values in-place.
- [ ] Slice: first four values, last three values, every second value, indices 2–5, and a reversed array.
- [ ] Iterate through all values in a 1D array, all rows in a 2D array, and every individual element using `nditer`.
- [ ] Use Boolean indexing to find all values greater than 50, below 30, even, odd, and between two bounds.
- [ ] Use a Boolean mask to replace every negative value in an array with zero.
- [ ] Extract a rectangular submatrix using row and column slices; use fancy indexing with an index array.
- [ ] Print passing marks (≥ 50), failing marks, and marks above 80 from a marks array.

### 4.4 Applied NumPy Challenges

- [ ] Build a payroll script: increase all salaries by 20%, identify salaries above ₹50,000, give a ₹3,000 raise to those below ₹40,000, and display final values.
- [ ] Build a student marks script: add five grace marks, identify passing students, flag scores above 80, and print updated marks.
- [ ] Write a function that receives a numeric array and returns count, mean, median, min, max, and standard deviation.
- [ ] Create a 2D marks matrix and calculate each student's total, average, highest score, and pass/fail result.
- [ ] Normalize a numeric array to a 0–1 range; handle the case where all values are identical.
- [ ] Combine two compatible arrays using `np.concatenate`, `np.vstack`, and `np.hstack`; explain when to use each.
- [ ] Save a NumPy array to disk with `np.save` / `np.savetxt` and load it back without losing shape or type.
- [ ] Compare a loop-based calculation with a vectorized NumPy calculation; measure the time difference with `time.perf_counter`.

---

## 5. Pandas and Data Analysis

### 5.1 DataFrame Basics

- [ ] Load the Amazon sales CSV with `pd.read_csv` and display the first five rows, last five rows, and five random rows.
- [ ] Print the DataFrame's `shape`, `columns`, `index`, `dtypes`, `info()`, and `describe()` (for both numeric and object columns).
- [ ] Explain the difference between a Pandas `Series` and a `DataFrame` with a code example for each.
- [ ] Select single columns, multiple columns, and a column subset; explain `[[]]` vs `[]` notation.
- [ ] Use `iloc` to select row 0, rows 0–4, rows 20–30, and a rectangular row/column slice.
- [ ] Use `loc` to select rows and columns by label; apply a Boolean filter using `loc`.
- [ ] Explain when `loc` is preferable to `iloc` and when `iloc` is the right choice.
- [ ] Rename columns, create a derived column (e.g., `Profit_Margin = Profit / Sales * 100`), and save the cleaned DataFrame to a new CSV.

### 5.2 Amazon Sales Analysis Questions

- [ ] How many orders are in the dataset? What is the data type of the Profit column?
- [ ] What is the minimum customer age, and which customer has the highest single-order profit?
- [ ] Filter and display all Electronics records; repeat for Fashion, Books, Beauty, and Home.
- [ ] Display orders where profit > 5,000, customer age > 40, delivery days > 7, discount < 10, and rating < 3.
- [ ] Calculate total sales and total profit by category; identify the best-performing category by each measure.
- [ ] Find the average sales, profit, discount, rating, and delivery days by category using `groupby`.
- [ ] Identify the five highest-profit orders and the five largest-discount orders using `nlargest`.
- [ ] Check for missing values and duplicate orders; explain how you would handle each issue.
- [ ] Create a reusable `generate_report(df)` function that prints key sales metrics to the console.
- [ ] Plot a bar chart of total sales by category and a histogram of customer ages using the DataFrame's `.plot()` method.

---

## 6. Networking Programming

### 6.1 Networking Foundations

- [ ] Explain IP addresses, ports, sockets, TCP, UDP, client–server architecture, and the TCP three-way handshake.
- [ ] Explain the differences among HTTP, HTTPS, DNS, TLS, and SSH.
- [ ] Use `socket.gethostbyname` and `socket.getaddrinfo` to resolve a hostname and compare the results.
- [ ] Describe DNS record types A, AAAA, CNAME, MX, NS, TXT, and PTR; explain the purpose of each.
- [ ] Explain private versus public IP ranges, loopback addresses, and why `127.0.0.1` is useful for practice.
- [ ] Trace the full lifecycle of an HTTPS request: DNS lookup → TCP connect → TLS handshake → HTTP request/response.
- [ ] Explain NAT, firewalls, proxies, and load balancers at a conceptual level.
- [ ] Convert a port number to a service name using `socket.getservbyport`.

### 6.2 Socket Programming

- [ ] Create a TCP echo server bound to `127.0.0.1` and a matching client that sends and receives text.
- [ ] Extend the TCP server to serve multiple local clients using threads.
- [ ] Create a UDP client and server on localhost; explain how UDP differs from TCP in reliability and ordering.
- [ ] Add connection timeouts, safe message framing (length-prefix protocol), structured logging, and clean shutdown to the TCP server.
- [ ] Build a local chat application with usernames and broadcast messages to all connected clients.
- [ ] Test a localhost server with malformed, oversized, and empty input; reject invalid messages gracefully.
- [ ] Use `selectors` or `asyncio` to handle multiple local client connections without one thread per client.
- [ ] Write a localhost TCP time server that returns the current ISO timestamp when a client sends `TIME\n`.
- [ ] Add a `QUIT` command to a local client/server pair that closes the connection cleanly on both ends.
- [ ] Demonstrate the difference between a blocking and a non-blocking socket with a short local example.

### 6.3 HTTP, REST APIs, and Concurrency

- [ ] Send HTTP GET, POST, PUT, PATCH, and DELETE requests with `requests` to a safe test API; inspect status code, headers, and JSON response.
- [ ] Explain REST resources, HTTP methods, status codes (2xx, 3xx, 4xx, 5xx), query parameters, headers, and pagination.
- [ ] Write a client that authenticates to a documented test API using a Bearer token stored in an environment variable.
- [ ] Implement exponential backoff with jitter for transient failures; respect a `Retry-After` header when present.
- [ ] Compare certificate verification enabled vs disabled; explain why verification must never be disabled in production.
- [ ] Use `ssl` to inspect the certificate presented by an HTTPS host (subject, issuer, validity dates).
- [ ] Query A, AAAA, MX, NS, and TXT records with `dnspython` for a domain you are permitted to examine.
- [ ] Resolve a hostname to all returned IPv4 and IPv6 addresses; print results in a readable table.
- [ ] Use `urllib.parse` to safely construct an API URL from a base URL and user-supplied query values.
- [ ] Fetch several permitted URLs concurrently with `asyncio.gather` and limit concurrency with `asyncio.Semaphore`.
- [ ] Write the same I/O-bound networking task three ways — `threading`, `multiprocessing`, and `asyncio` — and compare performance and code complexity.
- [ ] Measure sequential vs concurrent request time for several local or permitted HTTP requests.
- [ ] Explain how a client should handle: DNS failure, connection refused, timeout, TLS verification failure, and a 500 response.

### 6.4 `asyncio` Deep Dive

- [ ] Write a basic `async def` coroutine and run it with `asyncio.run()`.
- [ ] Use `await asyncio.sleep()` inside a coroutine and explain the difference from `time.sleep()`.
- [ ] Create multiple tasks with `asyncio.create_task()` and gather their results with `asyncio.gather()`.
- [ ] Use `asyncio.Queue` to implement a producer–consumer pattern where multiple producers and consumers run concurrently.
- [ ] Handle `asyncio.TimeoutError` by wrapping a coroutine with `asyncio.wait_for(coroutine, timeout=5)`.
- [ ] Explain the difference between a coroutine, a task, and a future in `asyncio`.

---

## 7. Python Security Libraries

### 7.1 HTTP and Web Automation

- [ ] **requests:** Use `Session` objects, custom headers, query parameters, cookies, basic auth, and bearer tokens.
- [ ] **requests:** Download a permitted JSON resource, validate its content type, and save a formatted local copy.
- [ ] **requests:** Create an HTTP client wrapper that retries only idempotent requests after selected transient status codes.
- [ ] **BeautifulSoup (bs4):** Parse a saved HTML file and extract page title, links, headings, tables, and elements by CSS selector.
- [ ] **BeautifulSoup (bs4):** Extract all unique, absolute links from a saved HTML page using `urljoin`.
- [ ] **BeautifulSoup (bs4):** Parse a local HTML table into a list of dictionaries and export it as CSV.
- [ ] **BeautifulSoup (bs4):** Parse pagination links from saved HTML and deduplicate the resulting URLs.
- [ ] **BeautifulSoup (bs4):** Build a polite scraper for a site you control, respecting its terms, `robots.txt`, rate limits, and errors.
- [ ] **Selenium:** Automate a local or permitted test page: open it, fill a form, wait for dynamic content, take a screenshot, and close the driver.
- [ ] **Selenium:** Write stable selectors and explain why explicit waits are safer than fixed `time.sleep` delays.
- [ ] **Selenium:** Test validation messages on a form in a local demo page without using fixed sleeps.
- [ ] **Selenium:** Capture browser console errors from a local test application and include them in a test report.
- [ ] Compare `requests`, BeautifulSoup, and Selenium: state when each is the appropriate choice.
- [ ] Write a web-automation configuration that defines a target allowlist, rate limit, timeout, and user agent.

### 7.2 Networking and Reconnaissance

- [ ] **socket:** Build an authorized localhost service checker that tests a fixed allowlist of ports and reports open/closed/error states.
- [ ] **socket:** Write a function that validates hostnames and ports before attempting a connection to an authorized target.
- [ ] **socket:** Use `socket.create_connection` with a timeout and return structured success or failure data.
- [ ] **socket:** Implement a banner reader only for a service you operate or are authorized to test; handle timeouts gracefully.
- [ ] **dnspython:** Enumerate A, AAAA, MX, NS, and TXT records for a domain you are authorized to query.
- [ ] **dnspython:** Retrieve MX records, sort them by priority, and print a readable mail-routing summary.
- [ ] **dnspython:** Compare a domain's A and AAAA records, and explain what an empty answer means.
- [ ] **dnspython:** Handle DNS timeouts and `NXDOMAIN` responses without crashing.
- [ ] **scapy:** Create and inspect packets in an isolated lab or offline PCAP; identify Ethernet, IP, TCP, UDP, and ICMP fields.
- [ ] **scapy:** Analyze a lab capture to count protocols, flag malformed fields, and extract DNS query names — without transmitting traffic.
- [ ] **scapy:** Load an offline capture and count TCP flags: SYN, ACK, FIN, RST.
- [ ] **pyshark:** Read an offline PCAP and summarize source/destination pairs, protocols, ports, and packet counts.
- [ ] **pyshark:** Filter an offline PCAP for DNS or HTTP traffic and export a small CSV summary.
- [ ] Compare Scapy and PyShark: installation, packet representation, filtering capabilities, and best use cases.

### 7.3 Cryptography and Secure Communications

- [ ] **cryptography:** Hash a message with SHA-256 and explain why passwords need a dedicated scheme (e.g., bcrypt, Argon2) rather than plain hashing.
- [ ] **cryptography:** Generate a Fernet key, encrypt a local test message, and prove that altering the ciphertext causes decryption to fail.
- [ ] **cryptography:** Derive an encryption key from a password using a salt and an approved KDF (e.g., PBKDF2HMAC or Scrypt).
- [ ] **cryptography:** Generate a signing key pair, sign a message, and verify the signature; explain what a valid signature proves.
- [ ] **cryptography:** Inspect an X.509 certificate's subject, issuer, validity period, and public key algorithm.
- [ ] **cryptography:** Verify certificate validity dates and report whether a certificate is currently valid.
- [ ] **cryptography:** Rotate a symmetric encryption key in a test application without exposing either key in source control.
- [ ] **cryptography:** Compare hashing, encryption, encoding, and signing — give one practical example of each.
- [ ] **paramiko:** Connect to an SSH server you administer using key-based authentication and run a harmless command.
- [ ] **paramiko:** Upload and download a test file over SFTP with host-key verification enabled.
- [ ] **paramiko:** Load a private key from a protected local path without printing its contents.
- [ ] **paramiko:** Use a known-hosts file and explain how it prevents connecting to an unexpected server.
- [ ] **paramiko:** Transfer a test directory listing over SFTP and handle connection and authentication failures cleanly.
- [ ] Explain the purpose of SSH host-key verification and the risk of automatically accepting unknown host keys.
- [ ] Design a key-rotation plan for an application that uses encrypted configuration values.

### 7.4 Advanced, Controlled Learning

- [ ] **pwntools (learn later):** Set up an isolated CTF or intentionally vulnerable local binary environment and learn the library's basic process and ELF-inspection APIs.
- [ ] **pwntools (learn later):** Practice only against CTF targets or binaries you own; document the target, authorization, and cleanup steps for every exercise.
- [ ] **pwntools (learn later):** Explain architecture, endianness, registers, and calling conventions before using the library.
- [ ] **pwntools (learn later):** Read an ELF's basic metadata in a CTF lab and explain PIE, NX, RELRO, and stack canaries at a high level.
- [ ] **pwntools (learn later):** Write a lab report that focuses on mitigation learning and defensive fixes, not deployment against real targets.

---

## 8. Authorized Automation Projects

- [ ] Build a localhost port checker with a small allowlist, connection timeout, clear consent prompt, and CSV/JSON report.
- [ ] Build an authorized banner grabber for your own test services, with rate limiting and graceful error handling.
- [ ] Build a directory/content discovery tool only for a lab site you own; include a fixed target scope, delay, and audit log.
- [ ] Build a web crawler for your own site that respects scope, `robots.txt` rules, depth limits, duplicate URLs, and rate limits.
- [ ] Build a log analyzer that extracts timestamps, levels, IPs, URLs, and error patterns from local logs using `re`.
- [ ] Build an IOC extractor for local text files that finds and deduplicates domains, IPs, URLs, and hashes, then exports a report.
- [ ] Build an API automation client with configuration files, retries, pagination, structured logs, and unit tests.
- [ ] Build an HTML or Markdown report generator from structured JSON/CSV results.
- [ ] Build a vulnerability configuration checker for an application or infrastructure you own; make checks non-destructive and explain every finding.
- [ ] Build an offline packet parser that summarizes a provided PCAP without capturing or transmitting live traffic.
- [ ] Add `argparse` options (`--help`, `--dry-run`, `--verbose`, `--output`), input validation, and meaningful exit codes to one of your automation projects.
- [ ] Add a dry-run mode that prints what the tool would do without making network requests or modifying files.
- [ ] Add structured JSON logs with a unique run identifier to an automation tool.
- [ ] Create a configuration schema and validate all configuration fields before the project begins work.
- [ ] Add rate limiting and a maximum-runtime safeguard to a permitted crawler or service checker.
- [ ] Generate both an executive summary and a technical-detail report from the same structured result data.
- [ ] Write unit tests using mocked network responses rather than relying on live internet services.
- [ ] Create a cleanup routine that closes files, sockets, and browser drivers even after errors.

---

## 9. Secure Coding Practices

- [ ] Validate type, length, range, format, and allowlisted values for every external input a script accepts.
- [ ] Create a validation function for a user record; test empty, oversized, malformed, and valid values.
- [ ] Explain why `eval`, `exec`, unsafe deserialization, unchecked subprocess calls, and string-built SQL are dangerous; demonstrate safe alternatives.
- [ ] Compare `subprocess.run(shell=True, ...)` with a safe argument-list invocation; explain the security risk of `shell=True`.
- [ ] Use `pathlib` to ensure a requested output file remains inside an allowed project directory (prevent path traversal).
- [ ] Design error messages for users and log entries for developers — without exposing secrets, stack traces, or internal infrastructure details.
- [ ] Configure the `logging` module with levels, timestamps, a rotating file handler, and redaction of sensitive fields.
- [ ] Store a sample API token in an environment variable, confirm that logs redact it, and verify it is excluded from version control.
- [ ] Load secrets from environment variables or an approved secret manager; add `.env` and private keys to `.gitignore`.
- [ ] Separate configuration by environment (development, test, production) and validate it at startup.
- [ ] Use virtual environments, version pinning, dependency review, and vulnerability scanning as part of dependency management.
- [ ] Create a dependency-update checklist that includes tests, changelog review, and vulnerability checks.
- [ ] Configure rotating logs and verify that an exception is recorded with useful, non-sensitive context.
- [ ] Write docstrings, type hints, a README setup guide, and a responsible-use statement for every security-related tool you build.
- [ ] Review one project for hard-coded secrets, overly broad permissions, unsafe defaults, and missing error handling.
- [ ] Add a security section to a README describing authorized use, data handling, and a process for reporting issues.

---

## 10. Code Quality and Professional Workflow

- [ ] Use a debugger (`pdb` or IDE breakpoints) to trace a failing program and identify the root cause — avoid relying on `print` statements.
- [ ] Profile a slow function with `cProfile`; identify the bottleneck and fix it using measured evidence.
- [ ] Write `pytest` unit tests covering normal cases, boundary values, invalid input, and expected exceptions for one project.
- [ ] Use `pytest` fixtures to share setup code across tests and `@pytest.mark.parametrize` to cover multiple input cases in one test function.
- [ ] Write a failing test before fixing a bug (test-driven approach); demonstrate that the test passes after the fix.
- [ ] Mock a file, clock, or network dependency in a unit test using `unittest.mock.patch`; explain why the mock is necessary.
- [ ] Format code with `black` (or `autopep8`), lint it with `flake8` or `ruff`, and explain each category of issue you fix.
- [ ] Add type hints to a module and run `mypy` to find and fix all reported type errors.
- [ ] Initialize a Git repository, make atomic focused commits, use a feature branch, resolve a simple merge conflict, and write a useful `.gitignore`.
- [ ] Write a descriptive commit message following a convention (e.g., Conventional Commits) and explain why commit messages matter.
- [ ] Create a `README.md` that describes purpose, installation, configuration, usage, tests, limitations, and authorization boundaries.
- [ ] Conduct a self-code-review checklist covering naming, complexity, duplication, tests, error handling, security, and documentation.
- [ ] Set up a formatter, linter, and test command that any developer can run consistently from the project root.
- [ ] Create a release checklist that verifies tests pass, dependencies are pinned, version number is bumped, documentation is current, and configuration defaults are safe.

---

## 11. Learning Outcome

- [ ] Build a professional Python automation or security-oriented tool from scratch with a clear scope, modular design, tests, documentation, secure configuration, logging, and a reproducible environment.
- [ ] Explain every dependency and major design decision in the tool.
- [ ] Demonstrate that the tool handles valid input, invalid input, expected failures, and cleanup correctly.
- [ ] Package the tool so a new user can install and run it from documented instructions alone.
- [ ] Produce a sample report using only synthetic or authorized data.
- [ ] Review the tool against a security checklist before every release.
- [ ] Obtain explicit written authorization and define scope before testing any non-local system.
- [ ] Present the tool's limitations and safe-use boundaries to another learner.
- [ ] Build one original automation or security tool from scratch — clean structure, full tests, complete docs, and secure configuration included.
