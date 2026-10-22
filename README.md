# School Community Platform

> **AI-agent implementation specification and project README**
>
> A secure school social network for students, teachers, and administrators, with user profiles, follows, posts, media uploads, likes, comments, reposts, a personalized For You feed, and a trusted Academic announcements feed.
>
> **Primary goal:** build a maintainable application that can be developed for free, run locally with Docker, and be load-tested for a target of 10,000–50,000 registered users. This is a capacity target, not a guarantee that free hosting tiers can sustain every traffic pattern.

---

## 0. Instructions for the AI coding agent

You are the implementation agent for this repository. Follow the requirements below as the source of truth.

### Mandatory working rules

1. **Inspect before changing.** Start by inspecting the repository, existing files, package manager, and Git status. Do not overwrite existing work without understanding it.
2. **Implement in phases.** Follow the numbered implementation phases in this README. Do not jump straight to UI polish before authentication, data integrity, authorization, and tests work.
3. **Keep the project runnable.** At the end of each phase, run relevant formatting, linting, type checking, tests, and builds. Fix failures before proceeding.
4. **Do not invent dependencies or APIs.** Check official package documentation and installed versions before using unfamiliar APIs. Pin compatible versions and commit the lockfile.
5. **Use free/open-source tooling first.** Do not add paid services as a requirement. External free tiers may have quotas, sleep/idle behavior, eligibility rules, and changing terms.
6. **Do not put secrets in Git.** Use `.env.example` with placeholders. Never commit real API keys, database passwords, access tokens, private keys, or production data.
7. **Do not trust the frontend for security.** Every protected operation must be authorized by the backend, including role checks, school membership, object ownership, feed visibility, and media access.
8. **Keep PostgreSQL authoritative.** Do not duplicate authoritative users, follows, likes, comments, roles, or academic announcements in MongoDB or Redis.
9. **Use explicit contracts.** DTOs, API response shapes, error codes, route names, and service interfaces must be documented and tested.
10. **Do not claim a feature is complete without tests.** Add tests for successful behavior, invalid input, unauthorized access, and relevant edge cases.
11. **Prefer a modular monolith initially.** Keep domain boundaries clean so services can be extracted later. Do not deploy every domain as a separate microservice during the first iteration.
12. **Ask only when blocked by a real product decision.** Otherwise follow the defaults in this document, record assumptions in `docs/decisions/`, and continue.
13. **Never use real student data for development or load testing.** Generate synthetic accounts and content.
14. **Keep an implementation log.** Maintain `docs/IMPLEMENTATION_STATUS.md` with completed tasks, commands run, test results, known issues, and the next step.

### Required completion behavior

At the end of each phase, report:
- Files created or changed.
- Important design decisions.
- Commands executed.
- Test/build results.
- Known limitations.
- The next phase to execute.

Do not mark a phase complete if its acceptance criteria fail.

---

## 1. Product scope

### 1.1 User types

The system has three primary roles:

- **Student** — can manage an allowed profile, publish permitted posts, upload approved media, follow users, like, comment, repost, share links, and read authorized academic notices.
- **Teacher** — can use social features and publish academic announcements only within their assigned school/grade/class/subject scope.
- **Admin** — can manage school membership and role assignments, moderate reported content, manage accounts, publish school-wide announcements, and view authorized audit records.

Roles are assigned and changed only through trusted server-side workflows. A user must never be able to elevate their own role by changing a request body or editing browser state.

A user may eventually have multiple role assignments, so implement role assignments as data rather than relying on a single client-supplied role. The API can return a `primaryRole` for display while preserving server-side permissions.

### 1.2 Social features

- Signup, login, logout, email verification, and password reset.
- User profile, avatar, bio, school affiliation, and permitted class/department details.
- Follow and unfollow; list followers and following.
- Create, edit, delete, and view posts.
- Text and image/video attachments.
- Like/unlike posts.
- Comment and reply to comments.
- Repost with attribution to the original author.
- Share a post using a link, subject to its visibility rules.
- Report posts, comments, and accounts.
- Block or mute users.
- Notifications for follows, likes, comments, mentions, reposts, and academic notices.
- Pagination, moderation, abuse prevention, and privacy controls.

### 1.3 Separate feeds

**For You feed**
- Personalized and ranked using recent posts, followed accounts, school/community relevance, and permitted engagement signals.
- Must exclude deleted, removed, suspended-author, blocked, and unauthorized content.
- Must use cursor-based pagination.
- Must not treat official academic notices as ordinary social content.

**Academic feed**
- A separate content domain for official notices, exam schedules, assignment deadlines, timetable changes, and school communications.
- Only authorized teachers and admins can publish.
- Support school/grade/class/subject targeting, priority, pinning, publication time, expiry, and acknowledgement when required.
- Must be durable and auditable.
- Notification delivery failures must not make an announcement disappear from the feed.
- Authorization must be checked on every read and acknowledgement operation.

### 1.4 Administration and safety

- User status management: active, suspended, disabled.
- Content reporting and moderation queues.
- Auditable role changes and administrative actions.
- Rate limits and spam prevention.
- Secure handling of student information and media.
- School-approved privacy, retention, consent, and guardian processes where applicable.
- Review applicable Indian data-protection requirements and current rules before production launch.

---

## 2. Technology stack

Use these technologies unless a documented technical reason requires a change.

| Area | Choice | Purpose |
|---|---|---|
| Frontend | Next.js, React, TypeScript | Responsive web application |
| Styling | Tailwind CSS, shadcn/ui | Consistent UI components |
| Client data fetching | TanStack Query | API requests, caching, mutations |
| Frontend validation | Zod | Form and client-side schema validation |
| Backend | NestJS, TypeScript | REST API and domain modules |
| API contract | OpenAPI/Swagger | API documentation and integration |
| SQL database | PostgreSQL | Source of truth and transactions |
| ORM/migrations | Prisma | Typed database access and migrations |
| Cache | Valkey or Redis-compatible implementation | Cache, rate-limit state, ephemeral data |
| Job processing | Start with a database-backed job/outbox design; add RabbitMQ if needed | Retryable asynchronous work |
| Media | S3-compatible object storage; Cloudflare R2 is an option | Images and videos |
| Image processing | Sharp | Resizing and thumbnails |
| Video processing | FFmpeg worker, added when needed | Transcoding and thumbnails |
| Local infrastructure | Docker Engine and Docker Compose | Reproducible local environment |
| Source control | Git and GitHub Free | Repository and CI |
| Testing | Jest or Vitest, Supertest, Playwright | Unit, API, and browser tests |
| Observability | Structured logs and health checks initially | Debugging and operations |

### 2.1 Database decision

