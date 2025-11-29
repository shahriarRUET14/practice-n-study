# Common Interview Questions

## Quick Reference

---

## Spring Boot Fundamentals

**Q: What is Spring Boot?**
- Opinionated framework for building production-ready Spring applications
- Auto-configuration, embedded servers, production-ready features

**Q: Difference between @Component, @Service, @Repository, @Controller?**
- All are stereotypes, functionally similar
- Semantic differences for clarity and AOP

---

## Dependency Injection

**Q: What is Dependency Injection?**
- Design pattern where objects receive dependencies from external source
- Promotes loose coupling and testability

**Q: Constructor vs Field Injection?**
- Constructor: Recommended, immutable, testable
- Field: Not recommended, harder to test

---

## Transaction Management

**Q: @Transactional propagation types?**
- REQUIRED (default), REQUIRES_NEW, NESTED, SUPPORTS, etc.

---

## Code Review Guidelines

**Q: What do you focus on during code review?**

**Answer:**
"I focus on 5 critical areas:

**1. Correctness & Logic**
- Business logic matches requirements
- Edge cases and failure scenarios handled
- No test/experimental code left behind

**2. Security**
- No hardcoded credentials/secrets
- Input validation and sanitization
- No sensitive data in logs/responses
- Secure patterns for auth and DB access

**3. Performance**
- No N+1 queries
- Avoid expensive operations in loops
- Proper caching where needed
- Check API/DB bottlenecks

**4. Readability & Maintainability**
- Clear naming and single-purpose functions
- Follows team patterns and conventions
- Separation of concerns (Controller → Service → Repository)
- Remove dead code and unused imports

**5. Error Handling**
- Meaningful error messages
- Consistent error formats
- No silent failures
- Proper exception flow

**Additional Checks:**
- **Testing**: Critical logic has unit tests, meaningful test coverage
- **Logging**: Correct log levels, no sensitive data logged
- **Documentation**: Complex logic documented, API docs updated
- **Architecture**: Proper structure, avoids tight coupling

**Review Approach:**
- Constructive and respectful feedback
- Focus on code quality, not the person
- Provide clear reasons for changes
- Approve when code is good (not perfect)"

**Key Focus Areas:**
1. ✅ Correctness & Logic
2. ✅ Security
3. ✅ Performance
4. ✅ Readability & Maintainability
5. ✅ Error Handling

---

