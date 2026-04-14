# Review Lens Criteria

Stack-aware evaluation criteria for the Faster-Scale-App (Next.js App Router + TypeScript + Tailwind + MongoDB + shadcn/ui).

---

## UX Lens

### Mobile Responsiveness
- All layouts must work on 320px-428px viewport widths (mobile-first)
- No horizontal scrolling on mobile
- Text remains readable without zooming
- Forms are usable on mobile (no tiny inputs)
- Modals/dialogs fit within mobile viewport

### Touch Targets
- Interactive elements: minimum 44x44px tap area
- Sufficient spacing between tap targets (8px minimum)
- No hover-only interactions (must work on touch)
- Links and buttons clearly distinguishable

### Accessibility
- Semantic HTML elements (nav, main, section, article, button vs div)
- ARIA labels on icon-only buttons
- Keyboard navigation works (Tab, Enter, Escape)
- Focus indicators visible
- Color contrast meets WCAG AA (4.5:1 text, 3:1 large text)
- Screen reader announcements for dynamic content
- Form labels properly associated with inputs

### Loading & Error States
- Loading indicators for async operations
- Skeleton screens or spinners during data fetch
- Error messages are user-friendly (not raw error objects)
- Empty states have helpful messaging
- Optimistic UI where appropriate

### Consistency
- Uses cn() for class merging (from @/lib/utils)
- Follows shadcn/ui component patterns
- Consistent spacing (Tailwind spacing scale)
- Consistent typography (Tailwind text classes)

---

## Security Lens

### Authentication & Authorization
- API routes wrapped with `withAuth()` or `withOptionalAuth()`
- User ID from JWT used for data scoping (not from request body)
- Role-based access enforced server-side
- No client-side-only auth checks

### Input Validation
- All user inputs validated server-side
- MongoDB query parameters sanitized
- No raw user input in database queries
- Email validation accounts for array format (`user.email[0]`)
- String comparisons use `.collation({ locale: 'en', strength: 2 })`

### XSS & Injection
- No `dangerouslySetInnerHTML` without sanitization
- User-generated content properly escaped in rendering
- No eval() or Function() with user input
- URL parameters validated before use

### Data Exposure
- API responses don't leak sensitive fields (password hashes, tokens)
- Pagination enforced on list endpoints
- Error messages don't expose internal details
- Feature-flagged data only returned when flag is enabled

### Token Handling
- JWT tokens stored in httpOnly cookies only
- Token refresh handled correctly
- Expired tokens properly rejected
- Token blacklist checked via `authenticateRequestAsync()`

---

## Performance Lens

### Database Queries
- No N+1 query patterns (fetching in loop vs. batch)
- Queries use `.lean()` when Mongoose documents aren't needed
- Projections limit returned fields when appropriate
- Pagination via `getPaginationParams()` with `.skip()/.limit()`
- `dbConnect()` called before every DB operation
- Indexes exist for frequently queried fields

### React Rendering
- No unnecessary re-renders (check deps arrays in useEffect/useMemo/useCallback)
- Large lists use virtualization or pagination
- Heavy computations memoized appropriately
- State lifted only as high as needed (not global for local state)
- No state updates in render path

### Bundle Size
- Dynamic imports for heavy components (`next/dynamic`)
- No importing entire libraries when tree-shaking is available
- Images optimized (next/image, appropriate formats)
- No duplicate dependencies

### API Efficiency
- Response payloads trimmed to what's needed
- Proper HTTP caching headers where appropriate
- Batch operations where possible
- Middleware chains are efficient (no redundant DB calls)

---

## Test Coverage Lens

### Unit Tests
- New functions/utilities have corresponding `.test.ts` files
- Components with logic have component tests
- API route handlers have route tests
- Edge cases covered (empty input, null, boundary values)

### Mock Quality
- MongoDB mock chains include ALL methods: `.find()`, `.lean()`, `.sort()`, `.limit()`, `.skip()`, `.exec()`, `.collation()`
- Each chained method returns mock object (for chaining), except terminal (.exec()) which resolves
- `.lean()` queries return plain objects, not Mongoose documents
- Mocks don't mask real bugs (overly permissive)

### E2E Coverage
- User-facing features have E2E tests in `app/e2e/tests/`
- Critical user flows covered (login, core features)
- E2E tests use proper selectors (data-testid preferred)
- Tests are independent (no shared state between tests)

### Test Isolation
- No shared mutable state between tests
- Each test sets up and tears down its own data
- No test ordering dependencies
- Async operations properly awaited

---

## Production Readiness Lens

### Error Handling
- try/catch around async operations
- Errors propagated with meaningful context
- API routes return proper HTTP status codes
- Client-side error boundaries for component failures
- Uses `successResponse()`/`errorResponse()` helpers

### Feature Flags
- New features gated behind LaunchDarkly flags
- Feature flag checks on both client and API routes
- Graceful degradation when flag is off
- Flag key follows `faster-scale` project naming

### Logging & Debugging
- Meaningful error logging (not swallowed errors)
- No console.log in production code (use proper logging)
- Enough context in errors to debug issues
- No sensitive data in log output

### Database Safety
- Pre-save hooks not double-lowercasing (hooks already normalize)
- Subscription updates use spread: `{ ...user.subscription, field: value }`
- Model exports use `mongoose.models.X || mongoose.model('X', schema)`
- Migration scripts if schema changes

### Rollback Safety
- Changes are backward compatible where possible
- Database changes don't break existing data
- Feature flags allow disabling new code without deploy
- No destructive operations without confirmation

### Environment
- New env vars documented
- `NEXT_PUBLIC_` prefix for client-visible vars
- No secrets in client-side code
- Env vars use `SCREAMING_SNAKE_CASE`
