Purpose
This controller powers a “Fund Explorer” API endpoint that returns a paginated list of mutual funds filtered by categories, AMC, SIP/Lumpsum minimum amounts, and a flexible search query, with supporting facet data for parent/child categories and AMC names to drive UI filters.

High-level flow
Validate and normalize incoming query parameters using a strict schema.

Build a MongoDB filter object dynamically based on the validated inputs.

Execute the main data query and parallel facet queries efficiently with Promise.all.

Return results with applied filters echo, facets, and robust pagination metadata.

Imports
Fund: Mongoose model for funds; exposes find, countDocuments, distinct, and aggregate APIs used for retrieval and faceting.

z from zod: runtime schema validation library used to define and enforce constraints on query parameters with precise error messages.

Helpers
escapeRegex(s): Escapes all regex metacharacters in user strings to prevent regex injection and unintended pattern expansion when constructing MongoDB regex queries; replaces characters like ., *, +, ?, ^, $, {, }, (, ), |, [, ], \ with safe literals .

toFiniteNumber(v): Safely coerces string inputs to finite numbers; returns null for undefined/null or non-finite results so NaN and Infinity never reach MongoDB operators, preserving predictable filters and enabling clean 400 validations when needed.

Validation schema (zod)
parent_category, child_category:

Required to be trimmed, length 1–60, and match a whitelist of characters: letters, digits, spaces, dot, ampersand, parentheses, dash, underscore, slash; this blocks control characters, quotes, and special symbols, reducing injection risk and ensuring predictable index usage.

amcName:

Trimmed, 1–100 length, allows letters/digits/spaces and common punctuation; helps keep regex scope safe and improves match quality while preventing degenerate inputs.

search:

Trimmed, length 2–100 to avoid trivial or overly expensive queries; this allows flexible substring matching while guarding against accidental collection scans from empty/1-char searches.

minSip, maxSip, minLumpsum, maxLumpsum, page, limit:

Accepted as strings (from querystring), then coerced and validated later; keeping them optional enables sparse, composable filters.

On validation failure, the controller returns HTTP 400 with a list of field-specific issues, enabling UI to highlight incorrect inputs precisely.

Pagination
page: parsed as integer, clamped to at least 1; defaults to 1 when absent or invalid.

limit: parsed as integer, clamped between 1 and 100 to prevent abuse and memory pressure; defaults to 20.

skip: computed as (page − 1) × limit for use with MongoDB skip/limit pagination.

Base filter
Starts with { isActive: true, "planOptions.option": "GROWTH" } to enforce only active funds and Growth plan options, keeping queries consistent and indexable across requests.

Dynamic filter construction
Categories:

If parent_category exists, adds filter["categories.parentCategoryId"] = parent_category.trim().

If child_category exists, adds filter["categories.name"] = child_category.trim().

Using nested paths aligns with the stored schema and aggregation paths, ensuring consistent behavior and index utilization.

AMC name:

Escapes and applies a case-insensitive, prefix-anchored regex on amcName, e.g., new RegExp(^HDFC, i), which improves performance with a normal index compared to unanchored regex scans.

Numeric ranges:

minSip/maxSip validate non-negative and min ≤ max; only add $gte/$lte if finite values are supplied; same logic for minLumpsum/maxLumpsum on minLumpsumAmount.

Early 400 responses on invalid combinations (e.g., min > max) prevent confusing “no results” outcomes and protect the database from ambiguous queries.

Search:

Constructs a “fuzzy” case-insensitive regex by removing spaces and allowing optional whitespace between characters: "H D F C" matches “HDFC,” “H D F C,” etc., improving user experience on varied spacing.

Applies this regex across multiple fields within $or: schemeName, amcName, planName, categories.parentCategoryId, and categories.name, so the search can discover matches across names and classifications while still honoring structured filters added above.

Net effect: all structured filters and the fuzzy search are combined so searches refine results rather than override other constraints, delivering intuitive filtering behavior in the UI.

Facets
Parent categories:

Fund.distinct("categories.parentCategoryId", { isActive: true }) provides a list of available parent categories to populate a top-level filter component, independent of current selection, which simplifies browsing.

Child categories and AMC (scoped):

When a parent_category is supplied, an aggregation pipeline fetches all unique child category names under that parent, and a distinct query fetches AMCs under that parent; this yields contextual facets relevant to the current selection, improving UX and reducing noise.

These facet queries run in parallel with the main data query to minimize latency.

Query execution
Main list:

Fund.find(filter).select("-__v").lean().sort({ createdAt: -1 }).skip(skip).limit(limit) returns lightweight plain objects, excludes internal __v, orders by recency, and applies pagination efficiently.

Count:

Fund.countDocuments(filter) computes the total result size for precise pagination controls and page count feedback.

Promise.all:

Executes the main query, count, and facet queries concurrently, reducing total response time compared to sequential execution.

Response contract
success/message: simple status and human-readable message.

categoryNames: array of { parentName } derived from distinct parent categories for UI population.

childCategories: aggregation result with _id: parentCategoryId and childNames: [names] when parent_category is chosen; empty otherwise.

AmcNames: distinct AMC names under the selected parent; empty when none selected.

appliedFilters: echoes normalized and coerced filters (numbers as numbers or null), aiding client-side state sync and debugging.

data: the paginated fund list.

pagination: currentPage, perPage, total, totalPages, hasNext, hasPrevious for client navigation.

Validation logic explained
String validations (categories, amcName):

trim removes accidental leading/trailing spaces; min length avoids empty matches; max length prevents payload abuse; whitelist regex blocks problematic characters and ensures consistent indexing behavior.

Search validation:

min length 2 avoids very broad scans; max 100 avoids abuse; additional normalization removes spaces and permits fuzzy whitespace tolerance to match user expectations across differently spaced inputs.

Numeric validations:

toFiniteNumber ensures only finite values enter the filter; rejects negative amounts; enforces min ≤ max; early 400s prevent wasteful DB queries and ambiguous UX.

Pagination validation:

Clamps page/limit to sensible bounds to prevent excessive memory usage and denial of service via extreme limits.

Operational considerations
Indexes:

Create compound indexes such as { isActive: 1, "planOptions.option": 1, "categories.parentCategoryId": 1, "categories.name": 1 } and { isActive: 1, "planOptions.option": 1, amcName: 1 } to accelerate common filters; consider text or prefix-search strategy if search is heavy.

Security:

escapeRegex prevents regex injection; whitelist validation reduces query ambiguity; returning 400 on invalid ranges prevents pathologically expensive queries.

Observability:

Keep the production error response generic while logging the full error server-side; include appliedFilters in success responses for easier debugging and analytics.

Example request/response
Request: GET /api/funds?parent_category=Equity&amcName=HDFC&minSip=500&search=hdfc bluechip&page=2&limit=20.

Behavior: Validates inputs; builds filter with isActive, Growth, parent category, AMC prefix regex, minSipAmount ≥ 500, and fuzzy search across names; returns page 2 with 20 results per page, plus parent/child/AMC facets and full pagination meta
