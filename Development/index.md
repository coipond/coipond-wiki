# Development

Commands for running the test suite and the local quality checks, plus what CI enforces on every push and pull request.

## Local Checks

Run the tests and quality tools locally:

```shell
bundle exec rspec                                 # tests (PostgreSQL required)
BUNDLE_GEMFILE=Gemfile.lint bundle exec rubocop   # lint (StandardRB via RuboCop)
bundle exec brakeman && bundle exec bundler-audit # security
bundle exec yard-lint                             # docs lint
```

## Continuous Integration

CI runs the full suite plus RuboCop, yard-lint, Brakeman, bundler-audit, and a gitleaks secret scan on every push and pull request.

For the security properties these checks back up, see [Security](Security.md).
