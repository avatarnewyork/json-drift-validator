# Contributing to json-drift-validator

Thank you for your interest in contributing!

This project is intended to be simple, reliable, and easy to maintain.  
Please follow the guidelines below when proposing changes.

---

## How to Contribute

1. Fork the repository.
2. Create a new feature branch:
   git checkout -b feature/my-change
3. Make your changes in that branch.
4. Test your changes locally (see "Testing" below).
5. Submit a Pull Request using the provided PR template.

---

## Testing


1. Use the "act" tool to simulate GitHub Actions:
   brew install act
   act -j validate

Please confirm your change behaves the same locally and on GitHub.

---

## Coding Guidelines

- Keep Bash scripts simple and POSIX-friendly when possible.
- Use jq consistently for JSON operations.
- Avoid adding large dependencies or slow steps.
- Comment any non-obvious logic.
- Do not include secrets or sensitive JSON in commits or PRs.

---

## Documentation

If your change affects user-facing behavior:
- Update README.md
- Update any relevant examples in the example/ folder

---

## Pull Requests

Before submitting a PR:
- Ensure all tests pass.
- Follow the PR template.
- Keep the scope focused (one logical change per PR).

---

## Security

Do NOT include sensitive data such as API keys or production JSON in PRs.  
If you discover a security issue, follow SECURITY.md and report it privately.

---

## Questions or Help

If you have questions before contributing, open an issue using the "Question" template.

Thank you again for contributing!
