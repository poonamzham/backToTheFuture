When printing tracebacks, the interpreter will now point to the exact expression that caused the error, instead of just the line. For example:

Traceback (most recent call last):
  File "distance.py", line 11, in <module>
    print(manhattan_distance(p1, p2))
          ^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "distance.py", line 6, in manhattan_distance
    return abs(point_1.x - point_2.x) + abs(point_1.y - point_2.y)
                           ^^^^^^^^^
AttributeError: 'NoneType' object has no attribute 'x'

Previous versions of the interpreter would point to just the line, making it ambiguous which object was None. These enhanced errors can also be helpful when dealing with deeply nested dict objects and multiple function calls:

Traceback (most recent call last):
  File "query.py", line 37, in <module>
    magic_arithmetic('foo')
  File "query.py", line 18, in magic_arithmetic
    return add_counts(x) / 25
           ^^^^^^^^^^^^^
  File "query.py", line 24, in add_counts
    return 25 + query_user(user1) + query_user(user2)
                ^^^^^^^^^^^^^^^^^
  File "query.py", line 32, in query_user
    return 1 + query_count(db, response['a']['b']['c']['user'], retry=True)
                               ~~~~~~~~~~~~~~~~~~^^^^^
TypeError: 'NoneType' object is not subscriptable

As well as complex arithmetic expressions:

Traceback (most recent call last):
  File "calculation.py", line 54, in <module>
    result = (x / y / z) * (a / b / c)
              ~~~~~~^~~
ZeroDivisionError: division by zero

Additionally, the information used by the enhanced traceback feature is made available via a general API, that can be used to correlate bytecode instructions with source code location. This information can be retrieved using:

    The codeobject.co_positions() method in Python.

    The PyCode_Addr2Location() function in the C API.

See PEP 657 for more details. (Contributed by Pablo Galindo, Batuhan Taskaya and Ammar Askar in bpo-43950.)

Note

This feature requires storing column positions in Code Objects, which may result in a small increase in interpreter memory usage and disk usage for compiled Python files. To avoid storing the extra information and deactivate printing the extra traceback information, use the -X no_debug_ranges command line option or the PYTHONNODEBUGRANGES environment variable.
PEP 654: Exception Groups and except*

PEP 654 introduces language features that enable a program to raise and handle multiple unrelated exceptions simultaneously. The builtin types ExceptionGroup and BaseExceptionGroup make it possible to group exceptions and raise them together, and the new except* syntax generalizes except to match subgroups of exception groups.

See PEP 654 for more details.

(Contributed by Irit Katriel in bpo-45292. PEP written by Irit Katriel, Yury Selivanov and Guido van Rossum.)
PEP 678: Exceptions can be enriched with notes

The add_note() method is added to BaseException. It can be used to enrich exceptions with context information that is not available at the time when the exception is raised. The added notes appear in the default traceback.

See PEP 678 for more details.

(Contributed by Irit Katriel in bpo-45607. PEP written by Zac Hatfield-Dodds.)
Windows py.exe launcher improvements

The copy of the Python Launcher for Windows included with Python 3.11 has been significantly updated. It now supports company/tag syntax as defined in PEP 514 using the -V:<company>/<tag> argument instead of the limited -<major>.<minor>. This allows launching distributions other than PythonCore, the one hosted on python.org.

When using -V: selectors, either company or tag can be omitted, but all installs will be searched. For example, -V:OtherPython/ will select the “best” tag registered for OtherPython, while -V:3.11 or -V:/3.11 will select the “best” distribution with tag 3.11.

When using the legacy -<major>, -<major>.<minor>, -<major>-<bitness> or -<major>.<minor>-<bitness> arguments, all existing behaviour should be preserved from past versions, and only releases from PythonCore will be selected. However, the -64 suffix now implies “not 32-bit” (not necessarily x86-64), as there are multiple supported 64-bit platforms. 32-bit runtimes are detected by checking the runtime’s tag for a -32 suffix. All releases of Python since 3.5 have included this in their 32-bit builds.
New Features Related to Type Hints

This section covers major changes affecting PEP 484 type hints and the typing module.
PEP 646: Variadic generics

PEP 484 previously introduced TypeVar, enabling creation of generics parameterised with a single type. PEP 646 adds TypeVarTuple, enabling parameterisation with an arbitrary number of types. In other words, a TypeVarTuple is a variadic type variable, enabling variadic generics.

This enables a wide variety of use cases. In particular, it allows the type of array-like structures in numerical computing libraries such as NumPy and TensorFlow to be parameterised with the array shape. Static type checkers will now be able to catch shape-related bugs in code that uses these libraries.

