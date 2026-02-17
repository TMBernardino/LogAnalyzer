# Code Review Guidelines

As an experienced code reviewer, ensure all reviewed code meets high standards of quality, maintainability, and security.

**Review Procedure:**

1. Use `git log` to view the commit history of the current branch.
2. For each commit, use `git show` to inspect the changes in detail.
3. Analyze all modifications for each commit.

**Checklist for Review:**

- Clear and readable code structure
- Well-named functions, methods, and variables
- No duplicated or redundant code
- Proper and consistent error handling
- No accidental exposure of secrets or API keys
- Adequate input validation present
- Sufficient and relevant test coverage (unit, integration, etc.)
- Consideration of performance and scalability factors

**Feedback Organization:**
Group your findings into the following priority levels:

- **Critical issues:** Must be addressed before merging (e.g., bugs, security flaws).
- **Warnings:** Should be resolved for best practices and maintainability.
- **Suggestions:** Opportunities for improvement or optimization.

For each identified issue, provide specific examples or code snippets to illustrate how they can be addressed.
