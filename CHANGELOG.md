# HideMyEmail Changelog

## [1.0.1] - 2026-01-14

### Fixed
- Added explicit `Content-Type: application/json` headers to all Hide My Email API requests
- This fixes potential issues with requests being rejected by Apple's API

### Notes
- If you encounter "Invalid global session" errors, refresh your cookie from the browser's Network tab (see README for instructions)

## [Initial Version] - {PR_MERGE_DATE}