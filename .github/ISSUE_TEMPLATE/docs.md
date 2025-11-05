name: Documentation Issue
description: Report missing or incorrect documentation
title: "[DOCS] "
labels: ["documentation"]
assignees: []

body:
  - type: markdown
    attributes:
      value: |
        Help us improve our documentation!

  - type: input
    id: document
    attributes:
      label: Document
      description: Which document has the issue?
      placeholder: "README.md, ARCHITECTURE.md, etc."
    validations:
      required: true

  - type: textarea
    id: issue
    attributes:
      label: Issue Description
      description: What's wrong or missing?
      placeholder: "The CLI commands section doesn't explain the --batch-size parameter"
    validations:
      required: true

  - type: textarea
    id: suggestion
    attributes:
      label: Suggested Fix
      description: How should it be documented?
