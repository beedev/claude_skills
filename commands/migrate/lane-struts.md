# Lane Knowledge: Struts → Spring Boot

Framework-specific migration guidance for Apache Struts 1.x/2.x to Spring Boot REST.

## Core Transform Table

| Struts Concept | Spring Boot Equivalent | Notes |
|---------------|------------------------|-------|
| `Action` class | `@RestController` class | Remove `ActionSupport` extends |
| `execute()` method | `@RequestMapping` / `@PostMapping` method | Map ActionForm to `@RequestBody` or `@ModelAttribute` |
| `ActionForm` subclass | `@ModelAttribute` DTO / `@RequestBody` POJO | Replace `reset()` with constructor defaults |
| `struts.xml` action path | `@RequestMapping(path="/...")` on controller | Path extracted from `<action path="...">` element |
| `ActionMapping` parameter | Encoded in `@RequestMapping` method+path | `method` attr maps to HTTP verb |
| `ActionForward` return | `ResponseEntity<T>` or `ModelAndView` | Success/failure paths → distinct endpoints or status codes |
| `DispatchAction` | Multiple `@RequestMapping` methods | Each dispatch method becomes a separate handler |
| `ActionErrors` / `ActionMessages` | `BindingResult` + `@Valid` + `FieldError` | Validation moves to Bean Validation annotations |
| `request.setAttribute(...)` | Model parameter in `@RequestMapping` | Or `@ModelAttribute` return |
| `session.setAttribute(...)` | `HttpSession` parameter injection | Or `@SessionAttributes` on controller |
| `request.getParameter(...)` | `@RequestParam` parameter | Type-safe, no casting needed |

## JSP Dispatch Decision Tree

```
ActionForward forward = mapping.findForward("success")
  → if forward is a redirect: return ResponseEntity with Location header
  → if forward renders a view: return ModelAndView("viewName", model)
  → if REST endpoint (no view): return ResponseEntity<DTO>.ok(dto)
```

## DI Container Wiring

Struts actions are NOT Spring beans by default. The migration must:
1. Add `@Service` or `@Component` to service classes that actions called directly
2. Add `@Autowired` constructor injection in the new controller
3. Register services in the Spring context (detected automatically via `@ComponentScan`)

Anti-pattern to fix: Struts thread-unsafe singletons (actions instantiated per-request by Struts but shared state via static fields or Struts plugin singletons).
Fix: Mark shared state beans as `@Scope("prototype")` or make them stateless.

## ActionMapping → @RequestMapping Extraction

```xml
<!-- Original struts.xml -->
<action path="/login" type="com.example.LoginAction" name="loginForm"
        scope="request" validate="true" input="/login.jsp">
    <forward name="success" path="/dashboard.jsp"/>
    <forward name="failure" path="/login.jsp"/>
</action>
```

Becomes:
```java
@RestController
@RequestMapping("/login")
public class LoginController {
    @PostMapping
    public ResponseEntity<LoginResponse> login(
            @Valid @RequestBody LoginRequest request,
            BindingResult errors) {
        if (errors.hasErrors()) {
            return ResponseEntity.badRequest().body(/* error dto */);
        }
        // ... business logic
        return ResponseEntity.ok(new LoginResponse(/* data */));
    }
}
```

## Context7 Lookup Map

When implementing, use Context7 to look up:
- `Spring Boot REST controllers` — `@RestController`, `@RequestMapping`, `@GetMapping`, `@PostMapping`
- `Spring Boot service layer` — `@Service`, `@Transactional`, `@Component`
- `Spring Boot dependency injection` — constructor injection, `@Autowired`, `@Bean` configuration
- `Spring Boot validation` — `@Valid`, `javax.validation` / `jakarta.validation` annotations
- `Spring Boot exception handling` — `@ExceptionHandler`, `@ControllerAdvice`

## Quality Gate Checklist

Before approving the transform phase, verify:
- [ ] All `<action>` paths in struts.xml are mapped to `@RequestMapping` annotations
- [ ] All ActionForms replaced with validated DTOs (`@NotNull`, `@Size`, etc.)
- [ ] Spring DI container properly wired (no direct `new ServiceClass()` calls)
- [ ] No action-specific mutable state in controllers (controllers must be stateless)
- [ ] `ActionForward` redirects converted to proper HTTP redirects or status codes
- [ ] Session usage replaced with `@SessionAttributes` or stateless JWT patterns
- [ ] Validation errors surfaced via `BindingResult` or `@ExceptionHandler`
- [ ] All external resource references (DB, services) injected via constructor or `@Autowired`

## Common Pitfalls

1. **Static state in Actions**: Struts actions are thread-safe because Struts creates a new instance per request. If you missed static fields holding request-scoped data, those become race conditions in Spring singletons.
2. **Tiles + ForwardAction patterns**: These require mapping to Thymeleaf or separate `/view` endpoints rather than pure REST.
3. **Struts 2 interceptors**: Map to Spring `HandlerInterceptor` or `Filter` beans — do not omit these.
4. **Locale resolution**: Struts `LocaleAction` → Spring `LocaleChangeInterceptor` + `LocaleResolver` bean.
