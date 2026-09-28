---
description: This prompt is used to review the code quality of a .NET solution and assign maintainability and overall code quality scores.
model: Claude Sonnet 4.6 (copilot)
name: .NET Code Quality Scorecard
agent: agent
---

Review all code in this solution and assign a maintainability and overall code quality score between 1-100.  When calculating the scores please consider the use of patterns, commenting, tight vs. loose coupling, adherence to naming conventions and other generally accepted C# standards and best practices.  Assign scores for each project in the solution and then an overall score for the entire solution. Finally provide recomendations for improving the scores.