See PEP 646 for more details.

(Contributed by Matthew Rahtz in bpo-43224, with contributions by Serhiy Storchaka and Jelle Zijlstra. PEP written by Mark Mendoza, Matthew Rahtz, Pradeep Kumar Srinivasan, and Vincent Siles.)
PEP 655: Marking individual TypedDict items as required or not-required

Required and NotRequired provide a straightforward way to mark whether individual items in a TypedDict must be present. Previously, this was only possible using inheritance.

All fields are still required by default, unless the total parameter is set to False, in which case all fields are still not-required by default. For example, the following specifies a TypedDict with one required and one not-required key:

class Movie(TypedDict):
   title: str
   year: NotRequired[int]

m1: Movie = {"title": "Black Panther", "year": 2018}  # OK
m2: Movie = {"title": "Star Wars"}  # OK (year is not required)
m3: Movie = {"year": 2022}  # ERROR (missing required field title)

The following definition is equivalent:

class Movie(TypedDict, total=False):
   title: Required[str]
   year: int

See PEP 655 for more details.

(Contributed by David Foster and Jelle Zijlstra in bpo-47087. PEP written by David Foster.)
PEP 673: Self type

The new Self annotation provides a simple and intuitive way to annotate methods that return an instance of their class. This behaves the same as the TypeVar-based approach specified in PEP 484, but is more concise and easier to follow.

Common use cases include alternative constructors provided as classmethods, and __enter__() methods that return self:

class MyLock:
    def __enter__(self) -> Self:
        self.lock()
        return self

    ...

class MyInt:
    @classmethod
    def fromhex(cls, s: str) -> Self:
        return cls(int(s, 16))

    ...

Self can also be used to annotate method parameters or attributes of the same type as their enclosing class.

See PEP 673 for more details.

(Contributed by James Hilton-Balfe in bpo-46534. PEP written by Pradeep Kumar Srinivasan and James Hilton-Balfe.)
PEP 675: Arbitrary literal string type

The new LiteralString annotation may be used to indicate that a function parameter can be of any literal string type. This allows a function to accept arbitrary literal string types, as well as strings created from other literal strings. Type checkers can then enforce that sensitive functions, such as those that execute SQL statements or shell commands, are called only with static arguments, providing protection against injection attacks.

For example, a SQL query function could be annotated as follows:

def run_query(sql: LiteralString) -> ...
    ...

def caller(
    arbitrary_string: str,
    query_string: LiteralString,
    table_name: LiteralString,
) -> None:
    run_query("SELECT * FROM students")       # ok
    run_query(query_string)                   # ok
    run_query("SELECT * FROM " + table_name)  # ok
    run_query(arbitrary_string)               # type checker error
    run_query(                                # type checker error
        f"SELECT * FROM students WHERE name = {arbitrary_string}"
    )

See PEP 675 for more details.

(Contributed by Jelle Zijlstra in bpo-47088. PEP written by Pradeep Kumar Srinivasan and Graham Bleaney.)
PEP 681: Data class transforms

dataclass_transform may be used to decorate a class, metaclass, or a function that is itself a decorator. The presence of @dataclass_transform() tells a static type checker that the decorated object performs runtime “magic” that transforms a class, giving it dataclass-like behaviors.

For example:

# The create_model decorator is defined by a library.
@typing.dataclass_transform()
def create_model(cls: Type[T]) -> Type[T]:
    cls.__init__ = ...
    cls.__eq__ = ...
    cls.__ne__ = ...
    return cls

# The create_model decorator can now be used to create new model classes:
@create_model
class CustomerModel:
    id: int
    name: str

c = CustomerModel(id=327, name="Eric Idle")

See PEP 681 for more details.

(Contributed by Jelle Zijlstra in gh-91860. PEP written by Erik De Bonte and Eric Traut.)
PEP 563 may not be the future