**PostgreSQL is the authoritative database.** Use it for users, school membership, role assignments, follows, posts, comments, likes, reposts, media metadata, academic announcements, notifications, reports, and audit logs.

Do **not** add MongoDB by default. It is optional and should only be introduced if measured workloads show a concrete benefit, such as document-oriented feed projections or flexible event documents. If MongoDB is added later, those documents must be rebuildable projections, not a second source of truth.

Use Valkey/Redis for cache and ephemeral state, not as the only record of a like, follow, announcement, or other durable operation.

### 2.2 Free-resource policy

- Local development should work without any cloud account.
- Docker Compose must run the core application and PostgreSQL locally.
- Use a free-tier hosted PostgreSQL provider for a pilot only after verifying current quotas, inactivity behavior, data processing terms, backups, and eligibility.
- Cloudflare Pages may host a compatible static frontend; a server-rendered Next.js deployment must use a compatible supported runtime.
- Cloudflare R2 may be used for media within its current free allowance.
- Hosted free compute is optional, not assumed to be permanently available or always on.
- Keep the app portable so the API, database, and workers can move to another host.
- Do not store real student data on a hobby/free provider until the school has reviewed its security, privacy, data location, access controls, and terms.

---

## 3. Repository structure

Create this structure. Keep names consistent; if the existing repository differs, document the mapping rather than creating duplicate architectures.

```text
school-platform/
├── apps/
│   ├── web/
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── (public)/
│   │   │   │   │   ├── login/page.tsx
│   │   │   │   │   ├── signup/page.tsx
│   │   │   │   │   └── forgot-password/page.tsx
│   │   │   │   ├── (app)/
│   │   │   │   │   ├── layout.tsx
│   │   │   │   │   ├── for-you/page.tsx
│   │   │   │   │   ├── academic/page.tsx
│   │   │   │   │   ├── notifications/page.tsx
│   │   │   │   │   ├── profile/[userId]/page.tsx
│   │   │   │   │   └── settings/page.tsx
│   │   │   │   ├── admin/
│   │   │   │   ├── layout.tsx
│   │   │   │   ├── page.tsx
│   │   │   │   └── providers.tsx
│   │   │   ├── components/
│   │   │   │   ├── layout/
│   │   │   │   ├── feed/
│   │   │   │   ├── posts/
│   │   │   │   ├── profiles/
│   │   │   │   ├── academic/
│   │   │   │   └── ui/
│   │   │   ├── features/
│   │   │   │   ├── auth/
│   │   │   │   ├── feed/
│   │   │   │   ├── posts/
│   │   │   │   ├── profiles/
│   │   │   │   ├── academic/
│   │   │   │   └── notifications/
│   │   │   ├── lib/
│   │   │   │   ├── api-client.ts
│   │   │   │   ├── auth.ts
│   │   │   │   ├── env.ts
│   │   │   │   └── utils.ts
│   │   │   └── types/
│   │   ├── public/
│   │   ├── Dockerfile
│   │   └── package.json
│   └── api/
│       ├── src/
│       │   ├── main.ts
│       │   ├── app.module.ts
│       │   ├── common/
│       │   │   ├── decorators/
│       │   │   ├── dto/
│       │   │   ├── filters/
│       │   │   ├── guards/
│       │   │   ├── interceptors/
│       │   │   ├── interfaces/
│       │   │   ├── pipes/
│       │   │   ├── policies/
│       │   │   └── utils/
│       │   ├── config/
│       │   ├── database/
│       │   ├── modules/
│       │   │   ├── auth/
│       │   │   ├── users/
│       │   │   ├── schools/
│       │   │   ├── follows/
│       │   │   ├── posts/
│       │   │   ├── comments/
│       │   │   ├── reactions/
│       │   │   ├── feeds/
│       │   │   ├── academic/
│       │   │   ├── media/
│       │   │   ├── notifications/
│       │   │   ├── moderation/
│       │   │   ├── admin/
│       │   │   └── health/
│       │   └── workers/
│       ├── prisma/
│       │   ├── schema.prisma
│       │   ├── migrations/
│       │   └── seed.ts
│       ├── test/
│       ├── Dockerfile
│       └── package.json
├── packages/
│   └── api-contracts/             # Optional shared, framework-neutral types/schemas
├── infrastructure/
│   ├── docker/
│   └── deployment/
├── docs/
│   ├── decisions/
│   ├── API.md
│   ├── DATABASE.md
│   ├── SECURITY.md
│   ├── DEPLOYMENT.md
│   └── IMPLEMENTATION_STATUS.md
├── docker-compose.yml
├── .env.example
├── .gitignore
├── package.json
├── pnpm-workspace.yaml
└── README.md
```

Use a pnpm workspace unless the repository already uses another package manager. Commit the lockfile. Avoid duplicated implementations of the same API client, role logic, database model, or UI component.

---

## 4. Naming and code conventions

### 4.1 General rules

- Files and folders: `kebab-case`.
- Classes, interfaces, enums, and React components: `PascalCase`.
- Functions, methods, variables: `camelCase`.
- Constants: `UPPER_SNAKE_CASE` for true global constants.
- DTO names: `<Action><Entity>Dto`, such as `CreatePostDto`.
- NestJS modules: `<Domain>Module`.
- Controllers: `<Domain>Controller`.
- Services: `<Domain>Service`.
- Repositories, if explicitly created: `<Domain>Repository`.
- Policy classes: `<Domain>Policy`.
- React components: descriptive nouns such as `PostCard`, `FeedList`, `AcademicNoticeCard`.
- Hooks: `use` prefix, such as `useForYouFeed`.
- API routes: plural resource names and kebab-case path segments.
- Database tables and fields: snake_case in PostgreSQL.
- IDs: UUIDs for primary keys.
- Time: UTC timestamps stored as `timestamptz`.

Do not create a generic `CommonService` that accumulates unrelated business logic. Put genuinely shared technical code in `common/`; keep domain behavior in its domain module.

### 4.2 Required shared backend classes

Implement these names, or document an equivalent existing implementation before renaming.

| Class/function | File | Responsibility |
|---|---|---|
| `CurrentUser` decorator | `common/decorators/current-user.decorator.ts` | Read the authenticated principal from the request |
| `Roles` decorator | `common/decorators/roles.decorator.ts` | Declare required roles for a route |
| `JwtAuthGuard` | `common/guards/jwt-auth.guard.ts` | Authenticate protected API requests |
| `RolesGuard` | `common/guards/roles.guard.ts` | Enforce declared role requirements |
| `SchoolMembershipGuard` | `common/guards/school-membership.guard.ts` | Check school membership where a route requires it |
| `RequestIdMiddleware` | `common/` or middleware folder | Attach/propagate a request ID |
| `HttpExceptionFilter` | `common/filters/http-exception.filter.ts` | Return a consistent error shape |
| `ValidationPipe` configuration | `main.ts` | Reject invalid request payloads |
| `PrismaService` | `database/prisma.service.ts` | Manage Prisma client lifecycle |
| `PrismaModule` | `database/prisma.module.ts` | Provide PrismaService to modules |
| `HealthController` | `modules/health/health.controller.ts` | Expose liveness/readiness endpoints |
| `ConfigModule` setup | `config/` and `app.module.ts` | Validate environment configuration |

