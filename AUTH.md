# Touchline accounts and sign-in

**Team decision · 23 November 2026 · Confirmed by Phatu.** Touchline’s two login options are **Continue with Facebook** and **Sign in with Apple**. A fan uses the same button to join for the first time or return on another device. Neither flow asks the fan to register a separate email address or create a Touchline password.

## Where accounts appear

Open the relevant Touchline project in the Supabase dashboard and select **Authentication → Users**. This is the account directory for both providers. On the first successful provider sign-in, Supabase creates or resolves the Auth user and issues a Touchline session. Signing in again with the same provider identity resolves to the same Supabase user ID. Staging and production each have their own directory. [Supabase Auth users](https://supabase.com/docs/guides/auth/users)

| System | What it handles |
|---|---|
| Facebook or Apple | Authenticates the fan and shares the identity details authorised for Touchline. The fan enters any provider credentials with that provider. |
| Supabase Auth | Maintains the Auth user, associated provider identity and session. A provider-supplied email address can appear on the account. With Apple, this may be a private relay address. [Facebook sign-in](https://supabase.com/docs/guides/auth/social-sign-in/auth-facebook) · [Apple sign-in](https://supabase.com/docs/guides/auth/social-sign-in/auth-apple) |
| Touchline API and database | The app sends its Supabase access token to the API. On `GET /api/v1/me`, the API verifies the token and uses its subject, the Supabase user UUID, to find or create the manager profile in `app.users`. Fantasy teams, picks, points and league memberships stay in Touchline’s database. [Database guide](DB.md) |

## What still needs verification

The reviewed Expo app still offers email/password and Google, and its Facebook and Apple sign-in handler currently runs only on web. The separate browser frontend still stores local accounts. Those local records do not automatically appear in Supabase Auth. The live Supabase provider settings and the production account directory have not been verified. [Main and UI guide](touchline_main.md)

The current API also requires an email claim to open a manager profile. If a provider sign-in succeeds without one, the user can exist under **Authentication → Users** while `/api/v1/me` returns 400. Resolve that case using the Supabase user ID while preserving the two-button login experience. Matching email addresses alone do not prove that two profiles belong to the same fan. [API guide](touchline_api.md)

**Staging acceptance check.** For each provider, sign in on a phone, find the test user in **Authentication → Users**, and confirm that the dashboard UUID matches `authUserId` from `/api/v1/me`. Sign out and return with the same provider on a second device. The same manager profile and fantasy team should load without an email registration step.
