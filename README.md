# Two AI Reviewers Walk Into a Pull Request and the Feedback Loop That Keeps A.I. From @$#%& Up Your Codebase

Dan Morgan - DevOps Days Boston 2026

This project is an accompaniment to the talk of the same title. 

### C+P Prompt
```txt
Use Claude Sonnet and ChatGPT 3.5 Codex to review the following PR <PR # HERE>

Take previous comments (including resolved statuses), responses and explanations into consideration before making recommendations. Validate any explanations for veracity or potential excuses

Generate a **Copy+Paste** Github URL link to the code where issues are flagged. Use Line highlighting e.g #L10-L35 where applicable

Produce C+P code remediation recommendations

Assert:

1. Take special care to note syntax best practices e.g. casing
2. Python3 best practices are being followed e.g. pathLib vs os.path
3. Tests are testing valid scenarios
4. Tests are properly testing error handling
5. Classes and methods are properly documented with DOCSTRINGS highlighting arguments and returns where applicable

Use Claude Opus to check the same PR for security issues. In addition to default practices use OWASP threat modeling and OWASP Top 10 as sources

Categorize and organize issues found by severity. 
```