PEP 563 Postponed Evaluation of Annotations (the from __future__ import annotations future statement) that was originally planned for release in Python 3.10 has been put on hold indefinitely. See this message from the Steering Council for more information.
Other Language Changes

    Starred unpacking expressions can now be used in for statements. (See bpo-46725 for more details.)

    Asynchronous comprehensions are now allowed inside comprehensions in asynchronous functions. Outer comprehensions implicitly become asynchronous in this case. (Contributed by Serhiy Storchaka in bpo-33346.)

    A TypeError is now raised instead of an AttributeError in with statements and contextlib.ExitStack.enter_context() for objects that do not support the context manager protocol, and in async with statements and contextlib.AsyncExitStack.enter_async_context() for objects not supporting the asynchronous context manager protocol. (Contributed by Serhiy Storchaka in bpo-12022 and bpo-44471.)

    Added object.__getstate__(), which provides the default implementation of the __getstate__() method. copying and pickleing instances of subclasses of builtin types bytearray, set, frozenset, collections.OrderedDict, collections.deque, weakref.WeakSet, and datetime.tzinfo now copies and pickles instance attributes implemented as slots. This change has an unintended side effect: It trips up a small minority of existing Python projects not expecting object.__getstate__() to exist. See the later comments on gh-70766 for discussions of what workarounds such code may need. (Contributed by Serhiy Storchaka in bpo-26579.)

    Added a -P command line option and a PYTHONSAFEPATH environment variable, which disable the automatic prepending to sys.path of the script’s directory when running a script, or the current directory when using -c and -m. This ensures only stdlib and installed modules are picked up by import, and avoids unintentionally or maliciously shadowing modules with those in a local (and typically user-writable) directory. (Contributed by Victor Stinner in gh-57684.)

    A "z" option was added to the Format Specification Mini-Language that coerces negative to positive zero after rounding to the format precision. See PEP 682 for more details. (Contributed by John Belmonte in gh-90153.)

    Bytes are no longer accepted on sys.path. Support broke sometime between Python 3.2 and 3.6, with no one noticing until after Python 3.10.0 was released. In addition, bringing back support would be problematic due to interactions between -b and sys.path_importer_cache when there is a mixture of str and bytes keys. (Contributed by Thomas Grainger in gh-91181.)

Other CPython Implementation Changes

    The special methods __complex__() for complex and __bytes__() for bytes are implemented to support the typing.SupportsComplex and typing.SupportsBytes protocols. (Contributed by Mark Dickinson and Donghee Na in bpo-24234.)

    siphash13 is added as a new internal hashing algorithm. It has similar security properties as siphash24, but it is slightly faster for long inputs. str, bytes, and some other types now use it as the default algorithm for hash(). PEP 552 hash-based .pyc files now use siphash13 too. (Contributed by Inada Naoki in bpo-29410.)

    When an active exception is re-raised by a raise statement with no parameters, the traceback attached to this exception is now always sys.exc_info()[1].__traceback__. This means that changes made to the traceback in the current except clause are reflected in the re-raised exception. (Contributed by Irit Katriel in bpo-45711.)

    The interpreter state’s representation of handled exceptions (aka exc_info or _PyErr_StackItem) now only has the exc_value field; exc_type and exc_traceback have been removed, as they can be derived from exc_value. (Contributed by Irit Katriel in bpo-45711.)

    A new command line option, AppendPath, has been added for the Windows installer. It behaves similarly to PrependPath, but appends the install and scripts directories instead of prepending them. (Contributed by Bastian Neuburger in bpo-44934.)

    The PyConfig.module_search_paths_set field must now be set to 1 for initialization to use PyConfig.module_search_paths to initialize sys.path. Otherwise, initialization will recalculate the path and replace any values added to module_search_paths.

    The output of the --help option now fits in 50 lines/80 columns. Information about Python environment variables and -X options is now available using the respective --help-env and --help-xoptions flags, and with the new --help-all. (Contributed by Éric Araujo in bpo-46142.)

    Converting between int and str in bases other than 2 (binary), 4, 8 (octal), 16 (hexadecimal), or 32 such as base 10 (decimal) now raises a ValueError if the number of digits in string form is above a limit to avoid potential denial of service attacks due to the algorithmic complexity. This is a mitigation for CVE-2020-10735. This limit can be configured or disabled by environment variable, command line flag, or sys APIs. See the integer string conversion length limitation documentation. The default limit is 4300 digits in string form.

New Modules

    tomllib: For parsing TOML. See PEP 680 for more details. (Contributed by Taneli Hukkinen in bpo-40059.)

    wsgiref.types: WSGI-specific types for static type checking. (Contributed by Sebastian Rittau in bpo-42012.)

