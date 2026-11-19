Code Review Guide
=================

> **Goal:** Encourage useful code review even when the reviewer is not fully familiar with the codebase, business domain, or feature being implemented.

Code review does not require the reviewer to know everything about the change.
A reviewer may not always be able to fully verify functional correctness. That should not prevent the review from happening.
The reviewer should focus on what they **can reasonably assess** and make the limits of the review clear.

* * *

1. Before Requesting Review
---------------------------

The author should make the change understandable to someone who may not already know the feature.
A pull request should briefly explain:
| Question | Expected Information |
| --- | --- |
| **What changed?** | Short description of the change |
| **Why?** | Problem, feature, bug, or requirement |
| **How was it checked?** | Manual checks, automated checks, or other validation performed |
| **Anything important?** | Risks, assumptions, or unusual design decisions |
The author should also:
*   Review their own changes.
    
*   Remove unrelated or debug code.
    
*   Ensure the code builds.
    
*   Perform reasonable validation of the change.
    
*   Keep the pull request focused where practical.
    
Good pull request context makes meaningful review possible even for someone unfamiliar with the feature.

* * *

2. Reviewer Familiarity
-----------------------

A reviewer does **not** need to be an expert in the entire codebase or feature.
A reviewer unfamiliar with the feature can still review:
*   Code clarity
    
*   Maintainability
    
*   Structure and design
    
*   Naming
    
*   Error handling
    
*   Security concerns
    
*   Performance concerns
    
*   Consistency with surrounding code
    
*   Obvious defects or unexpected behaviour
    
Where useful, the reviewer should inspect surrounding code to better understand the change.

> **Key principle:** Lack of domain knowledge should limit the scope of the review, not prevent the review entirely.

* * *

3. Correctness Review
---------------------

Verifying functional correctness is valuable and should be done when the reviewer has enough knowledge to do so.
However, this may not always be possible.
The reviewer should not imply that they have validated behaviour they do not understand.
When full correctness cannot be verified, the reviewer can still:
*   Check whether the implementation appears consistent with the pull request description.
    
*   Look for obvious logical errors.
    
*   Check error and edge-case handling.
    
*   Verify that assumptions are reasonably clear.
    
*   Ask questions where behaviour is unclear.
    
If correctness depends heavily on business or domain knowledge, another reviewer with that knowledge may be useful where available.

> **Correctness review is recommended, but not required for a review to be useful.**

* * *

4. What to Review
-----------------

### Clarity and Maintainability

*   Is the code understandable?
    
*   Are names meaningful?
    
*   Is the control flow easy to follow?
    
*   Is unnecessary complexity being introduced?
    
*   Would another developer be able to maintain it?
    

### Design

*   Does the change fit the surrounding code?
    
*   Does it duplicate existing functionality?
    
*   Are responsibilities placed in sensible components?
    
*   Is the solution more complicated than necessary?
    

### Error Handling

*   Are failures handled sensibly?
    
*   Are null, invalid, or unexpected inputs considered?
    
*   Are errors silently ignored?
    

### Security

Look for obvious concerns involving:
*   Authorization
    
*   Input validation
    
*   Sensitive data
    
*   External input
    
*   File handling
    
*   Database queries
    
*   Secrets
    
Security-sensitive changes may require additional specialist review.

### Performance

Look for obvious issues such as:
*   Repeated database queries
    
*   Unnecessary network calls
    
*   Expensive operations inside loops
    
*   Excessive data loading
    

### Consistency

Check whether the change follows existing:
*   Project patterns
    
*   Naming conventions
    
*   Error-handling approaches
    
*   API conventions
    
*   Architectural boundaries
    

### Tests and Validation

Where tests exist, they can provide useful context about expected behaviour.
Reviewers may consider:
*   Whether existing tests still make sense.
    
*   Whether new tests help clarify the intended behaviour.
    
*   Whether the author has described how the change was validated.
    
Tests are useful, but they are **not required for every review** under this guideline.

* * *

5. Review More Than the Diff
----------------------------

