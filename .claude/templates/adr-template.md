# ADR [0000]: [Short Noun Phrase Describing the Decision]

**Status:** [Proposed | Accepted | Rejected | Deprecated | Superseded]  
**Date:** [YYYY-MM-DD]  
**Author:** [Agent Name / Your Name]  
**Domain Context:** [e.g., internal/ordering, internal/fulfillment]

## 1. Context and Problem Statement

[Describe the context and problem here.]

## 2. Decision Drivers

* [Driver 1]
* [Driver 2]
* [Driver 3]

## 3. Considered Options

* **Option 1:** [Name of technology/pattern]
* **Option 2:** [Name of technology/pattern]
* **Option 3:** [Name of technology/pattern]

## 4. Decision Outcome

**Chosen Option:** [Option Name]

**Justification:** [Explain why this option was chosen over the others. Focus on how it solves the specific problem while adhering to our Go architectural standards.]

### 4a. Positive Consequences
* [e.g., Simplifies the internal/billing package]
* [e.g., Removes the need for a third-party dependency]

### 4b. Negative Consequences (Trade-offs)
* [e.g., Requires manual transaction management]
* [e.g., Increases latency by 5ms, which is acceptable for this domain]

## 5. Implementation Notes (Go Specific)

```go
// Example implementation boundary for the chosen option
package example

// ...