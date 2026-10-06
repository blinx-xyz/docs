# Signing a request from the CLI

A self-contained command-line signer for the Blinx Merchant API. It depends only on Node's `crypto`, so you can run it without the rest of the service. It prints the `x-api-key` / `x-timestamp` / `x-signature` headers (and optionally a ready-to-run `curl`) for any request.

### Usage

```bash
ts-node scripts/sign-payment-request.ts \
  --api-key <KEY> --api-secret <SECRET> \
  --method POST --url /api/merchant/payments \
  --body '{"amount":"5000","fiat":"NGN","token":"USDC","country":"NG","reference":"abc","purpose":"seize global content","type":"other"}'

# body from a file
ts-node scripts/sign-payment-request.ts --method POST --url /api/merchant/payments --body-file ./payload.json

# body from stdin
echo '{"amount":100}' | ts-node scripts/sign-payment-request.ts --method POST --url /api/merchant/payments --body-stdin
```

* Credentials may also come from env: `BLINX_API_KEY`, `BLINX_API_SECRET`.
* Add `--curl` (with `--base-url` or `BLINX_BASE_URL`) to also print a runnable `curl`.
* Add `--json` to emit the headers as JSON.

### Notes

* The signature covers the **exact** body bytes. Send the body verbatim — do not re-serialize or pretty-print it after signing, or the signature won't match.
* `--timestamp` defaults to now (unix epoch milliseconds) and must be within ±5 minutes of server time.

### Script

```ts
#!/usr/bin/env ts-node
/**
 * Blinx payments request signer.
 *
 * Produces the x-api-key / x-timestamp / x-signature headers that the
 * authenticateSeller middleware (src/middlewares/authenticateSeller.ts) expects
 * on /api/payments. Self-contained — depends only on Node's `crypto`, so a
 * merchant can run it without the rest of the service.
 *
 * Signature = hex HMAC-SHA256, keyed by the apiSecret, over the canonical string
 *
 *     `${timestamp}.${METHOD}.${url}.${sha256hex(body)}`
 *
 *   - timestamp : unix epoch MILLISECONDS (must be within ±5 min of server time)
 *   - METHOD    : upper-cased HTTP method
 *   - url       : the exact request target (path + query) as sent, e.g. /api/payments/123/status
 *   - body      : the raw request bytes exactly as sent (empty for bodyless requests)
 *
 * IMPORTANT: the signature covers the EXACT body bytes. Send the body verbatim —
 * do not re-serialize/pretty-print it after signing, or the signature won't match.
 *
 * Usage:
 *   ts-node scripts/sign-payment-request.ts \
 *     --api-key <KEY> --api-secret <SECRET> \
 *     --method POST --url /api/payments \
 *     --body '{"amount":100,"currency":"USDC"}'
 *
 *   # body from a file
 *   ts-node scripts/sign-payment-request.ts --method POST --url /api/payments --body-file ./payload.json
 *
 *   # body from stdin
 *   echo '{"amount":100}' | ts-node scripts/sign-payment-request.ts --method POST --url /api/payments --body-stdin
 *
 * Credentials may also come from env: BLINX_API_KEY, BLINX_API_SECRET.
 * Add --curl to also print a ready-to-run curl command (needs --base-url or BLINX_BASE_URL).
 */

import {createHash, createHmac} from 'crypto';
import {readFileSync} from 'fs';

interface Args {
  apiKey?: string;
  apiSecret?: string;
  method: string;
  url?: string;
  body: Buffer;
  timestamp: string;
  baseUrl?: string;
  curl: boolean;
  json: boolean;
}

function fail(message: string): never {
  process.stderr.write(`error: ${message}\n\nRun with --help for usage.\n`);
  process.exit(1);
}

function printHelp(): void {
  // The block comment above is the reference; keep this short.
  process.stdout.write(
    `Blinx payments request signer\n\n` +
      `Required: --url <path>  and credentials (--api-key/--api-secret or BLINX_API_KEY/BLINX_API_SECRET)\n` +
      `Options:  --method <M> (default GET), --body <json> | --body-file <path> | --body-stdin,\n` +
      `          --timestamp <epoch-ms> (default now), --curl, --base-url <url>, --json\n`,
  );
}

function parseArgs(argv: string[]): Args {
  const args: Args = {
    method: 'GET',
    body: Buffer.alloc(0),
    timestamp: String(Date.now()),
    curl: false,
    json: false,
    apiKey: process.env.BLINX_API_KEY,
    apiSecret: process.env.BLINX_API_SECRET,
    baseUrl: process.env.BLINX_BASE_URL,
  };
  let bodyInline: string | undefined;
  let bodyFile: string | undefined;
  let bodyStdin = false;

  for (let i = 0; i < argv.length; i++) {
    const arg = argv[i];
    const next = () => {
      const v = argv[++i];
      if (v === undefined) fail(`missing value for ${arg}`);
      return v;
    };
    switch (arg) {
      case '-h':
      case '--help': printHelp(); process.exit(0); break;
      case '--api-key': args.apiKey = next(); break;
      case '--api-secret': args.apiSecret = next(); break;
      case '--method': args.method = next(); break;
      case '--url':
      case '--path': args.url = next(); break;
      case '--body': bodyInline = next(); break;
      case '--body-file': bodyFile = next(); break;
      case '--body-stdin': bodyStdin = true; break;
      case '--timestamp': args.timestamp = next(); break;
      case '--base-url': args.baseUrl = next(); break;
      case '--curl': args.curl = true; break;
      case '--json': args.json = true; break;
      default: fail(`unknown argument: ${arg}`);
    }
  }

  const bodySources = [bodyInline !== undefined, bodyFile !== undefined, bodyStdin].filter(Boolean).length;
  if (bodySources > 1) fail('use only one of --body, --body-file, --body-stdin');
  if (bodyInline !== undefined) args.body = Buffer.from(bodyInline, 'utf8');
  else if (bodyFile !== undefined) args.body = readFileSync(bodyFile);
  else if (bodyStdin) args.body = readFileSync(0); // fd 0 = stdin

  if (!args.apiKey) fail('missing apiKey (pass --api-key or set BLINX_API_KEY)');
  if (!args.apiSecret) fail('missing apiSecret (pass --api-secret or set BLINX_API_SECRET)');
  if (!args.url) fail('missing --url (the request path, e.g. /api/payments)');
  if (!/^\//.test(args.url)) fail('--url must be a path starting with "/" (path + query as sent)');
  if (!/^\d+$/.test(args.timestamp)) fail('--timestamp must be unix epoch milliseconds');

  return args;
}

// POSIX single-quote escaping: wrap in '...' and rewrite each ' as '\''.
// Preserves every byte literally (newlines, quotes) when pasted into a shell.
function shellSingleQuote(s: string): string {
  return `'${s.replace(/'/g, `'\\''`)}'`;
}

