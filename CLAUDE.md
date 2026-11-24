# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

This directory is **not itself a git repo** — it holds two independently-versioned projects, each with its own `.git`, remote, and history. Always run `git` commands from inside the relevant subfolder, not the root.

- `routopia_back/` — Spring Boot 3.4 (Java 17) REST API. Remote: `git@github.com:marian3299/routopia_back.git`
- `tour_booking/` — React 19 + Vite SPA (package name `hotel_booking`). Remote: `git@github.com:marian3299/routopia_front.git`

Both repos use `main` and `develop` branches; work happens on `develop`.

The root also has a stray `package.json` (just `react-datepicker`) and a `routopia-uploader_accessKeys.csv` / `backer_key.pem` — these are credential files, not project config; never read, print, or commit their contents.

Together the app is **Routopia**: a tour/destination booking platform (browse destinations, book tours, favorites, reviews, admin management of destinations/categories/traits/users).

## Backend — `routopia_back/`

### Commands (run from `routopia_back/`)
- Run the app: `./mvnw spring-boot:run` (serves on `:8080`)
- Build: `./mvnw clean install`
- Run all tests: `./mvnw test`
- Run a single test class: `./mvnw test -Dtest=ClassName`
- Local config: copy `src/main/resources/application.properties.example` to `application.properties` and fill in AWS/JWT values — the real file is gitignored and never committed.

### Stack & architecture
- Java 17, Spring Boot 3.4 (`web`, `data-jpa`, `data-jdbc`, `security`, `validation`), H2 file DB at `./data/routopia` (`spring.jpa.hibernate.ddl-auto=update`), JWT auth (`jjwt`), AWS S3 for image storage, springdoc-openapi (Swagger UI at `/swagger-ui/**`).
- Layering: `controller` → `service` → `repositroy` (repository package name — note the typo is intentional/existing, keep it consistent rather than "fixing" it) → JPA `entity`. Request/response shapes go through `dto` classes, not entities directly.
- `SecurityConfig` (root package, not `config/`) is the single source of truth for route access: stateless JWT sessions, `JwtAuthenticationFilter` runs before `UsernamePasswordAuthenticationFilter`. Public (no auth): `/api/auth/**`, `/api/destino/**`, `/api/traits/**`, `/api/categories/**`, `/api/reviews/**` (GET), `/api/bookings/availability` (GET), `/share/**` (GET), Swagger, H2 console. `/api/admin/**` requires `ROLE_ADMIN`; everything else requires authentication. `@EnableMethodSecurity` is also on, so some endpoints enforce finer-grained checks in code.
- `User` implements Spring Security's `UserDetails` directly (email is the username) and carries a free-form `permissions` list (stored via `StringListConverter` as a delimited string column) in addition to `Role` (`ADMIN`/`USER`) — permission checks are granular and separate from role.
- `DataInitializer` (`CommandLineRunner`) seeds an admin (`admin@routopia.com` / `admin123`) and a demo user (`user@routopia.com` / `user123`) on startup if they don't exist — this is dev/demo seed data, not production behavior.
- `Destino` is the central entity: `ManyToOne` Category, `ManyToMany` Traits (shared catalog), `OneToMany` cascaded Policies, `@ElementCollection`s for secondary images and languages. Filtering/search (e.g. by category, traits) goes through `DestinoSpecification` (JPA Specifications), not hand-rolled queries.
- `S3Service` centralizes image upload/deletion to S3; controllers that accept `MultipartFile` (destinos, traits) call into it rather than touching the AWS SDK directly.
- `SharePageController` serves public server-rendered Open Graph meta tags for `/share/destino/{id}` links (for social previews), redirecting real users to `app.frontend-url`.

## Frontend — `tour_booking/`

### Commands (run from `tour_booking/`)
- Dev server: `npm run dev` (Vite, proxies `/api/*` → `http://localhost:8080/*`, stripping the `/api` prefix)
- Build: `npm run build`
- Lint: `npm run lint`
- Preview production build: `npm run preview`

There is no test runner configured in this project.

### Stack & architecture
- React 19 + Vite 6, React Router v7, Redux Toolkit (`redux/` — used alongside React Context, not instead of it), React Hook Form, Axios.
- **Hook/component split convention**: most stateful UI logic lives in a `use<Thing>`/`use<Thing>Form`/`use<Thing>List` hook in `src/hooks/`, and the corresponding component in `src/components/` or `src/routes/` stays close to presentational. When touching a feature, expect to edit the hook and the component together.
- `src/services/*.service.js` wrap a shared `api.js` Axios instance (baseURL hardcoded to `http://localhost:8080/api`, JWT pulled from the `token` cookie via `js-cookie` and attached as `Authorization: Bearer`). Endpoint path strings live centrally in `src/services/urls.js` (`URLS` object) — add new endpoints there rather than inlining paths in services.
- Auth/session state: `context/AuthContext.jsx`. Favorites: `context/FavoritesContext.jsx`. Both wrap the whole app in `App.jsx`.
- Authorization in the UI mirrors the backend's permission strings: `constants/permissions.js` defines a `PERMISSIONS` object (e.g. `destinos:create`, `traits:edit`) consumed by `ProtectedRoute` via a `requiredPermission` prop — keep new permission strings in sync with whatever the backend issues on the user.
- Routing is centralized in `App.jsx`: every non-auth route is wrapped in `ProtectedRoute` (plain auth check, or auth + specific permission for admin-ish routes like `/new-tour`, `/users`, `/traits`, `/categories`, `/admin`).
