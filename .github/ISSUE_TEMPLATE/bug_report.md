name: Bug Report
description: Report a bug in the OLTP simulator
title: "[BUG] "
labels: ["bug"]
assignees: []

body:
  - type: markdown
    attributes:
      value: |
        Thanks for reporting a bug! Please fill in the details below.

  - type: input
    id: version
    attributes:
      label: Version
      description: What version of alimentador-bd are you using?
      placeholder: "1.0.0"
    validations:
      required: true

  - type: input
    id: python_version
    attributes:
      label: Python Version
      description: What Python version are you running?
      placeholder: "3.11.0"
    validations:
      required: true

  - type: input
    id: postgres_version
    attributes:
      label: PostgreSQL Version
      description: What PostgreSQL version are you running?
      placeholder: "16.1"
    validations:
      required: true

  - type: textarea
    id: description
    attributes:
      label: Description
      description: Describe the bug clearly
      placeholder: "The stream.py crashes when..."
    validations:
      required: true

  - type: textarea
    id: steps
    attributes:
      label: Steps to Reproduce
      description: How can we reproduce this bug?
      value: |
        1. Configure `.env` with...
        2. Run `make init`
        3. Run `make seed`
        4. Run `make stream`
    validations:
      required: true

  - type: textarea
    id: expected
    attributes:
      label: Expected Behavior
      description: What should happen?
      placeholder: "The stream should insert 1 record every 2 seconds"
    validations:
      required: true

  - type: textarea
    id: actual
    attributes:
      label: Actual Behavior
      description: What actually happens?
      placeholder: "The stream crashes with IntegrityError"
    validations:
      required: true

  - type: textarea
    id: logs
    attributes:
      label: Error Logs
      description: Paste relevant logs (use code blocks)
      render: text
      placeholder: |
        ```
        ERROR scripts.stream [stream_loop] - IntegrityError: duplicate key...
        ```

  - type: textarea
    id: env
    attributes:
      label: Environment Details
      description: Additional environment info (OS, Docker, etc.)
      placeholder: |
        - OS: Ubuntu 22.04
        - Docker: Yes / No
        - PostgreSQL: Local / RDS
