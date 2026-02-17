# Bug Explanation

## What was the bug?

The `HttpClient.request()` method in `src/httpClient.ts` failed to refresh OAuth2 tokens when the token was stored as a plain JavaScript object (e.g., `{ accessToken: "stale", expiresAt: 0 }`) instead of an `OAuth2Token` class instance. This caused the Authorization header to be missing from API requests when a plain object token was expired or missing.

## Why did it happen?

The original code used an `instanceof` check to determine if the token was an `OAuth2Token` instance before checking if it was expired:

```typescript
if (
  !this.oauth2Token ||
  (this.oauth2Token instanceof OAuth2Token && this.oauth2Token.expired)
) {
  this.refreshOAuth2();
}
```

This logic missed the case where `oauth2Token` was a plain object. Since a plain object is truthy and fails the `instanceof OAuth2Token` check, the token refresh was never triggered even when the plain object had an expired timestamp.

## Why does your fix solve it?

The fix adds a generic check for the `expiresAt` property that works for both `OAuth2Token` instances and plain objects:

```typescript
const isExpired =
  this.oauth2Token &&
  "expiresAt" in this.oauth2Token &&
  (this.oauth2Token as { expiresAt: number }).expiresAt <= Math.floor(Date.now() / 1000);

if (!this.oauth2Token || isExpired) {
  this.refreshOAuth2();
}
```

This approach checks if the token exists and has an `expiresAt` property that has passed, regardless of whether it's a class instance or plain object. After refreshing, the new token is always an `OAuth2Token` instance, so the Authorization header is correctly set.

## One realistic case / edge case your tests still don't cover

The fix doesn't handle the case where a plain object token has a valid (non-expired) `expiresAt` but is missing the `accessToken` property. In this scenario, the token would pass the expiration check but would fail when trying to use it, since there's no token string to send in the Authorization header.