The role guard is not a substitute for object-level authorization. For example, a student may have permission to delete their own post but not another student's post. Enforce that in the domain service or policy.

### 4.3 Required domain classes and functions

Use these as the expected initial service API. Additional private helper methods are allowed.

**Auth**
- `AuthModule`
- `AuthController`
- `AuthService`
- `PasswordService`
- `TokenService` or a documented identity-provider adapter
- `SignupDto`, `LoginDto`, `RefreshSessionDto`, `ForgotPasswordDto`, `ResetPasswordDto`
- `AuthService.signup(dto, context)`
- `AuthService.login(dto, context)`
- `AuthService.logout(userId, sessionId)`
- `AuthService.refreshSession(dto, context)`
- `AuthService.getCurrentUser(userId)`

**Users**
- `UsersModule`
- `UsersController`
- `UsersService`
- `UpdateProfileDto`
- `UsersService.getProfile(viewer, targetUserId)`
- `UsersService.updateProfile(userId, dto)`
- `UsersService.getFollowers(viewer, targetUserId, cursor, limit)`
- `UsersService.getFollowing(viewer, targetUserId, cursor, limit)`

**Follows**
- `FollowsModule`
- `FollowsController`
- `FollowsService`
- `FollowsService.follow(followerId, targetUserId)`
- `FollowsService.unfollow(followerId, targetUserId)`
- `FollowsService.isFollowing(followerId, targetUserId)`

**Posts**
- `PostsModule`
- `PostsController`
- `PostsService`
- `CreatePostDto`, `UpdatePostDto`
- `PostsService.createPost(authorId, dto)`
- `PostsService.getPost(viewer, postId)`
- `PostsService.updatePost(actor, postId, dto)`
- `PostsService.deletePost(actor, postId)`
- `PostsService.listPosts(viewer, query)`

**Comments**
- `CommentsModule`
- `CommentsController`
- `CommentsService`
- `CreateCommentDto`
- `CommentsService.createComment(authorId, postId, dto)`
- `CommentsService.listComments(viewer, postId, query)`
- `CommentsService.deleteComment(actor, commentId)`

**Reactions**
- `ReactionsModule`
- `ReactionsController`
- `ReactionsService`
- `ReactionsService.likePost(userId, postId)`
- `ReactionsService.unlikePost(userId, postId)`
- `ReactionsService.repostPost(userId, postId)`
- `ReactionsService.removeRepost(userId, postId)`

**Feeds**
- `FeedsModule`
- `FeedsController`
- `FeedsService`
- `FeedRankingService`
- `FeedCursorService`
- `FeedsService.getForYouFeed(viewer, query)`
- `FeedsService.getFollowingFeed(viewer, query)`
- `FeedsService.getAcademicFeed(viewer, query)`
- `FeedRankingService.rankCandidates(viewer, candidates)`

**Academic**
- `AcademicModule`
- `AcademicController`
- `AcademicService`
- `CreateAnnouncementDto`, `UpdateAnnouncementDto`, `AcknowledgeAnnouncementDto`
- `AcademicService.createAnnouncement(actor, dto)`
- `AcademicService.updateAnnouncement(actor, announcementId, dto)`
- `AcademicService.getAnnouncement(viewer, announcementId)`
- `AcademicService.listAnnouncements(viewer, query)`
- `AcademicService.acknowledgeAnnouncement(userId, announcementId)`

**Media**
- `MediaModule`
- `MediaController`
- `MediaService`
- `CreateUploadUrlDto`, `CompleteUploadDto`
- `MediaService.createUploadUrl(actor, dto)`
- `MediaService.completeUpload(actor, assetId, dto)`
- `MediaService.getAsset(viewer, assetId)`

**Notifications**
- `NotificationsModule`
- `NotificationsController`
- `NotificationsService`
- `NotificationsService.createNotification(input)`
- `NotificationsService.listNotifications(userId, query)`
- `NotificationsService.markRead(userId, notificationId)`
- `NotificationWorker.process(job)`

**Moderation and administration**
- `ModerationModule`, `ModerationController`, `ModerationService`
- `AdminModule`, `AdminController`, `AdminService`
- `ModerationService.reportContent(actor, dto)`
- `ModerationService.reviewReport(moderator, reportId, dto)`
- `AdminService.changeUserStatus(admin, targetUserId, dto)`
- `AdminService.assignRole(admin, targetUserId, dto)`
- `AdminService.listAuditLogs(admin, query)`

### 4.4 Dependency direction

Use this dependency direction:

```text
Controller -> Domain Service -> PrismaService / repository
                           -> Policy / authorization helper
                           -> Outbox/event publisher for async work

Worker -> Domain Service or dedicated job handler
Frontend page -> Feature component -> API hook -> API client -> REST endpoint
```

Rules:
- Controllers handle HTTP, DTOs, status codes, and response mapping; they should not contain SQL or complex business logic.
- Domain services enforce business rules and object permissions.
- Prisma queries should live in domain services or a domain-specific repository, not in React components or controllers.
- Domain services must not import controllers.
- Avoid circular module dependencies. If domains need to communicate, use a clearly defined service interface or an asynchronous event.
- Do not make a controller call another controller.
- The frontend must not access PostgreSQL directly.
- The browser must never receive database credentials, service secrets, or private object-storage credentials.
- Avoid creating an event or microservice for every method. Use in-process service calls for synchronous work and durable events for work that can safely happen asynchronously.

---

## 5. API layout and conventions

### 5.1 Base path

All application APIs must be versioned:

```text
/v1
```

Production example:

```text
https://api.example.edu/v1
```

Local example:

```text
http://localhost:3000/v1
```

Configure the prefix once in `apps/api/src/main.ts`. Do not repeat `/v1` inconsistently in every controller.

### 5.2 Endpoint map

#### Authentication

| Method | Endpoint | Access |
|---|---|---|
| `POST` | `/auth/signup` | Public, subject to enrollment policy |
| `POST` | `/auth/login` | Public, rate-limited |
| `POST` | `/auth/logout` | Authenticated |
| `POST` | `/auth/refresh` | Valid refresh session |
| `POST` | `/auth/verify-email` | Valid verification token |
| `POST` | `/auth/forgot-password` | Public, rate-limited, non-enumerating response |
| `POST` | `/auth/reset-password` | Valid reset token |
| `GET` | `/auth/me` | Authenticated |