The diff may not provide enough context.
Where useful, inspect:
*   Surrounding code
    
*   Callers and dependencies
    
*   Existing implementations
    
*   Related validation or tests
    
*   Existing project patterns
    
The goal is not to understand the entire system.
The goal is to understand enough context to make useful observations about the change.

* * *

6. State the Scope of the Review
--------------------------------

Reviewers should be comfortable stating what they did and did not verify.
For example:

> Reviewed for code quality, structure, maintainability, and obvious issues. I am not familiar enough with this feature to fully validate the business logic.

This is preferable to either:
*   avoiding the review completely; or
    
*   approving the change while implying that everything has been verified.
    
A review can still be valuable even when its scope is limited.

* * *

7. Review Comments
------------------

Use simple labels where useful.

### Blocking

An issue that should be resolved before merging.

> **Blocking:** This value can be null here and may cause an exception.

### Suggestion

A recommended improvement that does not necessarily block the change.

> **Suggestion:** This logic may be easier to maintain if extracted into a separate method.

### Question

Used when the reviewer needs clarification.

> **Question:** What happens when this request returns no results?

### Nit

A minor readability or style issue.

> **Nit:** `customerIds` may be clearer than `ids`.

Nits should normally not block a merge.

* * *

8. Avoid Personal Style Debates
-------------------------------

Do not block a change simply because:

> "I would write this differently."

> "I prefer this syntax."

If a style rule matters, document it or automate it through formatting and linting.
Review effort should focus primarily on:
**Clarity · Maintainability · Design · Risk · Security · Correctness where possible**

* * *

9. Approval
-----------

Approval means:

> Based on the scope of my review, I do not see a reason this change should be blocked.

It does **not** necessarily mean:

> I have verified every aspect of the feature and guarantee that the implementation is functionally correct.

A reviewer should approve when:
*   The areas they were able to assess are acceptable.
    
*   No blocking issues remain.
    
*   The code is reasonably maintainable.
    
*   Any limitations of the review are understood.
    

* * *

10. When Additional Review Is Useful
------------------------------------

Consider involving another reviewer when the change depends heavily on knowledge the current reviewer does not have.
Examples:
*   Complex business rules
    
*   Authentication and authorization
    
*   Security-sensitive logic
    
*   Database migrations
    
*   Major architectural changes
    
*   Public API changes
    
*   Critical infrastructure
    
*   Financial logic
    
Multiple reviewers may cover different aspects of the same change.

* * *

Quick Review Checklist
----------------------

Even if you do not know the feature well, ask:
*    Can I understand what the change is trying to do?
    
*    Is the code clear and maintainable?
    
*    Does the structure make sense?
    
*    Are obvious edge cases handled?
    
*    Are errors handled appropriately?
    
*    Are there obvious security concerns?
    
*    Are there obvious performance issues?
    
*    Does the change fit the surrounding code?
    
*    Have I made clear anything I could not confidently verify?
    

> Core Principle
> --------------
> 
> **A useful code review does not require complete knowledge of the codebase or feature.**
> Review what you can assess confidently, ask questions where needed, and be clear about what you could not verify.

* * *

References
----------

1.  **Google Engineering Practices — Code Review**  
    [https://google.github.io/eng-practices/review/](https://google.github.io/eng-practices/review/)
    
2.  **Google — What to Look For in a Code Review**  
    [https://github.com/google/eng-practices/blob/master/review/reviewer/looking-for.md](https://github.com/google/eng-practices/blob/master/review/reviewer/looking-for.md)
    
3.  **GitLab — Code Review Guidelines**  
    [https://docs.gitlab.com/development/code_review/](https://docs.gitlab.com/development/code_review/)
    
4.  **GitHub — Reviewing Proposed Changes in a Pull Request**  
    [https://docs.github.com/en/pull-requests/how-tos/review-pull-requests/reviewing-proposed-changes-in-a-pull-request](https://docs.github.com/en/pull-requests/how-tos/review-pull-requests/reviewing-proposed-changes-in-a-pull-request)