# Hands-on Project 1 — Full CI Pipeline for a real Node.js app
This is production-quality — not tutorial code.

## The scenario: You have a Node.js REST API. Build a pipeline that:

- Runs on every PR and push to main
- Lints, tests with coverage, enforces 80% coverage threshold
- Scans for vulnerabilities
- Builds and pushes Docker image (main branch only)
- Uses caching for speed
- Notifies on failure
