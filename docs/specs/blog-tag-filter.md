# Spec: Blog tag filter

## Goal
Readers can narrow the public blog post list to posts carrying a chosen tag, and the selection is shareable via the URL.

Assumption (see Open questions Q1): "tag" means a NEW multi-valued `tags` field on a blog, distinct from the existing single `category`.

## Context
- Blog model has no tag field; `category` is a required free-form String: `server/src/models/Blog.js:3-12` (category at :7).
- Public list `GET /api/blog/all` (`server/src/routes/blogRoutes.js:21`) calls `Blog.find({ isPublished: true })` with no filter, sort or pagination and returns `{success,count,blogs}`: `server/src/controllers/blogController.js:60-67`. No `req.query` handling exists in `server/src`.
- `addBlog` does `JSON.parse(req.body.blog)` and requires title/description/category/image: `blogController.js:33-40`; `validateBlogInput` requires category: `server/src/validators/blogValidator.js:11-13`.
- Indexes are created by migrations named `YYYYMMDDNNNNNN-description.js` (up/down, dbLogger), e.g. `server/migrations/20241014000001-add-blog-indexes.js:18-22`.
- Seed data: `server/fixtures/blogs.js` (6 blogs), loaded by `server/scripts/seed.js:7-9,29,46-55`.
- Client fetches all blogs once into context: `client/src/context/AppContext/AppContext.jsx:13-48`; `blogApi.getAll()` takes no params: `client/src/api/blogApi.js:6-8`.
- `BlogList` filters client-side (search on title/category, category via local `useState('All')`) and renders `CategoryFilter` + `BlogGrid`: `client/src/components/blog/BlogList/BlogList.jsx:6-37`. `CategoryFilter` maps hardcoded `BLOG_CATEGORIES` (`client/src/constants/categories.js:1`), also used by admin `AddBlog` (`client/src/pages/admin/AddBlog/AddBlog.jsx:5,185`).
- No URL-based filter state (`useSearchParams`) exists in the client.
- Existing tests: client `BlogList.test.jsx`, `CategoryFilter.test.jsx`; server `tests/blogCrud.test.js`, `blogModel.test.js`, `blogValidator.test.js`, `tests/fixtures.js:28`.

## Requirements
- R1 Blog schema gains `tags: [String]`, default `[]`; tags are stored trimmed and lowercased, deduplicated, max 10 per blog, each 1-30 characters.
- R2 `addBlog` accepts an optional `tags` array in the `blog` JSON and rejects invalid tags with HTTP 400 and a clear message; existing payloads without `tags` keep working.
- R3 `GET /api/blog/all` accepts an optional `tag` query parameter and returns only published blogs whose `tags` contain that (normalized) tag; without it the response is unchanged. An unknown tag returns `success: true`, `count: 0`, `blogs: []`.
- R4 A migration adds an index on `tags` (up and down), following the existing migration style.
- R5 Seed fixtures give each of the 6 blogs 1-3 tags, with at least one tag shared by 2 or more blogs.
- R6 The blog list page shows a tag filter listing the distinct tags present in loaded published blogs, plus an "All" option.
- R7 Selecting a tag shows only blogs with that tag; the selection is stored in the URL as `?tag=<tag>`, and loading that URL pre-selects the tag.
- R8 Tag filtering combines with the existing search input and category filter using AND.
- R9 Admin AddBlog form lets the admin enter tags; they are sent in the `blog` JSON.

## Acceptance criteria
- [ ] AC1 (R1) Model test: tags `[" React ", "react", "A"]` save as `["react","a"]`; 11 tags or a 31-char tag fails validation.
- [ ] AC2 (R2) supertest: POST with `tags: ["node"]` returns 201 and the blog has `tags: ["node"]`; POST with 11 tags returns 400; POST without `tags` returns 201 with `tags: []`.
- [ ] AC3 (R3) supertest: with 2 blogs tagged `node` and 1 tagged `css`, `?tag=node` returns count 2, `?tag=NODE` returns count 2, `?tag=none` returns count 0, and no param returns all 3; unpublished blogs are never returned.
- [ ] AC4 (R4) Migration file exists with up/down; running up then down leaves the `tags` index created then removed.
- [ ] AC5 (R5) After `npm run seed`, every seeded blog has 1-3 tags and some tag appears on at least 2 blogs.
- [ ] AC6 (R6) Component test: BlogList renders one button per distinct tag plus "All".
- [ ] AC7 (R7) Component test: clicking a tag hides non-matching blogs and sets `tag` in the URL; rendering at `?tag=x` shows only tag-x blogs.
- [ ] AC8 (R8) Component test: tag + category + search text together return only blogs matching all three.
- [ ] AC9 (R9) AddBlog test: entering tags `a, b` submits `tags: ["a","b"]` in the blog JSON.
- [ ] AC10 Existing client and server test suites still pass.

## Out of scope
- Tag management UI (rename/merge/delete tags), tag autocomplete, tag pages/SEO routes.
- Editing tags on existing blogs (no update endpoint is covered) and backfilling tags on existing production data.
- Pagination, sorting, or server-side search.
- Removing or changing the existing `category` filter.

## Open questions
- Q1 Does "tag" mean the existing single `category`, or a new multi-valued `tags` field? The code cannot answer this. This spec assumes a new `tags` field. If `category` is meant, R1, R2, R4, R5 and R9 are dropped, and the work reduces to URL-persisted category filtering (R6-R8) plus an optional server `category` param.
- Q2 Should filtering be server-side (R3, as written) or client-side only, since the client already loads all blogs?
- Q3 Should tags be required, or optional as written?
- Q4 Are the limits (10 tags per blog, 30 characters per tag) acceptable?
