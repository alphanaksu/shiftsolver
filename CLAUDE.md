# ShiftSolver — instructions for Claude

## Who you're working with
- Owner: Industrial Engineering student, new to programming. Explain decisions in plain language.
- After each task, add 3–5 lines to LEARNING.md: what changed, the concept behind it,
  and one interview question I should be able to answer about it.

## What we're building
Staff-scheduling optimizer: demand per hour + staff availability -> cheapest weekly roster that
meets coverage and labor rules. Cafe mode and call-center mode (Erlang C). Spec: docs/SPEC.md.

## Stack
Python 3, OR-Tools (CP-SAT), Streamlit, pandas, Plotly, openpyxl, pytest.
Deployed on Streamlit Community Cloud from the main branch.

## Commands
- Install: pip install -r requirements.txt   (run at the start of every session)
- Tests:   pytest -q
- Smoke:   streamlit run app.py --server.headless true   (start, check no errors, stop)

## Layout
- app.py: Streamlit UI only, no model logic
- shiftsolver/requirements.py: demand -> staff needed (incl. Erlang C)
- shiftsolver/model.py: CP-SAT model
- shiftsolver/validate.py: checks a roster against every rule, independent of the solver
- shiftsolver/baseline.py: greedy heuristic for comparison
- data/: synthetic sample CSVs    tests/: pytest

## Rules
- For anything touching more than one file, show a plan first.
- One feature per session. Keep diffs small and readable.
- Every new constraint gets a validator check and a test.
- Ask before adding a dependency; pin versions in requirements.txt.
- No secrets and no real personal data; sample data is synthetic.
- If something is ambiguous, ask me instead of guessing.
- End every session by updating PROGRESS.md: done, next, open questions.

## Definition of done
Tests pass, the app starts without errors, PROGRESS.md and LEARNING.md updated.
