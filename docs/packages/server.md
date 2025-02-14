---
title: "@simplewebauthn/server"
---

## Installation

### Node LTS 16.x or higher
This package is available on **npm**:

```bash
npm install @simplewebauthn/server
```

It can then be imported into all types of Node applications thanks to its support for **both CommonJS and [ECMAScript modules (ESM)](https://nodejs.org/api/esm.html#enabling)** projects:

```ts
// CommonJS (NodeJS)
const SimpleWebAuthnServer = require('@simplewebauthn/server');

// ES Module (NodeJS w/module support, TypeScript, Babel, etc...)
import SimpleWebAuthnServer from '@simplewebauthn/server';
```

### Deno v1.33.x or higher

This package is also available for installation in **Deno** projects from **deno.land/x**:

```ts
import {...} from 'https://deno.land/x/simplewebauthn/deno/server.ts';
```

## Additional data structures

Documentation below will refer to the following TypeScript types. These are intended to be inspirational, a simple means of communicating the...types...of values you'll need to be capable of persisting in your database:

```ts
type UserModel = {
  id: string;
  username: string;
  currentChallenge?: string;
};

/**
 * It is strongly advised that authenticators get their own DB
 * table, ideally with a foreign key to a specific UserModel.
 *
 * "SQL" tags below are suggestions for column data types and
 * how best to store data received during registration for use
 * in subsequent authentications.
 */
type Authenticator = {
  // SQL: Encode to base64url then store as `TEXT`. Index this column
  credentialID: Uint8Array;
  // SQL: Store raw bytes as `BYTEA`/`BLOB`/etc...
  credentialPublicKey: Uint8Array;
  // SQL: Consider `BIGINT` since some authenticators return atomic timestamps as counters
  counter: number;
  // SQL: `VARCHAR(32)` or similar, longest possible value is currently 12 characters
  // Ex: 'singleDevice' | 'multiDevice'
  credentialDeviceType: CredentialDeviceType;
  // SQL: `BOOL` or whatever similar type is supported
  credentialBackedUp: boolean;
  // SQL: `VARCHAR(255)` and store string array as a CSV string
  // Ex: ['usb' | 'ble' | 'nfc' | 'internal']
  transports?: AuthenticatorTransport[];
};
```

## Identifying your RP

Start by defining some constants that describe your "Relying Party" (RP) server to authenticators:

```js
// Human-readable title for your website
const rpName = 'SimpleWebAuthn Example';
// A unique identifier for your website
const rpID = 'localhost';
// The URL at which registrations and authentications should occur
const origin = `https://${rpID}`;
```

These will be referenced throughout registrations and authentications to ensure that authenticators generate and return credentials specifically for your server.

## Registration

"Registration" is analogous to new account creation. Registration uses the following exported methods from this package:

```ts
import {
  generateRegistrationOptions,
  verifyRegistrationResponse,
} from '@simplewebauthn/server';
```

Registration occurs in two steps:

1. Generate registration options for the browser to pass to a supported authenticator
2. Verify the authenticator's response

Each of these steps need their own API endpoints:

### 1. Generate registration options

One endpoint (`GET`) needs to return the result of a call to `generateRegistrationOptions()`:

```ts
// (Pseudocode) Retrieve the user from the database
// after they've logged in
const user: UserModel = getUserFromDB(loggedInUserId);
// (Pseudocode) Retrieve any of the user's previously-
// registered authenticators
const userAuthenticators: Authenticator[] = getUserAuthenticators(user);

const options = await generateRegistrationOptions({
  rpName,
  rpID,
  userID: user.id,
  userName: user.username,
  // Don't prompt users for additional information about the authenticator
  // (Recommended for smoother UX)
  attestationType: 'none',
  // Prevent users from re-registering existing authenticators
  excludeCredentials: userAuthenticators.map(authenticator => ({
    id: authenticator.credentialID,
    type: 'public-key',
    // Optional
    transports: authenticator.transports,
  })),
  // See "Guiding use of authenticators via authenticatorSelection" below
  authenticatorSelection: {
    // Defaults
    residentKey: 'preferred',
    userVerification: 'preferred',
    // Optional
    authenticatorAttachment: 'platform',
  },
});

// (Pseudocode) Remember the challenge for this user
setUserCurrentChallenge(user, options.challenge);

return options;
```

These options can be passed directly into [**@simplewebauthn/browser**'s `startRegistration()`](packages/browser.mdx#startregistration) method.

:::tip Support for custom challenges

Power users can optionally generate and pass in their own unique challenges as `challenge` when calling `generateRegistrationOptions()`. In this scenario `options.challenge` still needs to be saved to be used in verification as described below.

:::

:::info Guiding use of authenticators via authenticatorSelection
`generateRegistrationOptions()` also accepts an `authenticatorSelection` option that can be used to fine-tune the registration experience. When unspecified, defaults are provided according to [passkeys best practices](advanced/passkeys.md#generateregistrationoptions). These values can be overridden based on Relying Party needs, however:

#### `residentKey` - one of:

- `'discouraged'`
  - Won't consume discoverable credential slots on security keys, but also won't generate synced passkeys on Android devices.
- `'preferred'`
  - Will always generate synced passkeys on Android devices, but will consume discoverable credential slots on security keys.
- `'required'`
  - Same as `'preferred'`

#### `userVerification` - one of:

- `'discouraged'`
  - Won't perform user verification if interacting with an authenticator won't automatically perform it (i.e. security keys won't prompt for PIN, but interacting with Touch ID on a macOS device will always perform user verification.) User verification will usually be `false`.
- `'preferred'`
  - Will perform user verification when possible, but will skip any prompts for PIN or local login password when possible. In these instances user verification can sometimes be `false`.
- `'required'`
  - Will always provides multi-factor authentication, at the expense of always requiring some users to enter their local login password during auth. User verification should never be `false`.

#### `authenticatorAttachment` - one of:

- `'cross-platform'`
  - Browsers will guide users towards registering a security key, or mobile device via hybrid registration.
- `'platform'`
  - Browser will guide users to registering the locally integrated hardware authenticator.
:::

### 1a. Supported Attestation Formats

If `attestationType` is set to `"direct"` when generating registration options, the authenticator will return a more complex response containing an "attestation statement". This statement includes additional verifiable information about the authenticator.

Attestation statements are returned in one of several different formats. SimpleWebAuthn supports [all current WebAuthn attestation formats](https://w3c.github.io/webauthn/#sctn-defined-attestation-formats), including:

- **Packed**
- **TPM**
- **Android Key**
- **Android SafetyNet**
- **Apple**
- **FIDO U2F**
- **None**

:::info
Attestation statements are an advanced aspect of WebAuthn. You can ignore this concept if you're not particular about the kinds of authenticators your users can use for registration and authentication.
:::

### 2. Verify registration response

The second endpoint (`POST`) should accept the value returned by [**@simplewebauthn/browser**'s `startRegistration()`](packages/browser.mdx#startregistration) method and then verify it:

```ts
const { body } = req;

// (Pseudocode) Retrieve the logged-in user
const user: UserModel = getUserFromDB(loggedInUserId);
// (Pseudocode) Get `options.challenge` that was saved above
const expectedChallenge: string = getUserCurrentChallenge(user);

let verification;
try {
  verification = await verifyRegistrationResponse({
    response: body,
    expectedChallenge,
    expectedOrigin: origin,
    expectedRPID: rpID,
  });
} catch (error) {
  console.error(error);
  return res.status(400).send({ error: error.message });
}

const { verified } = verification;
```

:::tip Support for multiple origins and RP IDs
SimpleWebAuthn optionally supports verifying registrations from multiple origins and RP IDs! Simply pass in an **array** of possible origins and IDs for `expectedOrigin` and `expectedRPID` respectively.
:::

When finished, it's a good idea to return the verification status to the browser to display
appropriate UI:

```ts
return { verified };
```

### 3. Post-registration responsibilities

Assuming `verification.verified` is true then RP's must, at the very least, save the credential data in `registrationInfo` to the database:

```ts
const { registrationInfo } = verification;
const {
  credentialPublicKey,
  credentialID,
  counter,
  credentialDeviceType,
  credentialBackedUp,
} = registrationInfo;

const newAuthenticator: Authenticator = {
  credentialID,
  credentialPublicKey,
  counter,
  credentialDeviceType,
  credentialBackedUp,
  // `body` here is from Step 2
  transports: body.response.transports,
};

// (Pseudocode) Save the authenticator info so that we can
// get it by user ID later
saveNewUserAuthenticatorInDB(user, newAuthenticator);
```

**Values:**

- `credentialID` (`bytes`): A unique identifier for the credential
- `credentialPublicKey` (`bytes`): The public key bytes, used for subsequent authentication signature verification
- `counter` (`number`): The number of times the authenticator has been used on this site so far

:::info Regarding `counter`
Tracking the ["signature counter"](https://www.w3.org/TR/webauthn/#signature-counter) allows Relying Parties to potentially identify misbehaving authenticators, or cloned authenticators. The counter on subsequent authentications should only ever increment; if your stored counter is greater than zero, and a subsequent authentication response's counter is the same or lower, then perhaps the authenticator just used to authenticate is in a compromised state.

It's also not unexpected for certain high profile authenticators, like Touch ID on macOS, to always return `0` (zero) for the signature counter. In this case there is nothing an RP can really do to detect a cloned authenticator, especially in the context of [multi-device credentials](https://fidoalliance.org/apple-google-and-microsoft-commit-to-expanded-support-for-fido-standard-to-accelerate-availability-of-passwordless-sign-ins/).

**@simplewebauthn/server** knows how to properly check the signature counter on subsequent authentications. RP's should only need to remember to store the value after registration, and then feed it back into `startAuthentication()` when the user attempts a subsequent login. RP's should remember to update the credential's counter value in the database afterwards. See [Post-authentication responsibilities](packages/server.md#3-post-authentication-responsibilities) below for how to do so.
:::

## Authentication

"Authentication" is analogous to existing account login. Authentication uses the following exported methods from this package:

```ts
import {
  generateAuthenticationOptions,
  verifyAuthenticationResponse,
} from '@simplewebauthn/server';
```

Just like registration, authentication span two steps:

1. Generate authentication options for the browser to pass to a FIDO2 authenticator
2. Verify the authenticator's response

Each of these steps need their own API endpoints:

### 1. Generate authentication options

One endpoint (`GET`) needs to return the result of a call to `generateAuthenticationOptions()`:

```ts
// (Pseudocode) Retrieve the logged-in user
const user: UserModel = getUserFromDB(loggedInUserId);
// (Pseudocode) Retrieve any of the user's previously-
// registered authenticators
const userAuthenticators: Authenticator[] = getUserAuthenticators(user);

const options = await generateAuthenticationOptions({
  rpID,
  // Require users to use a previously-registered authenticator
  allowCredentials: userAuthenticators.map(authenticator => ({
    id: authenticator.credentialID,
    type: 'public-key',
    transports: authenticator.transports,
  })),
  userVerification: 'preferred',
});

// (Pseudocode) Remember this challenge for this user
setUserCurrentChallenge(user, options.challenge);

return options;
```

These options can be passed directly into [**@simplewebauthn/browser**'s `startAuthentication()`](packages/browser.mdx#startAuthentication) method.

:::tip Support for custom challenges

Power users can optionally generate and pass in their own unique challenges as `challenge` when calling `generateAuthenticationOptions()`. In this scenario `options.challenge` still needs to be saved to be used in verification as described below.

:::

### 2. Verify authentication response

The second endpoint (`POST`) should accept the value returned by [**@simplewebauthn/browser**'s `startAuthentication()`](packages/browser.mdx#startAuthentication) method and then verify it:

```ts
const { body } = req;

// (Pseudocode) Retrieve the logged-in user
const user: UserModel = getUserFromDB(loggedInUserId);
// (Pseudocode) Get `options.challenge` that was saved above
const expectedChallenge: string = getUserCurrentChallenge(user);
// (Pseudocode} Retrieve an authenticator from the DB that
// should match the `id` in the returned credential
const authenticator = getUserAuthenticator(user, body.id);

if (!authenticator) {
  throw new Error(`Could not find authenticator ${body.id} for user ${user.id}`);
}

let verification;
try {
  verification = await verifyAuthenticationResponse({
    response: body,
    expectedChallenge,
    expectedOrigin: origin,
    expectedRPID: rpID,
    authenticator,
  });
} catch (error) {
  console.error(error);
  return res.status(400).send({ error: error.message });
}

const { verified } = verification;
```

When finished, it's a good idea to return the verification status to the browser to display
appropriate UI:

```ts
return { verified };
```

:::tip Support for multiple origins and RP IDs
SimpleWebAuthn optionally supports verifying authentications from multiple origins and RP IDs! Simply pass in an array of possible origins and IDs for `expectedOrigin` and `expectedRPID` respectively.
:::

### 3. Post-authentication responsibilities

Assuming `verification.verified` is true, then update the user's authenticator's `counter` property in the DB:

```ts
const { authenticationInfo } = verification;
const { newCounter } = authenticationInfo;

saveUpdatedAuthenticatorCounter(authenticator, newCounter);
```

## Troubleshooting

Below are errors you may see while using this library, and potential solutions to them:

### DOMException [NotSupportedError]: Unrecognized name.

Authentication responses may unexpectedly error out during verification. This appears as the throwing of an "Unrecognized name" error from a call to `verifyAuthenticationResponse()` with the following stack trace:

```
DOMException [NotSupportedError]: Unrecognized name.
    at new DOMException (node:internal/per_context/domexception:70:5)
    at __node_internal_ (node:internal/util:477:10)
    at normalizeAlgorithm (node:internal/crypto/util:220:15)
    at SubtleCrypto.importKey (node:internal/crypto/webcrypto:503:15)
    at importKey (/Users/swan/Developer/simplewebauthn/packages/server/src/helpers/iso/isoCrypto/importKey.ts:9:27)
    at verifyOKP (/Users/swan/Developer/simplewebauthn/packages/server/src/helpers/iso/isoCrypto/verifyOKP.ts:57:30)
    at Object.verify (/Users/swan/Developer/simplewebauthn/packages/server/src/helpers/iso/isoCrypto/verify.ts:31:21)
    at verifySignature (/Users/swan/Developer/simplewebauthn/packages/server/src/helpers/verifySignature.ts:34:20)
    at verifyAuthenticationResponse (/Users/swan/Developer/simplewebauthn/packages/server/src/authentication/verifyAuthenticationResponse.ts:206:36)
    at async /Users/swan/Developer/simplewebauthn/packages/server/345.ts:26:24
```

This appears to be an issue with some environments running **versions of Node prior to v18 LTS**.

To fix this, update your call to `generateRegistrationOptions()` to exclude `-8` (Ed25519) from the list of algorithms:

```ts
const options = await generateRegistrationOptions({
  // ...
  supportedAlgorithmIDs: [-7, -257],
});
```

You will then need to re-register any authenticators that generated credentials that cause this error.

### Error: Signature verification with public key of kty OKP is not supported by this method

Authentication responses may unexpectedly error out during verification. This appears as the throwing of a "Signature verification with public key of kty OKP is not supported" error from a call to `verifyAuthenticationResponse()` with the following stack trace:

```
Error: Signature verification with public key of kty OKP is not supported by this method
    at Object.verify (/xxx/node_modules/@simplewebauthn/server/script/helpers/iso/isoCrypto/verify.js:30:11)
    at verifySignature (/xxx/node_modules/@simplewebauthn/server/script/helpers/verifySignature.js:25:76)
    at verifyAuthenticationResponse (/xxx/node_modules/@simplewebauthn/server/script/authentication/verifyAuthenticationResponse.js:162:66)
```

This is caused by **security key responses in Firefox 118 and earlier being incorrectly composed by the browser** when the security key uses **Ed25519** for its credential keypair.

To fix this, update your call to `generateRegistrationOptions()` to exclude `-8` (Ed25519) from the list of algorithms:

```ts
const options = await generateRegistrationOptions({
  // ...
  supportedAlgorithmIDs: [-7, -257],
});
```

You will then need to re-register any authenticators that generated credentials that cause this error.

### ERROR extractStrings is not a function

Registration responses may unexpectedly error out during verification. This appears as the throwing of an "extractStrings is not a function" error from a call to `verifyRegistrationResponse()` with the following stack trace:

```
ERROR  extractStrings is not a function
    at readString (/node_modules/.pnpm/cbor-x@1.5.6/node_modules/cbor-x/dist/node.cjs:520:1)
    at read (/node_modules/.pnpm/cbor-x@1.5.6/node_modules/cbor-x/dist/node.cjs:343:1)
    at read (/node_modules/.pnpm/cbor-x@1.5.6/node_modules/cbor-x/dist/node.cjs:363:1)
    at checkedRead (/node_modules/.pnpm/cbor-x@1.5.6/node_modules/cbor-x/dist/node.cjs:202:1)
    at Encoder.decode (/node_modules/.pnpm/cbor-x@1.5.6/node_modules/cbor-x/dist/node.cjs:153:1)
    at Encoder.decodeMultiple (/node_modules/.pnpm/cbor-x@1.5.6/node_modules/cbor-x/dist/node.cjs:170:1)
    at Object.decodeFirst (/node_modules/.pnpm/@simplewebauthn+server@8.3.5/node_modules/@simplewebauthn/server/script/helpers/iso/isoCBOR.js:30:1)
    at decodeAttestationObject (/node_modules/.pnpm/@simplewebauthn+server@8.3.5/node_modules/@simplewebauthn/server/script/helpers/decodeAttestationObject.js:12:1)
    at verifyRegistrationResponse (/node_modules/.pnpm/@simplewebauthn+server@8.3.5/node_modules/@simplewebauthn/server/script/registration/verifyRegistrationResponse.js:100:1)
    at AuthnService.verifyRegistrationResponse (/home/deploy/mx/modules/authn/authn.service.js:89:1)
```

This is caused by the `@vercel/ncc` dependency not supporting runtime use of `require()` within other third-party packages used by the project, like **@simplewebauthn/server**'s use of **cbor-x**.

To fix this, add `CBOR_NATIVE_ACCELERATION_DISABLED=true` in your project's env file to disable the use of `require()` in **cbor-x**.

**Alternatively**, the following can be added to your project to inject this value into your project's runtime environment:

```js
function nodeEnvInjection() {
  /**
   * `@vercel/ncc` does not support the use of `require()` so disable its
   * use in the `@simplewebauthn/server` dependency called `cbor-x`.
   *
   * https://github.com/kriszyp/cbor-x/blob/master/node-index.js#L10
   */
  process.env['CBOR_NATIVE_ACCELERATION_DISABLED'] = 'true';
}

// Call this at the start of the project, before any imports
nodeEnvInjection()
```