function sign(args: Args): {timestamp: string; signature: string; canonical: string} {
  const method = args.method.toUpperCase();
  const bodyHash = createHash('sha256').update(args.body).digest('hex');
  const canonical = `${args.timestamp}.${method}.${args.url}.${bodyHash}`;
  const signature = createHmac('sha256', args.apiSecret!).update(canonical).digest('hex');
  return {timestamp: args.timestamp, signature, canonical};
}

function main(): void {
  const args = parseArgs(process.argv.slice(2));
  const {timestamp, signature, canonical} = sign(args);

  if (args.json) {
    process.stdout.write(
      JSON.stringify(
        {
          headers: {'x-api-key': args.apiKey, 'x-timestamp': timestamp, 'x-signature': signature},
          method: args.method.toUpperCase(),
          url: args.url,
          canonical,
        },
        null,
        2,
      ) + '\n',
    );
    return;
  }

  const out: string[] = [];
  out.push('Signed request headers:');
  out.push(`  x-api-key:   ${args.apiKey}`);
  out.push(`  x-timestamp: ${timestamp}`);
  out.push(`  x-signature: ${signature}`);
  out.push('');
  out.push(`canonical:     ${canonical}`);

  if (args.curl) {
    if (!args.baseUrl) fail('--curl requires --base-url (or BLINX_BASE_URL), e.g. http://localhost:3001');
    const base = args.baseUrl.replace(/\/$/, '');
    const parts = [
      `curl -X ${args.method.toUpperCase()} '${base}${args.url}'`,
      `  -H 'x-api-key: ${args.apiKey}'`,
      `  -H 'x-timestamp: ${timestamp}'`,
      `  -H 'x-signature: ${signature}'`,
    ];
    if (args.body.length > 0) {
      parts.push(`  -H 'content-type: application/json'`);
      // Emit the exact signed bytes in POSIX single quotes so the shell passes
      // them verbatim — real newlines and quotes are preserved. JSON.stringify
      // would emit literal \n / \" that bash does NOT expand inside double
      // quotes, changing the bytes and breaking the signature. --data-raw sends
      // the argument as-is (no @-file interpretation, no newline stripping).
      parts.push(`  --data-raw ${shellSingleQuote(args.body.toString('utf8'))}`);
    }
    out.push('');
    out.push('curl:');
    out.push(parts.join(' \\\n'));
  }

  process.stdout.write(out.join('\n') + '\n');
}

main();
```

_Updated: 27 August 2026 12:46_
