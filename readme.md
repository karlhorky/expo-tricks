# Expo Tricks

A collection of useful Expo tricks

## Load Node.js Native Addons in API Routes and Server Exports

[Expo Router API Routes cannot import Node.js native addons](https://github.com/expo/expo/issues/32350), eg. `@node-rs/bcrypt`.

To load native addons and include their binaries in server exports, apply pnpm patches for `@expo/cli@58.1.18` and `@expo/metro-config@58.0.20`, based on the workaround in [Expo PR #53246](https://github.com/expo/expo/pull/53246):

```bash
mkdir -p patches
curl --fail --location https://raw.githubusercontent.com/karlhorky/expo-tricks/main/load-node-js-native-addons-in-api-routes-and-server-exports/patches/@expo__cli@58.1.18.patch --output patches/@expo__cli@58.1.18.patch
curl --fail --location https://raw.githubusercontent.com/karlhorky/expo-tricks/main/load-node-js-native-addons-in-api-routes-and-server-exports/patches/@expo__metro-config@58.0.20.patch --output patches/@expo__metro-config@58.0.20.patch
```

Register the patches:

`pnpm-workspace.yaml`

```yaml
patchedDependencies:
  '@expo/cli@58.1.18': patches/@expo__cli@58.1.18.patch
  '@expo/metro-config@58.0.20': patches/@expo__metro-config@58.0.20.patch
```

Install dependencies to apply the patches:

```bash
pnpm install
```

- Use `"type": "commonjs"` in `package.json`, or omit `type`. This workaround does not fix [Expo's separate `"type": "module"` export issue](https://github.com/expo/expo/issues/52690).
- Deploy to a Node.js runtime. Edge runtimes cannot load `.node` files.
- Install or compile the native addon for the deployment server's operating system and CPU architecture. A binary built for macOS cannot run on Linux.

Code and reproduction: https://github.com/karlhorky/repro-expo-api-routes-node-rs-bcrypt-native-addons

## Protect API Routes from Cross-Site Request Forgery (CSRF)

**Update:** Starting in Expo &lt;unreleased version> (PR [`expo/expo#53115`](https://github.com/expo/expo/pull/53115)), API Routes have built-in protection against cross-origin form submissions.

[Expo Router API Routes do not reject cross-site form submissions by default](https://github.com/expo/expo/issues/53058), allowing attacks such as login CSRF.

To reject cross-site write requests, check the `Origin` host in [server middleware](https://docs.expo.dev/router/web/middleware/):

`src/app/+middleware.ts`

```ts
// Security: Reject write requests from other websites (CSRF protection)
// to prevent attacks like unwanted account changes or login CSRF
//
// Expo Router API Routes do not yet have built-in CSRF protection
// - https://github.com/expo/expo/issues/53058

export const unstable_settings = {
  matcher: {
    // Skip screens and other requests with safe methods
    methods: ['POST', 'PUT', 'PATCH', 'DELETE'],
  },
};

export default function middleware(request: Request) {
  const contentType = request.headers
    .get('content-type')
    ?.split(';', 1)[0]
    ?.trim()
    .toLowerCase();

  if (
    contentType &&
    ![
      'application/x-www-form-urlencoded',
      'multipart/form-data',
      'text/plain',
    ].includes(contentType)
  ) {
    return;
  }

  const origin = request.headers.get('origin');

  // Allow non-browser clients without Origin
  if (origin === null) return;

  if (
    !URL.canParse(origin) ||
    new URL(origin).host !==
      (request.headers.get('x-forwarded-host') || request.headers.get('host'))
  ) {
    return new Response('Cross-site request forbidden', { status: 403 });
  }
}
```

- Reproduction: https://github.com/karlhorky/repro-expo-router-api-routes-no-csrf-protection
