# Expo Tricks

A collection of useful Expo tricks

## Protect API Routes from Cross-Site Request Forgery (CSRF)

**Update:** [@kitten](https://github.com/kitten) added built-in protection against cross-origin form submissions in [Expo PR #53115](https://github.com/expo/expo/pull/53115), merged on November 10, 2026 for SDK 58. As of November 10, 2026, the latest published `expo-server` version is `58.0.6` and does not include the fix. Once released, prefer the built-in protection for this case. The workaround below remains useful for older versions.

[Expo Router API Routes accept cross-site form submissions without CSRF protection](https://github.com/expo/expo/issues/53058). A form on another website can submit JSON as `Content-Type: text/plain` without a CORS preflight, including to a login API Route which replaces the visitor's session with the attacker's account (login CSRF).

To reject these cross-site write requests, compare the `Origin` host with the request host in [server middleware](https://docs.expo.dev/router/web/middleware/):

`src/app/+middleware.ts`

```ts
// Security: Reject write requests from other websites (CSRF protection)
// to prevent attacks like unwanted account changes or login CSRF
//
// Expo Router API Routes do not yet have built-in CSRF protection
// - https://github.com/expo/expo/issues/53058

// eslint-disable-next-line @typescript-eslint/naming-convention -- Expo Router requires this export name
export const unstable_settings = {
  matcher: {
    // Skip screens and other requests with safe methods
    methods: ["POST", "PUT", "PATCH", "DELETE"],
    patterns: [
      "/api",
      "/api/[...path]",
      // Expo also serves auth API routes under /(auth)/api
      "/(auth)/api/[...path]",
    ],
  },
};

export default function middleware(request: Request) {
  const origin = request.headers.get("origin");

  // Allow non-browser clients without Origin
  if (origin === null) return;

  if (
    !URL.canParse(origin) ||
    new URL(origin).host !==
      (request.headers.get("x-forwarded-host") || request.headers.get("host"))
  ) {
    return new Response("Cross-site request forbidden", { status: 403 });
  }
}
```

This matches API Routes under `/api` and `/(auth)/api`. Adjust `patterns` if your API Routes use other paths. Requests with safe methods or without an `Origin` header are allowed, eg. native apps, server-to-server requests and webhooks.

The check compares hosts (including ports), not protocols. Behind a reverse proxy, make sure the proxy sets `x-forwarded-host` to the public host and overwrites any value supplied by the client.

The built-in protection in @kitten's PR checks form-compatible requests across all routes, using `Sec-Fetch-Site` with an `Origin` fallback. EAS Hosting and the development server skip that fallback. Unlike the middleware above, it allows requests with non-form content types such as `application/json`. It does not replace authentication or application-specific CORS rules.

- Code: [Next.js and Expo Router API origin checks](https://gist.github.com/karlhorky/61f751daf34799caee18f4f6d895ee29)
- Reproduction: https://github.com/karlhorky/repro-expo-router-api-routes-no-csrf-protection

## Load Node.js Native Addons in API Routes and Server Exports

[Importing Node.js native addons in Expo Router API Routes fails](https://github.com/expo/expo/issues/32350), eg. importing `@node-rs/bcrypt`:

`src/app/hash+api.ts`

```ts
import { hashSync } from "@node-rs/bcrypt";

export function GET() {
  return Response.json({ hash: hashSync("password", 4) });
}
```

Metro first fails to resolve the package's optional WASI fallback:

```text
Unable to resolve "./bcrypt.wasi.cjs" from "node_modules/.pnpm/@node-rs+bcrypt@2.0.0/node_modules/@node-rs/bcrypt/binding.js"
```

Adding `node` to Metro's `sourceExts` makes Metro parse the binary as JavaScript. Adding it to `assetExts` returns an asset URL instead of the native exports.

To load native addons with Node.js and include the binaries in server exports, patch Expo CLI and Expo Metro config with the prototype from [Expo PR #53246](https://github.com/expo/expo/pull/53246). The PR is still a draft as of November 10, 2026, pending review of the approach and upcoming serializer changes.

The [tested reproduction](https://github.com/karlhorky/repro-expo-api-routes-node-rs-bcrypt-native-addons) includes four compiled replacement files for `@expo/cli@58.1.18` and `@expo/metro-config@58.0.20`. Download the pinned files and their Expo MIT license:

```bash
(
set -e
mkdir -p expo-native-addons/cli-patch expo-native-addons/metro-patch

for file in cli-patch/MetroBundlerDevServer.js cli-patch/nativeNodeAddons.js cli-patch/withMetroMultiPlatform.js metro-patch/collect-dependencies.js LICENSE.expo; do
  curl --fail --location "https://raw.githubusercontent.com/karlhorky/repro-expo-api-routes-node-rs-bcrypt-native-addons/4f1d9cc854be205bd10693d9354afdd74818195b/$file" --output "expo-native-addons/$file"
done
)
```

Copy the replacement files into the installed packages:

`scripts/patch-expo-native-addons.cjs`

```js
const fs = require("node:fs");
const path = require("node:path");

const cliRoot = path.dirname(
  require.resolve("@expo/cli/package.json", {
    paths: [path.dirname(require.resolve("expo/package.json"))],
  }),
);
const metroRoot = path.dirname(
  require.resolve("@expo/metro-config/package.json", {
    paths: [cliRoot],
  }),
);

if (
  require(path.join(cliRoot, "package.json")).version !== "58.1.18" ||
  require(path.join(metroRoot, "package.json")).version !== "58.0.20"
) {
  throw new Error(
    "These replacement files require @expo/cli@58.1.18 and @expo/metro-config@58.0.20",
  );
}

for (const filename of [
  "MetroBundlerDevServer.js",
  "nativeNodeAddons.js",
  "withMetroMultiPlatform.js",
]) {
  fs.copyFileSync(
    path.join("expo-native-addons/cli-patch", filename),
    path.join(cliRoot, "build/src/start/server/metro", filename),
  );
}

fs.copyFileSync(
  "expo-native-addons/metro-patch/collect-dependencies.js",
  path.join(metroRoot, "build/transform-worker/collect-dependencies.js"),
);
```

Run the script after dependency installation and before starting Expo or exporting the app:

```bash
node scripts/patch-expo-native-addons.cjs
pnpm expo export --platform web --clear
```

The replacements let Node.js load `.node` files, copy the binaries into server exports, and recognize optional imports inside nested `try` blocks. No `metro.config.js` changes are required. Re-run the script after reinstalling dependencies, including during deployment builds.

The deployment server must support Node.js native addons, and the binaries must match its operating system and architecture. The prototype was tested with CommonJS package settings. It does not resolve [Expo's separate `type: module` server export issue](https://github.com/expo/expo/issues/52690).

- Code and reproduction: https://github.com/karlhorky/repro-expo-api-routes-node-rs-bcrypt-native-addons
- [Local development and moved server export results](https://github.com/karlhorky/repro-expo-api-routes-node-rs-bcrypt-native-addons/blob/4f1d9cc854be205bd10693d9354afdd74818195b/README.md?plain=1#L51-L84)
- [Vercel request results](https://github.com/karlhorky/repro-expo-api-routes-node-rs-bcrypt-native-addons/blob/4f1d9cc854be205bd10693d9354afdd74818195b/logs/vercel-requests.txt#L1-L30)