#### Users and follows

| Method | Endpoint | Access |
|---|---|---|
| `GET` | `/users/:userId` | Authenticated; visibility policy applies |
| `PATCH` | `/users/me` | Authenticated |
| `GET` | `/users/:userId/followers` | Authenticated; visibility policy applies |
| `GET` | `/users/:userId/following` | Authenticated; visibility policy applies |
| `PUT` | `/users/:userId/follow` | Authenticated |
| `DELETE` | `/users/:userId/follow` | Authenticated |
| `PUT` | `/users/:userId/block` | Authenticated |
| `DELETE` | `/users/:userId/block` | Authenticated |
| `POST` | `/users/:userId/report` | Authenticated |

#### Posts, comments, and reactions

| Method | Endpoint | Access |
|---|---|---|
| `POST` | `/posts` | Authenticated |
| `GET` | `/posts/:postId` | Authenticated; visibility policy applies |
| `PATCH` | `/posts/:postId` | Author or authorized moderator |
| `DELETE` | `/posts/:postId` | Author or authorized moderator |
| `GET` | `/posts/:postId/comments` | Authorized viewer |
| `POST` | `/posts/:postId/comments` | Authenticated, permitted viewer |
| `DELETE` | `/comments/:commentId` | Author or authorized moderator |
| `PUT` | `/posts/:postId/like` | Authenticated |
| `DELETE` | `/posts/:postId/like` | Authenticated |
| `PUT` | `/posts/:postId/repost` | Authenticated |
| `DELETE` | `/posts/:postId/repost` | Authenticated |
| `POST` | `/posts/:postId/report` | Authenticated |

Use `PUT` for idempotent state changes like following or liking, with a unique database constraint ensuring duplicate requests cannot create duplicate records.

#### Feeds

| Method | Endpoint | Access |
|---|---|---|
| `GET` | `/feeds/for-you` | Authenticated |
| `GET` | `/feeds/following` | Authenticated |
| `GET` | `/feeds/academic` | Authenticated, membership and audience checks |

Example:

```text
GET /v1/feeds/for-you?limit=20&cursor=<opaque-cursor>
GET /v1/feeds/academic?gradeId=<uuid>&limit=20&cursor=<opaque-cursor>
```

Never trust a `gradeId` from the request by itself. The backend must verify that the requester is allowed to access that grade.

#### Academic announcements

| Method | Endpoint | Access |
|---|---|---|
| `POST` | `/academic/announcements` | Authorized teacher/admin |
| `GET` | `/academic/announcements/:announcementId` | Authorized audience |
| `PATCH` | `/academic/announcements/:announcementId` | Author with scope or admin |
| `POST` | `/academic/announcements/:announcementId/acknowledge` | Authorized audience |
| `GET` | `/academic/announcements/:announcementId/acknowledgements` | Authorized publisher/admin, according to policy |

#### Media and notifications

| Method | Endpoint | Access |
|---|---|---|
| `POST` | `/media/upload-url` | Authenticated |
| `POST` | `/media/:assetId/complete` | Upload owner |
| `GET` | `/media/:assetId` | Authorized viewer |
| `GET` | `/notifications` | Authenticated |
| `PATCH` | `/notifications/:notificationId/read` | Notification owner |

#### Administration

| Method | Endpoint | Access |
|---|---|---|
| `GET` | `/admin/reports` | Moderator/admin |
| `PATCH` | `/admin/reports/:reportId` | Moderator/admin |
| `PATCH` | `/admin/users/:userId/status` | Admin |
| `PUT` | `/admin/users/:userId/roles/:role` | Admin with permission |
| `GET` | `/admin/audit-logs` | Explicitly authorized admin |

Use separate request DTOs and service methods for each endpoint. Do not expose unrestricted generic database update endpoints.

### 5.3 Response format

Success responses should use a consistent JSON structure. For list endpoints:

```json
{
  "data": [],
  "pagination": {
    "nextCursor": null,
    "hasMore": false
  }
}
```

For a post, return a documented response DTO with only fields the current viewer may access. Do not serialize Prisma model objects directly to the client.