Improved Modules
asyncio

    Added the TaskGroup class, an asynchronous context manager holding a group of tasks that will wait for all of them upon exit. For new code this is recommended over using create_task() and gather() directly. (Contributed by Yury Selivanov and others in gh-90908.)

    Added timeout(), an asynchronous context manager for setting a timeout on asynchronous operations. For new code this is recommended over using wait_for() directly. (Contributed by Andrew Svetlov in gh-90927.)

    Added the Runner class, which exposes the machinery used by run(). (Contributed by Andrew Svetlov in gh-91218.)

    Added the Barrier class to the synchronization primitives in the asyncio library, and the related BrokenBarrierError exception. (Contributed by Yves Duprat and Andrew Svetlov in gh-87518.)

    Added keyword argument all_errors to asyncio.loop.create_connection() so that multiple connection errors can be raised as an ExceptionGroup.

    Added the asyncio.StreamWriter.start_tls() method for upgrading existing stream-based connections to TLS. (Contributed by Ian Good in bpo-34975.)

    Added raw datagram socket functions to the event loop: sock_sendto(), sock_recvfrom() and sock_recvfrom_into(). These have implementations in SelectorEventLoop and ProactorEventLoop. (Contributed by Alex Grönholm in bpo-46805.)

    Added cancelling() and uncancel() methods to Task. These are primarily intended for internal use, notably by TaskGroup.

contextlib

    Added non parallel-safe chdir() context manager to change the current working directory and then restore it on exit. Simple wrapper around chdir(). (Contributed by Filipe Laíns in bpo-25625)

dataclasses

    Change field default mutability check, allowing only defaults which are hashable instead of any object which is not an instance of dict, list or set. (Contributed by Eric V. Smith in bpo-44674.)

datetime

    Add datetime.UTC, a convenience alias for datetime.timezone.utc. (Contributed by Kabir Kwatra in gh-91973.)

    datetime.date.fromisoformat(), datetime.time.fromisoformat() and datetime.datetime.fromisoformat() can now be used to parse most ISO 8601 formats (barring only those that support fractional hours and minutes). (Contributed by Paul Ganssle in gh-80010.)

enum

    Renamed EnumMeta to EnumType (EnumMeta kept as an alias).

    Added StrEnum, with members that can be used as (and must be) strings.

    Added ReprEnum, which only modifies the __repr__() of members while returning their literal values (rather than names) for __str__() and __format__() (used by str(), format() and f-strings).

    Changed Enum.__format__() (the default for format(), str.format() and f-strings) to always produce the same result as Enum.__str__(): for enums inheriting from ReprEnum it will be the member’s value; for all other enums it will be the enum and member name (e.g. Color.RED).

    Added a new boundary class parameter to Flag enums and the FlagBoundary enum with its options, to control how to handle out-of-range flag values.

    Added the verify() enum decorator and the EnumCheck enum with its options, to check enum classes against several specific constraints.

    Added the member() and nonmember() decorators, to ensure the decorated object is/is not converted to an enum member.

    Added the property() decorator, which works like property() except for enums. Use this instead of types.DynamicClassAttribute().

    Added the global_enum() enum decorator, which adjusts __repr__() and __str__() to show values as members of their module rather than the enum class. For example, 're.ASCII' for the ASCII member of re.RegexFlag rather than 'RegexFlag.ASCII'.

    Enhanced Flag to support len(), iteration and in/not in on its members. For example, the following now works: len(AFlag(3)) == 2 and list(AFlag(3)) == (AFlag.ONE, AFlag.TWO)

    Changed Enum and Flag so that members are now defined before __init_subclass__() is called; dir() now includes methods, etc., from mixed-in data types.

    Changed Flag to only consider primary values (power of two) canonical while composite values (3, 6, 10, etc.) are considered aliases; inverted flags are coerced to their positive equivalent.

fcntl

    On FreeBSD, the F_DUP2FD and F_DUP2FD_CLOEXEC flags respectively are supported, the former equals to dup2 usage while the latter set the FD_CLOEXEC flag in addition.

fractions

    Support PEP 515-style initialization of Fraction from string. (Contributed by Sergey B Kirpichev in bpo-44258.)

    Fraction now implements an __int__ method, so that an isinstance(some_fraction, typing.SupportsInt) check passes. (Contributed by Mark Dickinson in bpo-44547.)

functools

    functools.singledispatch() now supports types.UnionType and typing.Union as annotations to the dispatch argument.:
    >>>

from functools import singledispatch

@singledispatch

def fun(arg, verbose=False):

    if verbose:

        print("Let me just say,", end=" ")

    print(arg)


@fun.register

def _(arg: int | float, verbose=False):

    if verbose:

        print("Strength in numbers, eh?", end=" ")

    print(arg)


from typing import Union

@fun.register

def _(arg: Union[list, set], verbose=False):

    if verbose:

        print("Enumerate this:")

    for i, elem in enumerate(arg):

        print(i, elem)


    (Contributed by Yurii Karabas in bpo-46014.)

