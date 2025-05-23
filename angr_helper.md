## ANGR Helper

Below are essential commands and techniques for working with `angr`, including symbolic execution, state management, and constraint solving. Refer to the [official angr documentation](https://docs.angr.io/examples) and [angr YouTube tutorials](https://www.youtube.com/playlist?list=PL-nPhof8EyrGKytps3g582KNiJyIAOtBG) for more examples.

### Basic Setup

Start by importing the necessary libraries and defining the program to analyze:

```python
import angr
import logging
import claripy
import sys

PROGRAM = "./<program_name>"
```

### Adjust Logging

To debug where execution gets stuck, set the logging level to `INFO`:

```python
logging.getLogger("angr").setLevel(logging.INFO)
```

### Debug with IPython

```python
def hook(l=None):
    if l:
        locals().update(l)
    import IPython
    IPython.embed(banner1='', exit_msg='', confirm_exit=False)
    exit(0)
```

### Create a Project

Load the binary into an `angr` project. Use `auto_load_libs=False` to avoid loading shared libraries if not needed ([details](https://docs.angr.io/built-in-analyses/cfg#shared-libraries)):

```python
proj = angr.Project(PROGRAM)
proj = angr.Project(PROGRAM, load_options={"auto_load_libs": False})
```

### Symbolic Input

Create a symbolic variable for user input (e.g., 32 bytes):

```python
USER_DATA_LENGTH = 32
user_data = claripy.BVS("user_data", USER_DATA_LENGTH * 8)
```

### Initialize State

Set up the initial state for simulation, either via `stdin` or `argv`. Use options like `LAZY_SOLVES` or `BYPASS_UNSUPPORTED_SYSCALL` for optimization ([options reference](https://docs.angr.io/appendix/options#options)):

```python
# stdin
initial_state = proj.factory.entry_state(stdin=user_data)

# argv
initial_state = proj.factory.entry_state(args=[PROGRAM, user_data])
initial_state = proj.factory.entry_state(
    args=[PROGRAM, user_data],
    add_options={angr.options.LAZY_SOLVES, "BYPASS_UNSUPPORTED_SYSCALL"}
)
```

### Callable Functions

To analyze a specific function, find its address and create a callable:

```python
start_func = proj.loader.find_symbol("win_func").rebased_addr
func = proj.factory.callable(start_func)
func(<args>)
initial_state = func.result_state
```

### Input Constraints

Add constraints to ensure valid input. For example, restrict input to printable characters or specific values:

```python
import string

# Ensure input is printable
for i in range(USER_DATA_LENGTH):
    initial_state.solver.add(
        claripy.Or(*(
            user_data.get_byte(i) == x
            for x in string.printable.encode('utf-8')
        ))
    )

# Specific character constraint (e.g., starts with 'C')
initial_state.solver.add(user_data.chop(8)[0] == 'C')

# No NULL bytes (unless null-terminated strings are expected)
for byte in user_data.chop(8):
    initial_state.solver.add(byte != 0)
```

### Buffer Size

Increase the symbolic buffer size if more input bytes are needed (default is 60):

```python
byte_length = 64
if byte_length >= 60:
    initial_state.libc.buf_symbolic_bytes = byte_length + 1
```

### Simulation Manager

The simulation manager tracks states during symbolic execution, allowing pruning and exploration ([details](https://docs.angr.io/core-concepts/analyses#simulation-managers)):

```python
sm = proj.factory.simulation_manager(initial_state)
sm = proj.factory.simulation_manager(initial_state, veritesting=True)
```

### Exploration Strategies

Explore the binary to find desired states (e.g., a "win" function or specific output). Use `find` and `avoid` to guide execution:

```python
# Basic exploration
sm.run()

# Explore to a specific function address
flag_locate = proj.loader.find_symbol("win_func").rebased_addr
sm.explore(find=flag_locate, avoid=0x00000)

# Explore by address
sm.explore(find=0x400830, avoid=0x400850)

# Explore based on output (e.g., "Success!" in stdout)
sm.explore(
    find=lambda s: b"Success!" in s.posix.dumps(1),
    avoid=lambda s: b"Failure!" in s.posix.dumps(1),
    step_func=lambda lsm: lsm.drop(stash='avoid')
)

# Break exploration into stages
sm.explore(find=0x4016A3).unstash(from_stash='found', to_stash='active')
sm.explore(find=0x4016B7, avoid=[0x4017D6, 0x401699, 0x40167D]).unstash(from_stash='found', to_stash='active')
sm.explore(find=0x4017CF, avoid=[0x4017D6, 0x401699, 0x40167D]).unstash(from_stash='found', to_stash='active')
sm.explore(find=0x401825, avoid=[0x401811])
```

### Custom State Checkers

These functions are useful when the program's success or failure is indicated by specific messages:

```python
def is_success(state):
    return b"<success-message>" in state.posix.dumps(sys.stdout.fileno())

def is_fail(state):
    return b"<failure-message>" in state.posix.dumps(sys.stdout.fileno())
```

### Extract Results

Once a solution is found, extract and print the input:

```python
if sm.found:
    found = sm.found[0]
    solution = found.solver.eval(user_data, cast_to=bytes)
    print(f"Solution: {solution}")

    # Inspect stdin/stdout
    print(f"Stdin: {found.posix.stdin.concretize()}")
    print(f"Stdout: {found.posix.stdout.concretize()}")
```

### Optimization Tips

For faster execution, enable `unicorn` and `veritesting`:

- **Unicorn** does the math just like normal instead of trying all possible values. Then check the condition. After that, based on the condition will go back to specify the value for symbolic variable. ***(USE ONLY WHEN HAVING SYMBOLIC VARIABLE)***
- **Lazy Solve** explores more paths faster before stopping to solve.
- **Veritesting** works based on Program Counter. After the **if** or **for**, they land in the same Program Counter, then they will be merged.

```python
init_state = proj.factory.entry_state(add_options={angr.options.UNICORN, angr.options.LAZY_SOLVES})
simulation = proj.factory.simgr(init_state, veritesting=True)
```