Standard error structure:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request is invalid.",
    "requestId": "request-id-value",
    "details": []
  }
}
```

Expected error codes include:
- `VALIDATION_ERROR`
- `UNAUTHENTICATED`
- `FORBIDDEN`
- `RESOURCE_NOT_FOUND`
- `CONFLICT`
- `RATE_LIMITED`
- `UPLOAD_NOT_READY`
- `INTERNAL_ERROR`

Do not return stack traces, SQL details, secrets, or private information in API errors.

### 5.4 Pagination and filters

- Default page size: 20.
- Maximum page size: 50.
- Use opaque cursors based on stable sort fields, such as `(created_at, id)`.
- Validate cursor contents and reject malformed cursors.
- Use deterministic ordering.
- Avoid deep `OFFSET` pagination for feeds.
- Apply authorization filters before returning records.
- Add tests for items created or deleted while a user is paging.

---

## 6. Authentication and authorization design

### 6.1 Authentication defaults

Use an established, maintained authentication library or OIDC-compatible provider where appropriate. If implementing password authentication:
- Hash passwords with Argon2id or a suitably configured established password-hashing library.
- Never store plaintext passwords.
- Rate-limit login and recovery endpoints.
- Return generic password-reset responses so accounts cannot be enumerated.
- Use short-lived access credentials and revocable refresh sessions.
- Rotate refresh credentials on use and detect reuse where supported.
- For browser sessions, prefer secure, HTTP-only cookies with appropriate `SameSite` settings and CSRF protection for cookie-authenticated state changes.
- Require email verification or school-approved invitation/enrollment before enabling school membership.
- Do not allow public signup to select `teacher` or `admin`.

The exact token/session implementation must be documented in `docs/decisions/authentication.md` before it is considered complete.

### 6.2 Role and resource checks

Create a policy layer with explicit checks such as:
- `canViewProfile(viewer, targetUser)`
- `canEditProfile(viewer, targetUser)`
- `canViewPost(viewer, post)`
- `canEditPost(viewer, post)`
- `canDeletePost(viewer, post)`
- `canCommentOnPost(viewer, post)`
- `canPublishAnnouncement(viewer, audience)`
- `canViewAnnouncement(viewer, announcement)`
- `canAcknowledgeAnnouncement(viewer, announcement)`
- `canModerateReport(viewer, report)`
- `canAssignRole(actor, targetUser, role)`

These may be methods on domain policy classes such as `PostsPolicy`, `UsersPolicy`, and `AcademicPolicy`. Keep policy names stable and test them directly.

**Important:** role checks alone are insufficient. A teacher may publish only to assigned classes; an admin role change must be audited; a user cannot access another school's content merely by supplying its UUID.

---

## 7. Database model and constraints

Use Prisma migrations as the source-controlled schema evolution process. Review the generated migration SQL before applying it to production.

### 7.1 Required entities

At minimum, model:

- `User`
- `School`
- `SchoolMembership`
- `Role` or `UserRole`
- `Classroom` or `SchoolGroup`
- `Subject` if subject targeting is required
- `Follow`
- `Block`
- `Post`
- `PostMedia`
- `Like`
- `Comment`
- `Repost`
- `Announcement`
- `AnnouncementAudience`
- `AnnouncementAcknowledgement`
- `MediaAsset`
- `Notification`
- `Report`
- `AuditLog`
- `OutboxEvent`
- `RefreshSession` or equivalent session entity
- `EmailVerificationToken` and `PasswordResetToken`, or secure equivalents

### 7.2 Required integrity rules

- UUID primary keys.
- Foreign keys for relational integrity.
- Unique email or normalized email according to the account policy.
- Unique `(follower_id, following_id)` pair.
- Unique `(user_id, post_id)` for likes.
- Unique `(user_id, post_id)` for reposts.
- Prevent a user from following themselves.
- Prevent duplicate acknowledgements for the same user and announcement.
- Use explicit status enums or constrained values.
- Store timestamps as UTC-aware timestamps.
- Use `deleted_at` or moderation status when retention/audit rules require soft deletion.
- Keep `created_at` immutable; update `updated_at` on applicable edits.
- Add indexes for frequent filters and ordering.
- Do not rely only on denormalized counts for correctness.

### 7.3 Important indexes

Create indexes based on actual query patterns, including:
- `posts(author_id, created_at DESC, id DESC)`
- Partial index for published posts ordered by `created_at DESC, id DESC`
- `comments(post_id, created_at, id)`
- `likes(post_id, user_id)`
- `follows(following_id, follower_id)`
- `announcements(school_id, published_at DESC, id DESC)` as appropriate to the final schema
- Audience lookup indexes for school, class, grade, and subject targeting
- `notifications(user_id, created_at DESC, id DESC)`
- Report indexes for status and creation time
- Unique and foreign-key indexes required by constraints

Do not blindly add every index: verify query plans and write overhead.

### 7.4 Database access

`PrismaService` should extend or wrap Prisma Client, connect during module startup, and disconnect cleanly during shutdown. Inject it through `PrismaModule`.

Use a bounded connection pool. In serverless environments, follow the provider's pooling guidance. Keep transactions short. Do not make external network requests while holding database transactions open.

Use transactions for coupled database changes. Use an outbox record in the same transaction when a successful mutation must reliably publish an asynchronous event.

---

## 8. Feed architecture

### 8.1 Initial implementation

Start with a correct PostgreSQL query and cursor pagination. Do not begin with an elaborate ranking model or distributed fan-out system.

`FeedsService.getForYouFeed(viewer, query)` should:
1. Validate the cursor and bounded `limit`.
2. Fetch candidate posts from permitted sources (followed users and approved school/community content).
3. Exclude content the viewer cannot access.
4. Exclude blocked users and removed content.
5. Apply a deterministic initial ranking based on recency and followed-account relevance.
6. Return a response DTO with `nextCursor` and `hasMore`.
7. Use Redis/Valkey cache only after correctness is established.

`FeedsService.getFollowingFeed(viewer, query)` should return recent eligible posts from followed accounts.

`FeedsService.getAcademicFeed(viewer, query)` must query the academic domain, verify audience membership, and apply priority/publication/expiry rules. It must not call the normal social feed and simply filter on a text label.

### 8.2 Optimization path

When measurements justify it:
- Cache hot feed pages in Redis/Valkey.
- Cache post hydration in batches to avoid N+1 database queries.
- Use a hybrid fan-out model: enqueue feed references for ordinary accounts and merge recent posts at read time for high-follower accounts.
- Use durable outbox events for post-created and post-removed events.
- Treat cached feeds and denormalized projections as rebuildable.
- Recheck visibility and moderation state before returning feed items.
- Set cache TTLs and invalidation behavior explicitly.
- Load-test bursts caused by school-wide announcements.

Do not claim that caching alone guarantees a target latency. Measure p50, p95, and p99 latency under realistic load.

---

## 9. Media upload and storage

### 9.1 Upload flow

1. Browser calls `POST /v1/media/upload-url`.
2. `MediaService.createUploadUrl()` checks authentication, quotas, allowed media type, declared size, and ownership context.
3. The server creates a pending `MediaAsset` record and returns a short-lived signed upload URL and asset ID.
4. Browser uploads directly to object storage.
5. Browser calls `POST /v1/media/:assetId/complete`.
6. `MediaService.completeUpload()` verifies the object exists and checks size and metadata. Never trust the browser's claim alone.
7. A worker scans and processes the file; the asset remains pending/quarantined until approved.
8. The asset becomes ready and can be attached to a post.
9. A cleanup task removes expired pending uploads and orphaned objects according to policy.

### 9.2 Security requirements

- Validate file contents, extension, MIME type, size, and permitted media types.
- Use random object keys; never use user-supplied filenames as storage paths.
- Do not expose storage credentials to the browser.
- Keep private assets private; authorize access or use short-lived signed download URLs.
- Reject uploads that exceed per-user quotas.
- Do not allow unprocessed/quarantined media to render as approved content.
- Use image resizing and video transcoding workers where needed.
- Use CDN caching only for assets whose visibility policy permits it.
- Define retention and deletion behavior for media attached to deleted posts or accounts.

For local development, use a local S3-compatible service only if needed; otherwise implement a storage adapter and test it with a fake/in-memory implementation. Production storage must use a real provider adapter.

---

## 10. Async work, jobs, and notifications

Do not make a post creation request wait for email delivery, video transcoding, or all feed updates.

### 10.1 Event names

Use stable event names:
- `user.created`
- `user.status_changed`
- `post.created`
- `post.updated`
- `post.removed`
- `post.liked`
- `post.commented`
- `post.reposted`
- `follow.created`
- `announcement.published`
- `announcement.updated`
- `media.upload_completed`
- `media.approved`
- `report.created`

### 10.2 Outbox pattern

When a database mutation must trigger asynchronous work:
1. Begin a PostgreSQL transaction.
2. Write the domain record.
3. Write an `OutboxEvent` record in the same transaction.
4. Commit.
5. A worker publishes/processes pending outbox events.
6. Mark events processed only after successful handling.
7. Retry failures with backoff and a maximum attempt count.
8. Move permanently failing jobs to a dead-letter or failed-job state for inspection.

Workers must be idempotent. A repeated `post.created` event must not create duplicate notifications or duplicate feed entries. Use event IDs and unique constraints to prevent duplicate effects.

### 10.3 Notifications

In-app notifications should be stored durably. Email and push delivery can be optional adapters.

Implement:
- `NotificationsService.createNotification(input)`
- `NotificationsService.listNotifications(userId, query)`
- `NotificationsService.markRead(userId, notificationId)`
- `NotificationWorker.process(job)`

Only the notification owner may list or mark a notification read. Avoid revealing private post content in notifications when the recipient is not authorized to view the post.

---

## 11. Frontend implementation contract

### 11.1 Route structure

Implement these pages:
- `/login`
- `/signup`
- `/forgot-password`
- `/for-you`
- `/academic`
- `/notifications`
- `/profile/[userId]`
- `/settings`
- `/admin` and appropriate admin subroutes

Use route groups to separate public pages from authenticated application pages. UI route protection improves user experience but is not a security boundary; the API must authorize every protected operation.

### 11.2 Required reusable components

Create reusable, typed components:
- `AppShell`
- `AppSidebar`
- `MobileNavigation`
- `FeedTabs`
- `FeedList`
- `FeedSkeleton`
- `PostComposer`
- `PostCard`
- `PostMedia`
- `PostActions`
- `CommentList`
- `CommentComposer`
- `UserAvatar`
- `UserProfileHeader`
- `FollowButton`
- `AcademicNoticeCard`
- `AnnouncementComposer`
- `NotificationList`
- `ReportDialog`
- `RoleBadge`
- `EmptyState`
- `ErrorState`
- `ConfirmDialog`

Keep components focused. `PostCard` should render a post and delegate actions to hooks or callbacks; it should not implement network calls inline or contain authorization logic.

### 11.3 API client

Create one API client in `apps/web/src/lib/api-client.ts`. It should:
- Read the configured `NEXT_PUBLIC_API_BASE_URL`.
- Add the appropriate credentials/session behavior.
- Parse JSON and typed errors consistently.
- Handle non-2xx responses through a shared `ApiError`.
- Support `AbortSignal` for cancelled requests.
- Never log tokens, passwords, or sensitive response data.
- Never contain private server secrets.

Use typed feature hooks, such as:
- `useCurrentUser()`
- `useForYouFeed()`
- `useFollowingFeed()`
- `useAcademicFeed(filters)`
- `useCreatePost()`
- `useLikePost()`
- `useComments(postId)`
- `useCreateComment()`
- `useFollowUser(userId)`
- `useNotifications()`
- `useMarkNotificationRead()`

Use TanStack Query for server state and cache invalidation. For example, after a successful like, update the affected post's cached state or invalidate the relevant query. Do not use a full-page reload for ordinary feed interactions.

### 11.4 UI behavior

- Provide loading, empty, success, and error states.
- Disable repeated submit actions while a mutation is pending.
- Use optimistic updates only when rollback behavior is implemented.
- Show clear permission errors rather than silently hiding failed operations.
- Make forms keyboard accessible and label all inputs.
- Add responsive layouts for mobile, tablet, and desktop.
- Use consistent timestamps and handle missing avatars/media.
- Do not expose admin controls to unauthorized users as the only means of protection; the API must also reject the operation.

---

## 12. Docker and local development

### 12.1 Expected local services

The initial `docker-compose.yml` should provide:
- `postgres`
- `valkey` or `redis`
- `api`
- `web` where a containerized frontend is convenient
- Optional worker services when job processing is implemented

Do not add MongoDB unless the decision is documented. RabbitMQ is optional until the durable job workload justifies it; the initial outbox worker can use PostgreSQL.

Use persistent volumes for local database state. Do not expose database ports to the public internet. Bind local development ports to `127.0.0.1` where practical.

### 12.2 Environment variables

Create `.env.example` with placeholder values:

```dotenv
NODE_ENV=development

