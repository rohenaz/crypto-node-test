## Reproducing the Error

1.  **Clone the repository:**
    ```bash
    git clone <repository_url> # Replace <repository_url> with the actual URL
    cd crypto-node-test
    ```
2.  **Install dependencies:**
    ```bash
    npm install
    ```
3.  **Run the test script (this will also build the project):**
    ```bash
    npm run test:node
    ```
    *(Note: The exact file paths in the stack trace will reflect your local machine\'s path.)*

    You should see an error message printed to the console (the script will still exit with code 0 because the error is caught):
    ```
    Error: No secure random number generator is available...
    ```

## Details

The Node.js version used during this test was v22.2.0.

### Explanation of the Issue

The `@bsv/sdk`\'s `Random.ts` module attempts to detect the environment to choose a random number generator.
- It checks `typeof self === \'object\'`.
- If `self` is an object, it tries to use `self.crypto.getRandomValues` (Web Crypto API).
- If `self` is not an object (or the previous check fails in a way not covered), it tries `require(\'crypto\').randomBytes` (Node.js API).

In a standard Node.js ESM environment (`"type": "module"` in `package.json`), `self` is not typically defined globally by default in the same way it is in browsers. If some aspect of the environment (or a polyfill) defines `self` but `self.crypto.getRandomValues` is not present or functional, the SDK\'s current logic in `Random.ts` erroneously takes a path that results in it throwing an error ("No secure random number generator..."), without falling back to the Node.js native `require(\'crypto\').randomBytes`.

This test case demonstrates this failure mode when `PrivateKey.fromRandom()` is called.

### Bun Workaround (Demonstrates Successful Execution)

To see the core functionality working correctly (i.e., private key generation and token creation succeeding without the Node.js-specific random number generator error), you can run the test directly with Bun. Bun\'s environment provides a more browser-compatible `self` object.

```bash
npm run test:bun
```
This command should complete without any errors, printing the generated private key and auth token.

### Possible Solutions for the SDK

The underlying issue in the `@bsv/sdk`\'s `Random.ts` module could be addressed by enhancing its environment detection logic. For instance:

1.  **More Specific Checks:** Before attempting to use `self.crypto.getRandomValues`, the SDK could verify not only `typeof self === \'object\'` but also explicitly check for the existence and type of `self.crypto` and `self.crypto.getRandomValues`.
    ```javascript
    if (typeof self === \'object\' && self.crypto && typeof self.crypto.getRandomValues === \'function\') {
      // Use self.crypto.getRandomValues
    } // ... else fall back
    ```
2.  **Error-Safe Fallback:** Wrap the call to `self.crypto.getRandomValues` in a try-catch block. If an error occurs, then fall back to using `require(\'crypto\').randomBytes`.
    ```javascript
    try {
      // Attempt to use self.crypto.getRandomValues
    } catch (e) {
      // Fall back to require(\'crypto\').randomBytes
    }
    ```
3.  **Prioritize Node.js Native Crypto:** In a Node.js environment (detectable via `typeof process !== \'undefined\' && process.versions && process.versions.node`), the SDK could prioritize `require(\'crypto\').randomBytes` or ensure it\'s a more robustly utilized fallback if the Web Crypto API path fails.

These approaches could make the SDK more resilient to variations in JavaScript environments, particularly in Node.js ESM.