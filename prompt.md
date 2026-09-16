# Implement `ask`

Implement a command-line tool called `ask`: a single-file, dependency-free shell
script that answers a short question from the terminal by making one non-interactive
call to an LLM CLI and printing the reply to stdout.

## Input modes

The tool must support exactly three ways of being invoked:

1. `ask <words...>` — the question is the arguments. Standard input must NOT be
   read in this mode under any circumstance, so the tool never blocks when run
   inside a pipeline, a script, or with stdin redirected.
2. `cmd | ask - <words...>` — a lone `-` as the first argument means standard
   input is supplementary *context* (e.g. command output, a log, a file); the
   remaining arguments are the question.
3. `ask` — with no arguments, standard input is the question itself. This works
   both when piped and when typed interactively.

Edge cases that must behave correctly:

- `cmd | ask -` with no question words: the piped text becomes the question, and
  is not sent as context attached to an empty question.
- Interactive invocation with no arguments must print a short hint (to stderr,
  never stdout) explaining how to type the question and how to signal end of input.
- Empty or whitespace-only input: print a one-line usage summary of all three
  modes to stderr and exit with a distinct non-zero status. Do not call the LLM.

## Composing the request

- When context is present, present the question and the context to the model as
  clearly separated parts, with the context visibly delimited so the model treats
  it as quoted material rather than as further instructions.
- Gather a small block of facts about the machine the user is asking from, and
  supply it to the model as background: operating system (specific distribution
  or release where obtainable, including which family it derives from when that
  determines the package manager), CPU architecture, login shell, graphical
  session details when applicable, user and host, current working directory, and
  the current local date and time with timezone. Any fact that cannot be
  determined on the current system must be omitted entirely — never emitted as an
  empty or placeholder value.
- Instruct the model to use that environment block only when it changes the
  answer, and to prefer the local system's package manager and tooling in any
  command it suggests.

## Answer style to request from the model

The system prompt must ask for: short answers; the same language as the question;
correctness prioritised over completeness; the direct answer first; exact commands
or code where useful; only caveats that materially change the answer; explicit
admission of uncertainty rather than invented commands, flags, APIs or facts; no
restating of the question and no filler.

## Model call

Invoke the LLM CLI in one-shot, non-interactive mode. Choose a small, fast, cheap
model at low reasoning effort. Grant it web search and web fetch capability, and
nothing else — no file system or shell access. Do not persist the session. Ensure
the call cannot stall waiting on standard input after the script has already
consumed it.

## Portability and quality constraints

- Must run on the oldest bash shipped by current macOS as well as on modern Linux
  bash, with no extra runtime dependencies beyond the LLM CLI and standard system
  utilities. Where a convenient modern shell feature is unavailable on the old
  shell, degrade gracefully (omit the affected detail) rather than emitting
  corrupted output.
- Enable strict error handling, and make sure no ordinary control-flow condition
  trips it.
- Quote and handle input safely: questions and piped context may contain spaces,
  newlines, quotes, apostrophes, backticks and shell metacharacters, and must
  reach the model intact.
- Diagnostics and prompts go to stderr; only the model's answer goes to stdout,
  so `ask ... > file` and `ask ... | less` work cleanly.
- Add brief comments only where the code does something non-obvious — explaining
  the constraint being worked around, not restating the code.

Verify all three invocation modes plus the error cases before reporting done.