# Frontend
NEXT_PUBLIC_API_BASE_URL=http://localhost:3000/v1

# Backend
API_PORT=3000
WEB_ORIGIN=http://localhost:3001
DATABASE_URL=postgresql://school_app:change_me@localhost:5432/school
REDIS_URL=redis://localhost:6379

# Authentication: replace with a secure random secret for local use
AUTH_ACCESS_TOKEN_SECRET=replace_with_a_long_random_local_secret
AUTH_ACCESS_TOKEN_TTL_SECONDS=900
AUTH_REFRESH_TOKEN_TTL_SECONDS=2592000

# Media storage adapter
MEDIA_STORAGE_DRIVER=local
MEDIA_MAX_UPLOAD_BYTES=10485760

# Optional object storage; leave blank for local development
S3_ENDPOINT=
S3_REGION=
S3_BUCKET=
S3_ACCESS_KEY_ID=
S3_SECRET_ACCESS_KEY=
S3_PUBLIC_BASE_URL=

# Email adapter
EMAIL_DRIVER=console
EMAIL_FROM=no-reply@example.test

# Rate limiting
RATE_LIMIT_WINDOW_SECONDS=60
RATE_LIMIT_MAX_REQUESTS=100
```

Adjust variable names to the actual implementation and validate them at startup. Do not silently fall back to insecure production defaults. Production secrets must be injected by the deployment environment, not stored in source control.

If browser sessions use cookies, configure cookie security, allowed origins, CSRF protection, and credentialed CORS explicitly. Do not assume bearer tokens and cookie sessions have identical security requirements.

### 12.3 Common local commands

Document the actual commands after configuring the workspace scripts. The intended workflow is:

```bash
# Install dependencies
pnpm install

# Start infrastructure
docker compose up -d postgres valkey

# Generate Prisma client
pnpm --filter api prisma:generate

# Apply local development migrations
pnpm --filter api prisma:migrate:dev

# Seed synthetic development data
pnpm --filter api seed

# Start API and web app
pnpm dev

# Run checks
pnpm lint
pnpm typecheck
pnpm test
pnpm build

# Stop infrastructure without deleting persistent data
docker compose down

