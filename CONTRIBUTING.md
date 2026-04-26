# Contribution guide
This guide defines the expected workflow for pull requests in this repository.
## Scope
- Contributions are welcome for bug fixes, documentation improvements, tests, and focused feature work.
- For breaking changes or larger refactors, open an issue first and agree scope before implementation.
## Workflow (step-by-step)
1. Fork this repository.
2. Create a branch using a descriptive name.
3. Keep the change focused on one concern.
4. Update user-facing documentation in parallel with code changes.
5. Run local checks before opening a pull request.
6. Commit with a clear message that describes the change intent.
7. Open a pull request with a concise summary and test evidence.
## Branch naming
- Use one of these prefixes:
	- `feature/<short-topic>`
	- `fix/<short-topic>`
	- `docs/<short-topic>`
	- `chore/<short-topic>`
## Local validation
Run Markdown syntax checks before opening a pull request:
```sh
npx markdownlint-cli2 "**/*.md" "#node_modules"
```
For docs-only changes, state this clearly in the pull request and include the command result.
## Pull request checklist
Before submitting, confirm:
1. The change solves one clear problem.
2. Documentation was updated where relevant.
3. Validation commands were run and results are included.
4. No secrets, credentials, or private keys are included.
5. The pull request description lists affected files and expected behaviour impact.
## Security and secrets
- Never commit secrets, API keys, tokens, private keys, or production credentials.
- Keep example values as placeholders in public files.
- If you discover a security issue, report it privately to the maintainer instead of opening a public exploit description.
## Review process
- Address review comments with focused follow-up commits.
- Keep discussion technical, concise, and evidence-based.
- Mark unresolved risks or assumptions explicitly in the pull request.