gzip

    The gzip.compress() function is now faster when used with the mtime=0 argument as it delegates the compression entirely to a single zlib.compress() operation. There is one side effect of this change: The gzip file header contains an “OS” byte in its header. That was traditionally always set to a value of 255 representing “unknown” by the gzip module. Now, when using compress() with mtime=0, it may be set to a different value by the underlying zlib C library Python was linked against. (See gh-112346 for details on the side effect.)

hashlib

    hashlib.blake2b() and hashlib.blake2s() now prefer libb2 over Python’s vendored copy. (Contributed by Christian Heimes in bpo-47095.)

    The internal _sha3 module with SHA3 and SHAKE algorithms now uses tiny_sha3 instead of the Keccak Code Package to reduce code and binary size. The hashlib module prefers optimized SHA3 and SHAKE implementations from OpenSSL. The change affects only installations without OpenSSL support. (Contributed by Christian Heimes in bpo-47098.)

    Add hashlib.file_digest(), a helper function for efficient hashing of files or file-like objects. (Contributed by Christian Heimes in gh-89313.)

IDLE and idlelib

    Apply syntax highlighting to .pyi files. (Contributed by Alex Waygood and Terry Jan Reedy in bpo-45447.)

    Include prompts when saving Shell with inputs and outputs. (Contributed by Terry Jan Reedy in gh-95191.)

inspect

    Add getmembers_static() to return all members without triggering dynamic lookup via the descriptor protocol. (Contributed by Weipeng Hong in bpo-30533.)

    Add ismethodwrapper() for checking if the type of an object is a MethodWrapperType. (Contributed by Hakan Çelik in bpo-29418.)

    Change the frame-related functions in the inspect module to return new FrameInfo and Traceback class instances (backwards compatible with the previous named tuple-like interfaces) that includes the extended PEP 657 position information (end line number, column and end column). The affected functions are:

        inspect.getframeinfo()

        inspect.getouterframes()

        inspect.getinnerframes(),

        inspect.stack()

        inspect.trace()

    (Contributed by Pablo Galindo in gh-88116.)

locale

    Add locale.getencoding() to get the current locale encoding. It is similar to locale.getpreferredencoding(False) but ignores the Python UTF-8 Mode.

logging

    Added getLevelNamesMapping() to return a mapping from logging level names (e.g. 'CRITICAL') to the values of their corresponding Logging Levels (e.g. 50, by default). (Contributed by Andrei Kulakovin in gh-88024.)

    Added a createSocket() method to SysLogHandler, to match SocketHandler.createSocket(). It is called automatically during handler initialization and when emitting an event, if there is no active socket. (Contributed by Kirill Pinchuk in gh-88457.)

math

    Add math.exp2(): return 2 raised to the power of x. (Contributed by Gideon Mitchell in bpo-45917.)

    Add math.cbrt(): return the cube root of x. (Contributed by Ajith Ramachandran in bpo-44357.)

    The behaviour of two math.pow() corner cases was changed, for consistency with the IEEE 754 specification. The operations math.pow(0.0, -math.inf) and math.pow(-0.0, -math.inf) now return inf. Previously they raised ValueError. (Contributed by Mark Dickinson in bpo-44339.)

    The math.nan value is now always available. (Contributed by Victor Stinner in bpo-46917.)

operator

    A new function operator.call has been added, such that operator.call(obj, *args, **kwargs) == obj(*args, **kwargs). (Contributed by Antony Lee in bpo-44019.)

os

    On Windows, os.urandom() now uses BCryptGenRandom(), instead of CryptGenRandom() which is deprecated. (Contributed by Donghee Na in bpo-44611.)

pathlib

    glob() and rglob() return only directories if pattern ends with a pathname components separator: sep or altsep. (Contributed by Eisuke Kawasima in bpo-22276 and bpo-33392.)

re

    Atomic grouping ((?>...)) and possessive quantifiers (*+, ++, ?+, {m,n}+) are now supported in regular expressions. (Contributed by Jeffrey C. Jacobs and Serhiy Storchaka in bpo-433030.)

shutil

    Add optional parameter dir_fd in shutil.rmtree(). (Contributed by Serhiy Storchaka in bpo-46245.)

socket

    Add CAN Socket support for NetBSD. (Contributed by Thomas Klausner in bpo-30512.)

    create_connection() has an option to raise, in case of failure to connect, an ExceptionGroup containing all errors instead of only raising the last error. (Contributed by Irit Katriel in bpo-29980.)

