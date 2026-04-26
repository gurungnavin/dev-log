# Dev Log: VS Code Python Environment / Package Import Warning

## Date

2026-04-26

## Context

While setting up the FastAPI backend project, we created a local Python virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Then we installed backend dependencies from requirements.txt:

```bash
pip install -r requirements.txt
```

However, VS Code still showed dotted underline warnings on package names inside requirements.txt.

Example warning:

```bash
Package `fastapi` is not installed in the selected environment.
```

## Problem

> The packages were installed correctly in the local .venv, but VS Code/Pylance was using a different Python interpreter.
> So the situation was:

```bash
Terminal Python = .venv Python = packages installed = OK
VS Code Python = different interpreter = packages not found = warning
```

- This caused VS Code to think packages like fastapi, sqlmodel, and sqlalchemy were missing, even though they were already installed.

## Why This Happens

- VS Code does not always automatically select the newly created virtual environment.
For example, the terminal may be using:

```bash
/Users/taimatsu/Documents/square-samurai-test/square-samurai-test-backend/.venv/bin/python
```

But VS Code may still be checking the project with another Python, such as:

```bash
/usr/bin/python3
```

or another global/system Python environment.
That other environment does not have the project packages installed, so VS Code shows import/package warnings.

# How To Confirm
- Run this command in the VS Code terminal:

```bash
which python
```

Expected result:

```bash
/Users/taimatsu/Documents/square-samurai-test/square-samurai-test-backend/.venv/bin/python
```

Also check that FastAPI is installed in the current environment:

```bash
python -m pip show fastapi
```

Expected result should include:

```bash
Name: fastapi
Version: 0.115.12
Location: .../.venv/lib/python...
```

# Solution

Select the correct Python interpreter in VS Code.

1. Steps:

Press:
```bash
Cmd + Shift + P
```


2. Search:

```bash
Python: Select Interpreter
```

3. Select the project virtual environment:
```bash
.venv/bin/python
```

Full example path:
```bash
/Users/taimatsu/Documents/square-samurai-test/square-samurai-test-backend/.venv/bin/python
```

4. Reload VS Code if warnings do not disappear immediately:
```bash
Cmd + Shift + P
Developer: Reload Window
```

## Recommended VS Code Setting
- To prevent this issue in the future, create:
```bash
.vscode/settings.json
```
- Add:
```json
{
  "python.defaultInterpreterPath": ".venv/bin/python",
  "python.terminal.activateEnvironment": true
}
```

## Project structure:
```bash
square-samurai-test-backend/
├─ .venv/
├─ .vscode/
│  └─ settings.json
├─ requirements.txt
└─ app/
```
