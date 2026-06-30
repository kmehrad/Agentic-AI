# Running Notebooks From the Terminal

This directory contains tutorial notebooks that can be run either interactively
in Jupyter or non-interactively from the terminal with `nbconvert`.

## Prepare the Environment

From the project root, install dependencies with `uv`:

```bash
uv sync
```

If a notebook imports a package directly, that package should be listed as a
project dependency. For example, if a notebook imports `pydantic` directly:

```bash
uv add pydantic
```

This updates both `pyproject.toml` and `uv.lock`.

## Register the Project Kernel

`uv run jupyter nbconvert` runs the `jupyter` command from the project
environment, but the notebook itself may still use a different Jupyter kernel.
Register a kernel that points at this repository's `.venv`:

```bash
uv run python -m ipykernel install --user \
  --name tutorial-agentic-ai \
  --display-name "Python (tutorial-agentic-ai)"
```

Then execute notebooks with that kernel explicitly:

```bash
uv run jupyter nbconvert \
  --to markdown \
  --execute notebooks/exercises_langgraph_complex.ipynb \
  --stdout \
  --ExecutePreprocessor.kernel_name=tutorial-agentic-ai
```

## Load `.env` From the Project Root

When a notebook is run interactively, the current working directory may be the
project root. When it is executed with `nbconvert`, the current working
directory may differ. Avoid assuming `Path.cwd()` is the project root.

Use a root-discovery helper based on `pyproject.toml`:

```python
from pathlib import Path
from dotenv import load_dotenv


def find_project_root(start: Path | None = None) -> Path:
    current = (start or Path.cwd()).resolve()

    for path in [current, *current.parents]:
        if (path / "pyproject.toml").exists():
            return path

    raise FileNotFoundError("Could not find project root containing pyproject.toml")


project_root = find_project_root()
print(f"project_root: {project_root}")

load_dotenv(project_root / ".env")
```

This works when the notebook starts from either the project root or the
`notebooks/` directory.

## Avoid Interactive Input During `nbconvert`

`nbconvert --execute` cannot answer live prompts from `input()`. A notebook that
contains a prompt such as this can run interactively in Jupyter, but fails under
terminal execution:

```python
user_input = input("Enter message: ")
```

Use a helper that prompts normally in Jupyter and falls back to scripted
messages when stdin is unavailable:

```python
import json
import os


DEFAULT_MESSAGES = [
    "hi",
    "how is it going?",
    "I am Kam",
    "what's my name?",
    "/quit",
]


def iter_user_messages(default_messages: list[str] = DEFAULT_MESSAGES):
    while True:
        try:
            user_input = input("Enter message: ")
        except Exception as exc:
            if exc.__class__.__name__ != "StdinNotImplementedError":
                raise

            print("No interactive stdin available; using scripted messages.")
            messages_json = os.getenv("NOTEBOOK_MESSAGES")
            messages = json.loads(messages_json) if messages_json else default_messages

            for message in messages:
                if message not in {"/exit", "/quit"}:
                    print("Human:", message)
                    yield message
            return

        if user_input in {"/exit", "/quit"}:
            return

        yield user_input
```

Then use the helper in the notebook loop:

```python
for user_input in iter_user_messages():
    print("Human:", user_input)
    response = graph.invoke({"messages": user_input}, config=config)
    print("AI:", response["messages"][-1])
```

When running from the terminal, override the scripted messages with an
environment variable:

```bash
NOTEBOOK_MESSAGES='["hi", "I am Kam", "what is my name?"]' \
uv run jupyter nbconvert \
  --to markdown \
  --execute notebooks/exercises_langgraph_complex.ipynb \
  --stdout \
  --ExecutePreprocessor.kernel_name=tutorial-agentic-ai
```

The practical rule is that any notebook intended for `nbconvert --execute`
should provide a non-interactive path for prompts, credentials, approvals, or
other runtime input.