sqlite3

    You can now disable the authorizer by passing None to set_authorizer(). (Contributed by Erlend E. Aasland in bpo-44491.)

    Collation name create_collation() can now contain any Unicode character. Collation names with invalid characters now raise UnicodeEncodeError instead of sqlite3.ProgrammingError. (Contributed by Erlend E. Aasland in bpo-44688.)

    sqlite3 exceptions now include the SQLite extended error code as sqlite_errorcode and the SQLite error name as sqlite_errorname. (Contributed by Aviv Palivoda, Daniel Shahaf, and Erlend E. Aasland in bpo-16379 and bpo-24139.)

    Add setlimit() and getlimit() to sqlite3.Connection for setting and getting SQLite limits by connection basis. (Contributed by Erlend E. Aasland in bpo-45243.)

    sqlite3 now sets sqlite3.threadsafety based on the default threading mode the underlying SQLite library has been compiled with. (Contributed by Erlend E. Aasland in bpo-45613.)

    sqlite3 C callbacks now use unraisable exceptions if callback tracebacks are enabled. Users can now register an unraisable hook handler to improve their debug experience. (Contributed by Erlend E. Aasland in bpo-45828.)

    Fetch across rollback no longer raises InterfaceError. Instead we leave it to the SQLite library to handle these cases. (Contributed by Erlend E. Aasland in bpo-44092.)

    Add serialize() and deserialize() to sqlite3.Connection for serializing and deserializing databases. (Contributed by Erlend E. Aasland in bpo-41930.)

    Add create_window_function() to sqlite3.Connection for creating aggregate window functions. (Contributed by Erlend E. Aasland in bpo-34916.)

    Add blobopen() to sqlite3.Connection. sqlite3.Blob allows incremental I/O operations on blobs. (Contributed by Aviv Palivoda and Erlend E. Aasland in bpo-24905.)

string

    Add get_identifiers() and is_valid() to string.Template, which respectively return all valid placeholders, and whether any invalid placeholders are present. (Contributed by Ben Kehoe in gh-90465.)

sys

    sys.exc_info() now derives the type and traceback fields from the value (the exception instance), so when an exception is modified while it is being handled, the changes are reflected in the results of subsequent calls to exc_info(). (Contributed by Irit Katriel in bpo-45711.)

    Add sys.exception() which returns the active exception instance (equivalent to sys.exc_info()[1]). (Contributed by Irit Katriel in bpo-46328.)

    Add the sys.flags.safe_path flag. (Contributed by Victor Stinner in gh-57684.)

sysconfig

    Three new installation schemes (posix_venv, nt_venv and venv) were added and are used when Python creates new virtual environments or when it is running from a virtual environment. The first two schemes (posix_venv and nt_venv) are OS-specific for non-Windows and Windows, the venv is essentially an alias to one of them according to the OS Python runs on. This is useful for downstream distributors who modify sysconfig.get_preferred_scheme(). Third party code that creates new virtual environments should use the new venv installation scheme to determine the paths, as does venv. (Contributed by Miro Hrončok in bpo-45413.)

tempfile

    SpooledTemporaryFile objects now fully implement the methods of io.BufferedIOBase or io.TextIOBase (depending on file mode). This lets them work correctly with APIs that expect file-like objects, such as compression modules. (Contributed by Carey Metcalfe in gh-70363.)

threading

    On Unix, if the sem_clockwait() function is available in the C library (glibc 2.30 and newer), the threading.Lock.acquire() method now uses the monotonic clock (time.CLOCK_MONOTONIC) for the timeout, rather than using the system clock (time.CLOCK_REALTIME), to not be affected by system clock changes. (Contributed by Victor Stinner in bpo-41710.)

time

    On Unix, time.sleep() now uses the clock_nanosleep() or nanosleep() function, if available, which has a resolution of 1 nanosecond (10-9 seconds), rather than using select() which has a resolution of 1 microsecond (10-6 seconds). (Contributed by Benjamin Szőke and Victor Stinner in bpo-21302.)

    On Windows 8.1 and newer, time.sleep() now uses a waitable timer based on high-resolution timers which has a resolution of 100 nanoseconds (10-7 seconds). Previously, it had a resolution of 1 millisecond (10-3 seconds). (Contributed by Benjamin Szőke, Donghee Na, Eryk Sun and Victor Stinner in bpo-21302 and bpo-45429.)

tkinter

    Added method info_patchlevel() which returns the exact version of the Tcl library as a named tuple similar to sys.version_info. (Contributed by Serhiy Storchaka in gh-91827.)

traceback

    Add traceback.StackSummary.format_frame_summary() to allow users to override which frames appear in the traceback, and how they are formatted. (Contributed by Ammar Askar in bpo-44569.)

    Add traceback.TracebackException.print(), which prints the formatted TracebackException instance to a file. (Contributed by Irit Katriel in bpo-33809.)

