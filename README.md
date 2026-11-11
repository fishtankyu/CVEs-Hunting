# BookStack — Incorrect Authorization in Image/Drawio Paths Allows Access to Unpublished Draft Images

- **Reporter:** Zeng Yu (zengyu82010@gmail.com)
- **Vendor:** BookStackApp
- **Product:** BookStack
- **CWE:** CWE-863 Incorrect Authorization
- **CVSS 3.1 (estimate):** 5.4 Medium — `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:L/A:N`
  (Estimate by the reporter; same score as the sibling CVE-2026-116020, which was CVSS 5.4.)
- **CVE ID:** Requested from VulnCheck (pending assignment)
- **Status:** Reported privately 30 Oct 2026 · Confirmed valid by the maintainer 7 Nov 2026 · Fixed in 26.09.3, released 11 Nov 2026

## Summary

In a shared book, an authenticated user who cannot view another user's unpublished draft page can still access that draft's images by ID. A Viewer can read the draft's Drawio diagram, and an Editor can rename, replace, or delete the draft's images. The draft-aware image-gallery API correctly denies the same draft, so the image paths are draft-blind rather than unchecked entirely.

This is a sibling of the draft-attachment issue fixed in v26.09.2 (CVE-2026-116020). That fix covered attachments; the image / Drawio paths below were not covered by it.

## Affected versions

- **Vulnerable:** BookStack before 26.09.3.
  - Lab-tested on development commit `045c6a98cac21ea86906674bb91d01f375460db6` (version file v26.05-dev), run locally only.
  - A static code check of the v26.09.2 tag showed the same draft-blind pattern, but the lab was not re-run on that tag, so v26.09.2 exploitability is not claimed as proven here.
- **Fixed:** 26.09.3 (released 11 Nov 2026).

## Affected endpoints

1. `GET /images/drawio/base64/{id}` — Drawio diagram read.
   Code: `app/Uploads/Controllers/DrawioImageController.php::getAsBase64()`, loading via `ImageRepo::getById()` (plain ID lookup, no draft/visible scope).
2. `PUT /images/{id}` — rename.
3. `PUT /images/{id}/file` — replace file contents.
4. `DELETE /images/{id}` — delete.
   Code: `app/Uploads/Controllers/ImageController.php`, shared gate `checkImagePermission()`. These load by plain `getById()` and check PageView on the related page plus role-based image permissions, without tying the image to the caller's own pages or to draft visibility.

## Impact

- **Confidentiality (Viewer):** reading unpublished draft content (a Drawio diagram) that the author has not published.
- **Integrity / availability (Editor):** renaming, replacing the file contents of, or deleting unpublished draft images — destroying unpublished work, or making a later-published page show attacker-chosen content.
- **Scope:** tested with local test accounts only. No live, public, or third-party BookStack site was tested, and no real user's data was involved.

## Reproduction (local lab, two test accounts only)

1. Victim creates an unpublished draft page with images, including a Drawio diagram, in a book the attacker can view.
2. Attacker (Viewer, not the draft owner) tries the draft page URL → 404, and the draft-aware API `GET /api/image-gallery/{id}` → 404.
3. Attacker (Viewer) requests `GET /images/drawio/base64/{id}` → **200**, with content decoding to the exact image the victim uploaded.
4. Negative control: in a book the attacker cannot view after an explicit deny, the same Drawio endpoint → 404, showing PageView is checked but is draft-blind for a shared book.
5. Attacker (Editor, not the draft owner) on separate victim draft images: rename → 200 and the database name changed; replace file → 200 and the on-disk file hash changed; delete → 200 and the database row and file were gone. The same Editor's draft-page and image-gallery controls → 404, with no name change.
6. Control: as Viewer, rename → 302 redirect, name unchanged, while the Drawio read still returned 200.

## Root cause

The image-gallery API uses a draft-aware visible scope (`getVisiblePageImageById()` / `findVisibleByIdOrFail()`), while the paths above load by plain ID and check only PageView plus role. Applying the same draft-aware visibility check to the Drawio and image write/delete paths is the consistent fix.

## Fix

BookStack 26.09.3 (security release, 11 Nov 2026): "Images & drawings could be unexpectedly accessed and modified when attached to inaccessible draft/trashed pages." Fix note: "Updated image fetching to fully check access via page access, considering draft and trashed status."
Release notes: https://www.bookstackapp.com/blog/bookstack-release-v26-09-3/

## Email correspondence

The maintainer (Dan Brown, lead maintainer of BookStack) confirmed the report valid on 7 Nov 2026 and asked the reporter to raise the CVE, on condition the details were not made public until the patch release was public. Screenshot of that correspondence:

<img width="938" height="364" alt="image" src="https://github.com/user-attachments/assets/cb0304de-bc32-4fdd-a291-7c6510ca9512" />