# WARNING: this removes local volumes and local database data
docker compose down -v
```

Create matching scripts in the root and package manifests. Do not leave README commands that do not exist in `package.json`.

---

## 13. Step-by-step build order and acceptance criteria

The AI agent must execute the phases in order.

### Phase 0 — Repository discovery and plan

Tasks:
1. Inspect existing files, package manager, runtime versions, and Git status.
2. Identify existing code that can be reused.
3. Create or update this README.
4. Create `docs/IMPLEMENTATION_STATUS.md`.
5. Record architectural decisions in `docs/decisions/`.

Acceptance criteria:
- Existing work is preserved.
- Planned repository structure is documented.
- The next phase and validation commands are recorded.

### Phase 1 — Workspace and tooling

Tasks:
1. Initialize the pnpm workspace.
2. Create Next.js and NestJS apps.
3. Add TypeScript strict mode, ESLint, formatting, and environment validation.
4. Create Dockerfiles and Compose for PostgreSQL and Valkey.
5. Add health endpoints and basic CI.

Acceptance criteria:
- Fresh checkout can install dependencies.
- PostgreSQL and Valkey start with Compose.
- API returns a health response.
- Frontend renders a basic page.
- Lint, typecheck, and build pass.

### Phase 2 — Database and common backend

Tasks:
1. Configure Prisma and PostgreSQL.
2. Add `PrismaService` and `PrismaModule`.
3. Implement migrations for schools, users, memberships, role assignments, and sessions.
4. Add common error handling, validation, request IDs, logging, and response DTO conventions.
5. Add seed data for a synthetic school, student, teacher, and admin.
6. Document the schema in `docs/DATABASE.md`.

Acceptance criteria:
- Migrations run from a clean database.
- Seed is repeatable or fails safely without duplicating records.
- Constraints prevent invalid role and membership states.
- API errors follow the documented shape.

### Phase 3 — Authentication and authorization

Tasks:
1. Implement signup and enrollment policy.
2. Implement login/logout/session handling.
3. Implement email verification and password recovery adapters.
4. Implement `JwtAuthGuard` or the chosen session guard, `CurrentUser`, `Roles`, and `RolesGuard`.
5. Implement school membership and object-level policy checks.
6. Add rate limits to sensitive endpoints.
7. Add tests for privilege escalation and cross-school access.

Acceptance criteria:
- A public user cannot create an admin or teacher account by choosing a role.
- Unauthenticated requests cannot access protected endpoints.
- Users cannot access another user's private data by changing IDs.
- Suspended/disabled accounts are blocked according to policy.
- Passwords and secrets are never returned in API responses.

### Phase 4 — Profiles and social graph

Tasks:
1. Implement profile reads and edits.
2. Implement follow/unfollow with uniqueness constraints.
3. Implement block/mute behavior.
4. Implement followers/following pagination.
5. Build `UserProfileHeader`, `FollowButton`, and profile pages.

Acceptance criteria:
- Duplicate follows are impossible.
- Self-follow is rejected.
- Profile privacy is enforced on API responses.
- Blocked users are handled consistently in feeds and interactions.

### Phase 5 — Posts, comments, likes, and reposts

Tasks:
1. Implement create/read/update/delete post operations.
2. Implement comments and replies.
3. Implement idempotent like/unlike.
4. Implement repost/remove repost with original attribution.
5. Implement shareable post links that do not bypass authorization.
6. Add moderation and report hooks.
7. Build `PostComposer`, `PostCard`, `PostActions`, and comment components.

Acceptance criteria:
- Authors can edit/delete their own content according to policy.
- Other users cannot modify content without permission.
- Duplicate likes/reposts are prevented by database constraints.
- Removed content cannot reappear in feeds.
- API tests cover unauthorized and invalid operations.

### Phase 6 — Media uploads

Tasks:
1. Create `MediaAsset` schema and storage interface.
2. Implement local development storage.
3. Implement signed object-storage URLs behind an adapter.
4. Implement upload completion verification and file limits.
5. Add processing states and a worker interface.
6. Add tests for invalid MIME type, oversize files, expired URLs, and unauthorized asset access.

Acceptance criteria:
- Upload credentials never reach the browser except for short-lived scoped signed URLs.
- A user cannot attach another user's asset.
- Pending or quarantined media is not exposed as approved.
- Missing objects and failed processing produce recoverable errors.

### Phase 7 — Feeds

Tasks:
1. Implement `FeedsService` and cursor encoding/validation.
2. Implement For You and Following feed queries.
3. Implement a separate Academic feed query.
4. Add deterministic ordering and pagination tests.
5. Add Redis/Valkey caching only after baseline correctness.
6. Measure SQL query plans and prevent N+1 queries.

Acceptance criteria:
- Feed responses are stable and bounded.
- Unauthorized, removed, blocked, or suspended-author content is excluded.
- Cursor pagination does not rely on unbounded offsets.
- Academic audience filtering is enforced on the server.
- Tests cover new/deleted posts between pagination requests.

### Phase 8 — Academic announcements

Tasks:
1. Implement announcement creation and editing.
2. Implement grade/class/subject audience targeting.
3. Implement priority, pinning, publish time, and expiry.
4. Implement acknowledgement records if enabled by the school.
5. Add audit logs for publication and edits.
6. Build the Academic feed UI and announcement composer.

Acceptance criteria:
- Students cannot publish official notices by using a normal post endpoint.
- Teachers cannot publish beyond their assigned scope.
- Expired or unpublished notices follow the documented policy.
- Acknowledgement is idempotent and accessible only to authorized users.
- Announcements remain readable even if email/push delivery fails.

### Phase 9 — Notifications and background jobs

Tasks:
1. Implement `OutboxEvent` and transaction-safe event creation.
2. Implement a worker that processes pending outbox records.
3. Implement notification creation, listing, and marking read.
4. Add retry/backoff and failed-job visibility.
5. Add event deduplication/idempotency.
6. Add email adapter; use console output locally.

Acceptance criteria:
- A committed event is not lost simply because a worker is temporarily down.
- Reprocessing an event does not duplicate notifications.
- Only the owner can access or mark a notification read.
- Failed jobs can be retried and are observable.

### Phase 10 — Administration and moderation

Tasks:
1. Implement report submission and report queues.
2. Implement moderator decisions and content status transitions.
3. Implement account suspension/restoration.
4. Implement role assignment with strict permission checks.
5. Implement audit logging for sensitive actions.
6. Build admin pages and confirmation dialogs.

Acceptance criteria:
- Only authorized staff can moderate or assign roles.
- Administrative actions are auditable.
- Users cannot self-promote.
- Reported content follows the documented review process.

### Phase 11 — Frontend completeness and accessibility

Tasks:
1. Complete all routes and responsive navigation.
2. Add loading, empty, error, and success states.
3. Add form validation and accessible labels.
4. Implement feed interactions through typed API hooks.
5. Add optimistic updates only where rollback is implemented.
6. Run Playwright end-to-end tests for core journeys.

Acceptance criteria:
- Core user journeys work on desktop and mobile widths.
- Keyboard navigation and accessible form labels are present.
- Errors are visible and actionable.
- UI visibility is not used as a substitute for API authorization.

### Phase 12 — Security, performance, and deployment

Tasks:
1. Review authentication, authorization, CORS, CSRF, rate limiting, secrets, and upload controls.
2. Add database backups and document restore testing.
3. Add health checks, structured logs, request IDs, and basic metrics.
4. Load-test normal, peak, and announcement-burst scenarios.
5. Optimize slow queries and add only justified indexes.
6. Document free-tier limitations and a migration path.
7. Deploy a pilot only after school/privacy review.

Acceptance criteria:
- All automated tests pass.
- No critical/high-severity known security issue remains unreviewed.
- Database restore procedure is tested.
- Load-test results and environment assumptions are documented.
- No secret or real student data is committed to the repository.

---

## 14. Testing strategy

### Unit tests
Test services and policies, including:
- `AuthService`
- `UsersPolicy`
- `PostsPolicy`
- `AcademicPolicy`
- `FeedCursorService`
- `FeedRankingService`
- `MediaService` validation
- Event idempotency and notification rules

### API integration tests
Use a disposable test database and test:
- Signup/login and session lifecycle.
- Role and membership checks.
- Cross-user and cross-school access.
- Post creation, edit, delete, likes, comments, and reposts.
- Academic audience targeting.
- Media upload completion.
- Notification ownership.
- Admin role assignment and audit records.

### End-to-end tests
Use Playwright for:
1. Student logs in and publishes a post.
2. Another student follows the author and sees eligible content.
3. A student likes and comments on a post.
4. A teacher publishes an announcement to an assigned class.
5. A student in that class sees the announcement.
6. A student outside the audience cannot read the announcement by guessing its ID.
7. An admin reviews a report and suspends an account.
8. An unauthorized user cannot access admin actions.

### Load testing
Use a free/open-source load-testing tool such as k6 or an equivalent. Generate synthetic accounts and posts. Test:
- Normal browsing and feed requests.
- Concurrent likes/comments.
- Many users refreshing an academic notice.
- Database connection pool saturation.
- Cache cold starts and cache misses.
- Worker backlog and retry behavior.

Record p50, p95, p99 latency, throughput, error rate, database CPU/connections, cache hit rate, and queue depth. Do not claim a capacity target is met without recording the test configuration and results.

---

## 15. Performance and scaling rules

Initial proposed targets, to be validated by load tests:
- Cached feed/API read p95: under 200 ms where practical.
- Ordinary database-backed API read p95: under 300 ms where practical.
- Common indexed query p95: under 50 ms where practical.
- Initial monthly availability objective: 99.9% only after production redundancy and operational controls exist.

These are engineering targets, not guarantees.

Optimization order:
1. Correct SQL queries and indexes.
2. Cursor pagination.
3. Bounded connection pooling.
4. Batch data hydration to prevent N+1 queries.
5. Cache hot, permission-safe reads.
6. Move slow tasks to background workers.
7. Use hybrid feed fan-out only when measured read/write patterns justify it.
8. Extract feed, media, or notification components into separately deployed services when independent scaling or ownership justifies it.
9. Add read replicas or a second database only after measurement.

Do not assume 50,000 registered users implies a specific server size. Measure daily active users, concurrency, requests per second, media size, and peak announcement traffic.

---

## 16. Free deployment strategy

### Development
- Local Next.js and NestJS.
- PostgreSQL and Valkey in Docker Compose.
- Local synthetic data and local media adapter.
- GitHub Free repository and Actions within plan limits.

### Pilot
- Frontend on a suitable free static hosting plan, if compatible with the chosen Next.js deployment mode.
- PostgreSQL on a free hosted tier such as Neon or Supabase, after reviewing current limits and terms.
- API on a free compute plan only if the provider permits the intended use and its sleep/uptime limitations are acceptable.
- Object storage such as Cloudflare R2 within current free allowances.
- Keep a local Docker deployment available for reproducibility and migration testing.

### Important free-tier constraints
- Free tiers may sleep, pause, limit requests, restrict background workers, or change eligibility.
- A frontend free tier does not automatically provide a permanently running NestJS API.
- Free database tiers are not a substitute for a tested backup and recovery plan.
- A free storage allowance is not unlimited media hosting.
- Do not promise school-wide uptime on a single hobby-tier deployment.
- Verify current provider pricing, terms, data location, privacy commitments, and quotas before deployment.

### Migration path
Keep:
- Environment-based configuration.
- Standard PostgreSQL migrations.
- A storage adapter with local and S3-compatible implementations.
- A queue/outbox abstraction.
- Documented REST contracts.
- Dockerfiles and Compose configuration.
- Stateless API instances wherever practical.

This allows migration from free hosting to a paid VM or managed services without redesigning the product.

---

## 17. Security checklist

Before production:
- [ ] Passwords use a secure password-hashing algorithm.
- [ ] Signup cannot grant privileged roles.
- [ ] Role and membership checks are server-side.
- [ ] Object-level authorization exists for profiles, posts, comments, announcements, and media.
- [ ] Rate limiting covers login, recovery, posting, comments, and uploads.
- [ ] CORS allows only configured origins.
- [ ] Cookie-based authentication has CSRF protection where required.
- [ ] Tokens, passwords, and secrets are not logged.
- [ ] Uploads are size/type/content validated and quarantined until approved.
- [ ] Private media cannot be accessed through predictable public URLs.
- [ ] Administrative changes create audit records.
- [ ] Database backups and restore procedures are tested.
- [ ] Dependency and container images are reviewed and updated.
- [ ] Student data is not used in development or public demo environments.
- [ ] Privacy, retention, consent, and guardian policies are reviewed before launch.

---

## 18. Definition of done

The project is ready for a controlled pilot only when:
1. All core requirements in Section 1 are implemented or explicitly marked out of scope.
2. Database migrations work from a clean environment.
3. Core API endpoints are documented in OpenAPI.
4. Authentication and object-level authorization tests pass.
5. Social and academic feeds enforce their separate rules.
6. Media upload and private-media access controls are tested.
7. Critical workflows have integration and end-to-end coverage.
8. Docker startup and local development steps work from a clean checkout.
9. No secrets or real student records are committed.
10. Backup and restore procedures have been exercised.
11. Load-test results are documented.
12. Free-tier limits and production risks are understood by the school.
13. `docs/IMPLEMENTATION_STATUS.md` accurately describes the state of the project.

---

## 19. AI agent's first task

Start with repository inspection and Phase 0. Then implement **Phase 1 only**. Do not generate the entire application in one unreviewed batch.

At the end of Phase 1:
1. Run all lint, typecheck, test, and build commands that exist.
2. Fix failures.
3. Verify Docker Compose starts PostgreSQL and Valkey.
4. Verify the API health endpoint and frontend startup.
5. Update `docs/IMPLEMENTATION_STATUS.md`.
6. List any deviations from this README and explain why.
7. Proceed to Phase 2 only when Phase 1 acceptance criteria pass.

**Project principle:** correctness, privacy, and maintainability come before premature microservice complexity. Build a reliable modular application first, measure it, and extract services only when evidence justifies the added operational cost.
