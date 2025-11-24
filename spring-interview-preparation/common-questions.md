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

