# Explanation

## What was the bug?

The `Client.request()` method in `app/http_client.py` failed to refresh the OAuth2 token when `oauth2_token` was set to a plain `dict` instead of an `OAuth2Token` instance. This caused API requests to be sent without an Authorization header.

## Why did it happen?

The original condition used `not self.oauth2_token` to check for a missing token. A non-empty dict is truthy in Python, so `not {"access_token": "stale", "expires_at": 0}` evaluates to `False`. The second branch (`isinstance(self.oauth2_token, OAuth2Token) and self.oauth2_token.expired`) also evaluates to `False` because a dict is not an `OAuth2Token`. As a result, `refresh_oauth2()` was never called, and since the subsequent `isinstance` guard on line 35 also failed, no Authorization header was set.

## Why does your fix solve it?

The fix replaces the compound condition with `not isinstance(self.oauth2_token, OAuth2Token) or self.oauth2_token.expired`. This correctly triggers a refresh whenever the token is anything other than a valid, non-expired `OAuth2Token` — covering `None`, `dict`, and expired token cases uniformly.

## One realistic case/edge case the tests still don't cover

The tests don't cover the scenario where `refresh_oauth2()` itself fails (e.g., network error, invalid credentials). In production, a failed refresh could leave `oauth2_token` in an inconsistent state, and the request would proceed without authentication or raise an unhandled exception.