typing

For major changes, see New Features Related to Type Hints.

    Add typing.assert_never() and typing.Never. typing.assert_never() is useful for asking a type checker to confirm that a line of code is not reachable. At runtime, it raises an AssertionError. (Contributed by Jelle Zijlstra in gh-90633.)

    Add typing.reveal_type(). This is useful for asking a type checker what type it has inferred for a given expression. At runtime it prints the type of the received value. (Contributed by Jelle Zijlstra in gh-90572.)

    Add typing.assert_type(). This is useful for asking a type checker to confirm that the type it has inferred for a given expression matches the given type. At runtime it simply returns the received value. (Contributed by Jelle Zijlstra in gh-90638.)

    typing.TypedDict types can now be generic. (Contributed by Samodya Abeysiriwardane in gh-89026.)

    NamedTuple types can now be generic. (Contributed by Serhiy Storchaka in bpo-43923.)

    Allow subclassing of typing.Any. This is useful for avoiding type checker errors related to highly dynamic class, such as mocks. (Contributed by Shantanu Jain in gh-91154.)

    The typing.final() decorator now sets the __final__ attributed on the decorated object. (Contributed by Jelle Zijlstra in gh-90500.)

    The typing.get_overloads() function can be used for introspecting the overloads of a function. typing.clear_overloads() can be used to clear all registered overloads of a function. (Contributed by Jelle Zijlstra in gh-89263.)

    The __init__() method of Protocol subclasses is now preserved. (Contributed by Adrian Garcia Badarasco in gh-88970.)

    The representation of empty tuple types (Tuple[()]) is simplified. This affects introspection, e.g. get_args(Tuple[()]) now evaluates to () instead of ((),). (Contributed by Serhiy Storchaka in gh-91137.)

    Loosen runtime requirements for type annotations by removing the callable check in the private typing._type_check function. (Contributed by Gregory Beauregard in gh-90802.)

    typing.get_type_hints() now supports evaluating strings as forward references in PEP 585 generic aliases. (Contributed by Niklas Rosenstein in gh-85542.)

    typing.get_type_hints() no longer adds Optional to parameters with None as a default. (Contributed by Nikita Sobolev in gh-90353.)

    typing.get_type_hints() now supports evaluating bare stringified ClassVar annotations. (Contributed by Gregory Beauregard in gh-90711.)

    typing.no_type_check() no longer modifies external classes and functions. It also now correctly marks classmethods as not to be type checked. (Contributed by Nikita Sobolev in gh-90729.)

unicodedata

    The Unicode database has been updated to version 14.0.0. (Contributed by Benjamin Peterson in bpo-45190).

unittest

    Added methods enterContext() and enterClassContext() of class TestCase, method enterAsyncContext() of class IsolatedAsyncioTestCase and function unittest.enterModuleContext(). (Contributed by Serhiy Storchaka in bpo-45046.)

venv

    When new Python virtual environments are created, the venv sysconfig installation scheme is used to determine the paths inside the environment. When Python runs in a virtual environment, the same installation scheme is the default. That means that downstream distributors can change the default sysconfig install scheme without changing behavior of virtual environments. Third party code that also creates new virtual environments should do the same. (Contributed by Miro Hrončok in bpo-45413.)

warnings

    warnings.catch_warnings() now accepts arguments for warnings.simplefilter(), providing a more concise way to locally ignore warnings or convert them to errors. (Contributed by Zac Hatfield-Dodds in bpo-47074.)

zipfile

    Added support for specifying member name encoding for reading metadata in a ZipFile’s directory and file headers. (Contributed by Stephen J. Turnbull and Serhiy Storchaka in bpo-28080.)

    Added ZipFile.mkdir() for creating new directories inside ZIP archives. (Contributed by Sam Ezeh in gh-49083.)

    Added stem, suffix and suffixes to zipfile.Path. (Contributed by Miguel Brito in gh-88261.)

Optimizations

