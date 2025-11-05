name: Feature Request
description: Suggest a new feature
title: "[FEATURE] "
labels: ["enhancement"]
assignees: []

body:
  - type: markdown
    attributes:
      value: |
        Great! Share your feature idea below.

  - type: textarea
    id: problem
    attributes:
      label: Problem Statement
      description: What problem does this feature solve?
      placeholder: "Currently, the stream only inserts/updates. We need DELETE operations..."
    validations:
      required: true

  - type: textarea
    id: solution
    attributes:
      label: Proposed Solution
      description: How should this feature work?
      placeholder: |
        Add DELETE operations with low probability (5%) to simulate record removals.
        This would help test Debezium's DELETE capture.
    validations:
      required: true

  - type: textarea
    id: alternatives
    attributes:
      label: Alternative Approaches
      description: Any other ways to achieve this?

  - type: textarea
    id: use_cases
    attributes:
      label: Use Cases
      description: What scenarios does this enable?
      placeholder: |
        - Testing full CDC sync (including deletes)
        - Validating tombstone records in Kafka
