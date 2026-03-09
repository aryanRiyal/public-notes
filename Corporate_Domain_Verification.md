# Architecture Proposal: Dynamic Corporate Domain Verification

## 1. Objective

To replace the existing hardcoded `EMAIL_DOMAIN_DENYLIST` with a dynamic, scalable, and partially automated domain verification pipeline. This ensures high-trust corporate onboarding while preventing abuse from disposable email providers, minimizing manual admin intervention, and keeping our database lean.

## 2. Proposed Data Model Changes

We will introduce a new entity and update existing ones to separate **Authentication** (OTP verification) from **Authorization** (Corporate domain approval).

### New Entity: `CorporateDomain`

Stores the dynamic list of processed corporate domains.
| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | `Long` | Primary Key. |
| `domainName` | `String` | Unique domain (e.g., `houseclay.com`). |
| `status` | `Enum` | `ALLOWED`, `DENIED`, `PENDING`. |
| `websiteTitle` | `String` | (Optional) Auto-fetched `<title>` tag of the domain's website to assist admins. |
| `createdAt` | `Timestamp` | Record creation time. |
| `updatedAt` | `Timestamp` | Last status update time. |

### Updates to `User` Entity

Adds explicit tracking for the user's benefit authorization state.
| Field | Type | Description |
| :--- | :--- | :--- |
| `corporateBenefitStatus` | `Enum` | `NONE`, `PENDING_ADMIN_APPROVAL`, `APPROVED`, `REJECTED`. |

_(Note: The existing `isCorporateEmailVerified` boolean remains to confirm OTP ownership)._

### Updates to `UserUpdateType` Enum

Adds precise audit logging for the `UserUpdateLog` table.

- `CORPORATE_BENEFIT_PENDING`
- `CORPORATE_BENEFIT_VERIFIED`
- `CORPORATE_BENEFIT_REJECTED`

## 3. The Verification Pipeline

The backend will process verification requests using a "fail-fast" hybrid approach to minimize unnecessary database hits and external API calls.

### Phase 1: Initiation (`/verify-corporate-email-init`)

When a user submits an email (e.g., `user@company.com`), the system evaluates the domain in the following order:

1. **In-Memory Disposable Check (O(1)):** \* Check the domain against a `HashSet` of known disposable domains (maintained via a weekly cron job fetching from public open-source lists).
   - _If match:_ Reject instantly (`400 Bad Request`). Do not touch the DB.
2. **Database Known-Entity Check:**
   - Query `CorporateDomain` table (pre-populated with global giants like Gmail/Yahoo as `DENIED`).
   - _If DENIED:_ Reject instantly.
   - _If ALLOWED:_ Proceed to OTP generation.
3. **DNS MX Record Check:**
   - Perform a standard Java DNS lookup to verify the domain has active mail servers.
   - _If no MX records:_ Reject instantly.
4. **Domain Metadata Fetch (Async/Background):**
   - If the domain is entirely new, trigger a quick HTTP GET to `https://{domain}` to scrape the `<title>` tag.
   - Insert the domain into `CorporateDomain` with status `PENDING` and save the scraped title.
5. **OTP Generation:** \* Send the OTP to the user.

### Phase 2: Confirmation (`/verify-corporate-email-confirm`)

Once the user submits the correct OTP:

1. Set `User.isCorporateEmailVerified = true`.
2. Check the associated `CorporateDomain.status`:
   - **If ALLOWED:** Update user `corporateBenefitStatus = APPROVED`. Automatically grant connects/benefits. Log `CORPORATE_BENEFIT_VERIFIED`.
   - **If PENDING:** Update user `corporateBenefitStatus = PENDING_ADMIN_APPROVAL`. Log `CORPORATE_BENEFIT_PENDING`. Return a `202 Accepted` response indicating the account is under review.

## 4. Admin Portal Enhancements

- **Endpoint:** `GET /api/admin/corporate-domains?status=PENDING`
- **Workflow:** Admins will see a list of pending domains alongside the automatically fetched `websiteTitle` (e.g., Domain: `sharmarealestate.in` | Title: `Sharma Real Estate - Ludhiana`).
- **Action:** Admins can bulk approve or deny domains.
- **Cascading Updates:** Approving a domain automatically triggers a background process to find all `User` records with `PENDING_ADMIN_APPROVAL` for that domain, upgrades them to `APPROVED`, and grants their connects.

## 5. Future Scope (Optional Enhancements)

- **Third-Party Enrichment Integration:** For instant automated approvals, we can integrate an API like Abstract API (Free Tier), Hunter.io, or Clearbit between Step 3 and 4 of the Initiation phase. If the API returns a high-confidence match for an established corporate entity, we can bypass the `PENDING` state and insert the domain directly as `ALLOWED`.