This section covers specific optimizations independent of the Faster CPython project, which is covered in its own section.

    The compiler now optimizes simple printf-style % formatting on string literals containing only the format codes %s, %r and %a and makes it as fast as a corresponding f-string expression. (Contributed by Serhiy Storchaka in bpo-28307.)

    Integer division (//) is better tuned for optimization by compilers. It is now around 20% faster on x86-64 when dividing an int by a value smaller than 2**30. (Contributed by Gregory P. Smith and Tim Peters in gh-90564.)

    sum() is now nearly 30% faster for integers smaller than 2**30. (Contributed by Stefan Behnel in gh-68264.)

    Resizing lists is streamlined for the common case, speeding up list.append() by ≈15% and simple list comprehensions by up to 20-30% (Contributed by Dennis Sweeney in gh-91165.)

    Dictionaries don’t store hash values when all keys are Unicode objects, decreasing dict size. For example, sys.getsizeof(dict.fromkeys("abcdefg")) is reduced from 352 bytes to 272 bytes (23% smaller) on 64-bit platforms. (Contributed by Inada Naoki in bpo-46845.)

    Using asyncio.DatagramProtocol is now orders of magnitude faster when transferring large files over UDP, with speeds over 100 times higher for a ≈60 MiB file. (Contributed by msoxzw in gh-91487.)

    math functions comb() and perm() are now ≈10 times faster for large arguments (with a larger speedup for larger k). (Contributed by Serhiy Storchaka in bpo-37295.)

    The statistics functions mean(), variance() and stdev() now consume iterators in one pass rather than converting them to a list first. This is twice as fast and can save substantial memory. (Contributed by Raymond Hettinger in gh-90415.)

    unicodedata.normalize() now normalizes pure-ASCII strings in constant time. (Contributed by Donghee Na in bpo-44987.)

Faster CPython

CPython 3.11 is an average of 25% faster than CPython 3.10 as measured with the pyperformance benchmark suite, when compiled with GCC on Ubuntu Linux. Depending on your workload, the overall speedup could be 10-60%.

This project focuses on two major areas in Python: Faster Startup and Faster Runtime. Optimizations not covered by this project are listed separately under Optimizations.
Faster Startup
Frozen imports / Static code objects

Python caches bytecode in the __pycache__ directory to speed up module loading.

Previously in 3.10, Python module execution looked like this:

Read __pycache__ -> Unmarshal -> Heap allocated code object -> Evaluate

In Python 3.11, the core modules essential for Python startup are “frozen”. This means that their Code Objects (and bytecode) are statically allocated by the interpreter. This reduces the steps in module execution process to:

Statically allocated code object -> Evaluate

Interpreter startup is now 10-15% faster in Python 3.11. This has a big impact for short-running programs using Python.

(Contributed by Eric Snow, Guido van Rossum and Kumar Aditya in many issues.)
Faster Runtime
Cheaper, lazy Python frames

Python frames, holding execution information, are created whenever Python calls a Python function. The following are new frame optimizations:

    Streamlined the frame creation process.

    Avoided memory allocation by generously re-using frame space on the C stack.

    Streamlined the internal frame struct to contain only essential information. Frames previously held extra debugging and memory management information.

Old-style frame objects are now created only when requested by debuggers or by Python introspection functions such as sys._getframe() and inspect.currentframe(). For most user code, no frame objects are created at all. As a result, nearly all Python functions calls have sped up significantly. We measured a 3-7% speedup in pyperformance.

(Contributed by Mark Shannon in bpo-44590.)
Inlined Python function calls

During a Python function call, Python will call an evaluating C function to interpret that function’s code. This effectively limits pure Python recursion to what’s safe for the C stack.

In 3.11, when CPython detects Python code calling another Python function, it sets up a new frame, and “jumps” to the new code inside the new frame. This avoids calling the C interpreting function altogether.

Most Python function calls now consume no C stack space, speeding them up. In simple recursive functions like fibonacci or factorial, we observed a 1.7x speedup. This also means recursive functions can recurse significantly deeper (if the user increases the recursion limit with sys.setrecursionlimit()). We measured a 1-3% improvement in pyperformance.

(Contributed by Pablo Galindo and Mark Shannon in bpo-45256.)
PEP 659: Specializing Adaptive Interpreter

PEP 659 is one of the key parts of the Faster CPython project. The general idea is that while Python is a dynamic language, most code has regions where objects and types rarely change. This concept is known as type stability.

At runtime, Python will try to look for common patterns and type stability in the executing code. Python will then replace the current operation with a more specialized one. This specialized operation uses fast paths available only to those use cases/types, which generally outperform their generic counterparts. This also brings in another concept called inline caching, where Python caches the results of expensive operations directly in the bytecode.

The specializer will also combine certain common instruction pairs into one superinstruction, reducing the overhead during execution.

Python will only specialize when it sees code that is “hot” (executed multiple times). This prevents Python from wasting time on run-once code. Python can also de-specialize when code is too dynamic or when the use changes. Specialization is attempted periodically, and specialization attempts are not too expensive, allowing specialization to adapt to new circumstances.
