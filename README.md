# ServiceNow IP Lookup — explained simply

> **In one sentence:** this app asks a website called ipinfo.io *"where is the IP address 8.8.8.8?"*
> and shows you the answer.

It is a small ServiceNow app (`IPLookup`) that demonstrates the most common integration task there
is: **ServiceNow calling somebody else's website and reading the reply.**

There is no login, no server to install, and no password. If you can read a web address and run a
script, you can follow this.

Part of [eltonaguiar/ServiceNow_Portfolio](https://github.com/eltonaguiar/ServiceNow_Portfolio)
(entry **#7**).

## Contents

1. [Start here: the 60-second version](#start-here-the-60-second-version)
2. [Fundamentals: why call a website at all, and why this way?](#fundamentals-why-call-a-website-at-all-and-why-this-way)
   - [What else could you use instead?](#what-else-could-you-use-instead)
   - [When is this the wrong tool?](#when-is-this-the-wrong-tool)
3. [What is happening, in everyday terms](#what-is-happening-in-everyday-terms)
4. [Why the address has a blank in it](#why-the-address-has-a-blank-in-it)
5. [What the answer looks like](#what-the-answer-looks-like)
6. [What is actually stored in the app](#what-is-actually-stored-in-the-app)
7. [The script, explained line by line](#the-script-explained-line-by-line)
8. [Real code: three complete examples](#real-code-three-complete-examples)
9. [Run the original Fix Script yourself](#run-the-original-fix-script-yourself)
10. [If it does not work](#if-it-does-not-work)
11. [Prove it is the website, not ServiceNow](#prove-it-is-the-website-not-servicenow)
12. [Build it yourself in four steps](#build-it-yourself-in-four-steps)
13. [Words you will bump into](#words-you-will-bump-into)
14. [What is in this repo](#what-is-in-this-repo)
15. [Which branch to read](#which-branch-to-read)
16. [Two things not to do](#two-things-not-to-do)
17. [Appendix: the three ways in and out](#appendix-the-three-ways-in-and-out-and-tying-this-to-a-real-database)
    - [A1. What a REST Message actually gives you](#a1-what-a-rest-message-actually-gives-you)
    - [A2. Your Toronto events database](#a2-making-it-practical-your-toronto-events-database)
    - [A3. Scripted REST API lab](#a3-scripted-rest-api--the-lab-servicenow-serves)
    - [A4. SOAP lab](#a4-soap--the-lab)
    - [A5. Side by side](#a5-side-by-side)
    - [A6. What triggers what](#a6-what-triggers-what-and-why)
    - [A7. Debug checklist](#a7-one-debug-checklist-for-all-three)

---

## Start here: the 60-second version

| Question | Short answer |
|---|---|
| What does it do? | Visits `https://ipinfo.io/8.8.8.8/json` and shows what comes back |
| Where is the code? | Fix Script **Elton Test Integration** |
| Where is the web address stored? | In a REST Message called **`IPInfo`** |
| Why `${ipaddress}` in the address? | It is a blank that gets filled in with `8.8.8.8` |
| How do I see it work? | Run the Fix Script — a pop-up shows the answer |
| Do I need a MID Server? | No. ServiceNow makes the call itself |

See the answer for yourself before reading any code — paste this into a browser:

<https://ipinfo.io/8.8.8.8/json>

---

## Fundamentals: why call a website at all, and why this way?

### First, why would ServiceNow talk to an outside site?

Because the answer you need does not live in ServiceNow. Common reasons:

- **Fill in information** — an IP address arrives, you want the city and company behind it.
- **Tell somebody** — a ticket is created, a chat channel should hear about it.
- **Check on something** — is that server still up? did that payment go through?
- **Get a list** — pull today's orders from the finance system so they can be worked on here.

All four are the same shape: *ask an outside computer a question, read the answer, do something with
it*. That shape has a name — a **REST call** — and this app is the smallest working example of it.

### Then why a saved REST Message, rather than just writing the web address in code?

Because you get a record an administrator can look at, instead of a URL buried inside a script.

| Written straight into code | Saved as a REST Message (this app) |
|---|---|
| only a developer can find or change it | sits in a menu, searchable, editable by an admin |
| password or key ends up in the script | auth lives on the record (basic, OAuth, token) |
| every script writes its own version | one definition, reused by many scripts |
| the blank to fill in is improvised | named variables, visible in one place |
| changing the address means editing code | change the record, no code change |
| hard to see what talks to the outside world | you can list every REST Message the instance has |

`RESTMessageV2` is the piece of script that *uses* that saved record. So the split is: **the record
holds the configuration, the script holds the logic.** That is why the same two lines work from a Fix
Script, a Business Rule or a scheduled job — the details live in the record.

### What else could you use instead?

ServiceNow has several ways to reach another system. They are not rivals so much as different tools:

| Approach | What it looks like | Good when | Watch out for |
|---|---|---|---|
| **Saved REST Message + `RESTMessageV2`** (this app) | a record plus two lines of script | you want one shared, admin-visible definition; auth profiles; reuse | you maintain a record; the script runs while the user waits |
| **`RESTMessageV2` with a dynamic address** (`setEndpoint()`) | same class, address built in code | the URL is not known in advance (per-customer hosts, signed links) | nothing is visible in the menus any more — back to a URL in code |
| **Flow Designer / Integration Hub action** | drag a REST step, no script | non-developers should own the automation; you want retries and error branches | may need a paid entitlement; harder to unit-test than script |
| **A ready-made spoke** (Teams, Slack, SAP, Workday…) | pick the action, fill the form | a supported connector already exists | the connector decides what the call can do |
| **MID Server routing** (`Use MID Server` on the REST Message) | same REST Message, ticked differently | the site is inside your network and ServiceNow cannot reach it | needs a healthy MID Server |
| **Scripted REST API** | you *publish* an address others call | the direction is the other way round — outsiders calling in | nothing to do with this app; it is the mirror image |
| **JDBC data source / export sets / Remote Table** | database link, file drop, live table | bulk movement, or a BI tool that wants files | not for one-off questions; some need a MID Server |

Rule of thumb: **a question answered in a second, from a website → REST Message.** A nightly file or a
database link → one of the bottom row's tools. A process a business analyst should own → Flow Designer.

### When is this the wrong tool?

- **Thousands of calls in a row.** The user (or the job) waits for each reply. Use a scheduled job in
  batches (example 3 below) or a queue.
- **The reply must never be lost.** A plain script call has no retry. Flow Designer, or a table that
  records what still needs sending, gives you one.
- **A supported connector already exists.** Rebuilding the Teams or SAP call by hand is wasted effort.
- **The site is not reachable from ServiceNow.** Then add a MID Server rather than rewriting anything.

---

## What is happening, in everyday terms

Four pieces, and it helps to keep them apart:

| Piece | What it really is | Everyday comparison |
|---|---|---|
| **ipinfo.io** | a public website that knows where IP addresses are | the phone book |
| **REST Message `IPInfo`** | the saved web address ServiceNow will call | a phone number saved in your contacts |
| **Method `Default GET`** | the exact request, including the blank to fill in | "call this number and ask for a lookup" |
| **`RESTMessageV2`** | the script that presses "call" | the phone you pick up |

The whole journey:

```
You run the script
      │
      ▼
"Use REST Message x_1569490_iplookup.IPInfo, method Default GET"
      │
      ▼
Address becomes https://ipinfo.io/8.8.8.8/json     ← the blank is filled in
      │
      ▼
ipinfo.io replies: {"ip":"8.8.8.8","city":"Mountain View", ...}
      │
      ▼
The script puts that reply on the screen
```

Nothing is hidden: the app stores *what to call*, the script *makes the call*, and one small piece
*changes each time*.

---

## Why the address has a blank in it

Because `https://ipinfo.io/${ipaddress}/json` is a **blank in a sentence**, not a finished address.

`https://ipinfo.io/${ipaddress}/json` cannot be typed into a browser and expected to work. The
`${ipaddress}` part is a placeholder. When the script runs, ServiceNow replaces it with whatever
value you gave for something called `ipaddress`.

```
What is written today:   https://ipinfo.io/${ipaddress}/json
What gets filled in:     ipaddress = 8.8.8.8
What is actually called: https://ipinfo.io/8.8.8.8/json
```

Why not just hard-code 8.8.8.8 in the address? Because then the app could only ever look up that one
address. With a blank, the same saved REST Message can look up **any** address — one from a record,
from someone typing it in, from a list — by changing a single line in the script.

| If you set the blank to | ServiceNow calls |
|---|---|
| `8.8.8.8` | `https://ipinfo.io/8.8.8.8/json` |
| `1.1.1.1` | `https://ipinfo.io/1.1.1.1/json` |

Two small rules:

1. **The name has to match on both sides.** There are two places, and both must say `ipaddress`:

   | Side | Exact location | What you should see |
   |---|---|---|
   | the address | REST Message `IPInfo` → **HTTP Methods** tab → open `Default GET` → **Endpoint** field | `https://ipinfo.io/${ipaddress}/json` |
   | the definition | the **same** `Default GET` form → scroll to the **Variable Substitutions** related list at the bottom → the row's **Name** field | `ipaddress` (with `8.8.8.8` in **Value**) |

   In menu terms: **All → System Web Services → Outbound → REST Message → `IPInfo`**, open the method,
   and look at the two areas above. In URL terms: `sys_rest_message_list.do` for the message,
   `sys_rest_message_fn_list.do` for the methods, `sys_rest_message_fn_parameters_list.do` for the
   variables. If the names differ by even one letter — or the Variable Substitutions list is empty —
   ServiceNow sends the text `${ipaddress}` to ipinfo.io instead of an address, and ipinfo.io politely
   replies that it is not an IP address.
2. **There is a "fill in exactly as written" version** of the fill-in command
   (`setStringParameterNoEscape`). The other version tidies your value for use inside a web address
   (turning spaces into `%20`). For something like `8.8.8.8` either works, and the app uses the
   "exactly as written" one.

### Where does the name `ipaddress` come from, exactly?

It is not magic and it is not built in — **somebody typed it**, and you can see the typing here:

**Breadcrumb:** All → System Web Services → Outbound → REST Message → `IPInfo` → **HTTP Methods**
tab → open `Default GET` → scroll to the **Variable Substitutions** related list. (Faster: put
`sys_rest_message_fn_parameters_list.do` in the URL bar — one row per variable, filter by
*REST Message = IPInfo*.)

That list is the whole story. Each row is a name and a default value, and the endpoint above simply
mentions one of those names in `${...}` form:

| Piece | Where it lives | In this app |
|---|---|---|
| the text `${ipaddress}` | the method's **Endpoint** field | `https://ipinfo.io/${ipaddress}/json` |
| the variable itself | the method's **Variable Substitutions** list | Name `ipaddress`, Value `8.8.8.8` |
| the value used for one specific call | your script | `setStringParameterNoEscape('ipaddress', '1.1.1.1')` — overrides the default for that call only |
| what happens when no match is found | nowhere to substitute | the text `${ipaddress}` is sent to the website literally |

At `execute()` time ServiceNow scans the endpoint, the headers and the body for `${someName}` and
replaces each one with the value of the variable with that name. That is all the "template engine" is.

### "But how does the code *know* to put `8.8.8.8` in there?"

It does not know. It is **told** — in the second line of the script:

```javascript
rm.setStringParameterNoEscape('ipaddress', '8.8.8.8');
//                             ^^^^^^^^^    ^^^^^^^
//                             which blank  what to put in it
```

Two arguments: **the name of the blank**, and **the value for it**. That is the entire mechanism —
there is no inference, no looking at the reply, no guessing from the URL.

The `"ip": "8.8.8.8"` you see in the JSON is a different thing entirely: it is ipinfo **telling you
what it just received**. An echo, not a source. Look at the order of events — the reply comes last, so
it cannot have supplied the value:

```
1. script says:        ipaddress = 8.8.8.8
2. ServiceNow builds:  https://ipinfo.io/8.8.8.8/json
3. request goes out
4. reply comes back:   {"ip": "8.8.8.8", "city": "Mountain View", ...}
```

Three `8.8.8.8`s appear in this page, and they are three different things:

| Where it appears | What it is | Who put it there |
|---|---|---|
| `${ipaddress}` in the endpoint | a hole waiting to be filled | whoever built the REST Message |
| `'8.8.8.8'` in `setStringParameterNoEscape(...)` | the value for that hole, for this one call | your script |
| `"ip": "8.8.8.8"` in the reply | the API confirming what it received | ipinfo.io |

**Prove it in ten seconds.** Change one string in the script:

```javascript
rm.setStringParameterNoEscape('ipaddress', '1.1.1.1');
```

Run it again and the reply says `"ip": "1.1.1.1"` — because the value you handed to the blank changed,
so the URL changed, so the question changed. Nothing else was edited.

**And if the script never sets it?** The default on the Variable Substitutions row is used: `8.8.8.8`.
So an untouched app still works — that default is what makes `Elton Test Integration` runnable as
shipped.

### "Can someone just put whatever?"

Two different answers, because two different things are arbitrary:

**The name is arbitrary.** `ipaddress` was chosen by whoever built the app. You could name it `${thing}`
and, as long as the Variable Substitutions list has a row named `thing`, ServiceNow would happily fill
it in. The name is only a label shared between the URL and the list — it carries no meaning to
ServiceNow. (It does carry meaning to the *website*: `${thing}` in that slot still lands in the path
where ipinfo.io expects an IP, so a working blank with a silly name is still a broken lookup.)

**The value is arbitrary too — and nothing checks it.** The variable is text. Put `hello` in it and
ServiceNow builds `https://ipinfo.io/hello/json` and sends it; the *website* is the thing that decides
that is not an address. That is exactly why the reusable Script Include further down checks the shape
of an IPv4 address before spending a call, and why a failed lookup is reported as a failure instead of
being stored.

Two consequences worth knowing:

- **This is configuration, not code.** Anyone who can edit the REST Message (an admin, or a delegated
developer role) can change the endpoint, the variable names, or the authentication. Review it like you
would review a script — it decides where your data goes.
- **A variable is not a guard against bad input.** If the value comes from a field a user typed into,
  it can contain spaces, slashes or `?&=` characters, and it goes straight into a URL path. Validate it
  (the IPv4 check) or use the escaping version of the fill-in command, which encodes those characters.

---

## What the answer looks like

This is the real reply for `8.8.8.8`, which is Google's public DNS service:

```json
{
  "ip": "8.8.8.8",
  "hostname": "dns.google",
  "city": "Mountain View",
  "region": "California",
  "country": "US",
  "loc": "37.4056,-122.0775",
  "org": "AS15169 Google LLC",
  "postal": "94043",
  "timezone": "America/Los_Angeles",
  "readme": "https://ipinfo.io/missingauth",
  "anycast": true
}
```

In plain words:

| What you see | What it means |
|---|---|
| `"ip": "8.8.8.8"` | the address you asked about, repeated back so you can check it |
| `"hostname": "dns.google"` | the friendly name for that address |
| `"city"`, `"region"`, `"country"` | roughly where it is; `US` is the country code |
| `"loc": "37.4056,-122.0775"` | a map pin (latitude, longitude) — a neighbourhood, not a street address |
| `"org": "AS15169 Google LLC"` | who runs that network: the number is the network's ID, `Google LLC` is the company |
| `"postal"`, `"timezone"` | postcode guess and local time zone — handy for "what time is it there?" |
| `"readme": "…missingauth"` | a friendly note: *you did not use a free account key*. Fine for trying it out |
| `"anycast": true` | this one address is served from many places at once, which is why `8.8.8.8` is fast everywhere |

That `readme` line is not an error. It is ipinfo.io saying "you are welcome to test without signing
up, but if you call me a lot, please create a free key". Nothing in this app breaks because of it.

---

## What is actually stored in the app

| Stored thing | Its value here | Why it exists |
|---|---|---|
| the app | `IPLookup` (its own little namespace, `x_1569490_iplookup`) | keeps its names separate from everything else |
| the REST Message | `IPInfo` → `https://ipinfo.io/` | the saved web address |
| the method | `Default GET` → `https://ipinfo.io/${ipaddress}/json` | the exact request, with one blank |
| the variable | `ipaddress`, default `8.8.8.8` | the blank's name and its starting value |
| permissions ×6 | 4 access rules, 2 roles | what the app was allowed to do |
| the script | `Elton Test Integration` | the code that makes the call |

---

## The script, explained line by line

Every line commented, with what you have in hand once the line has run:

```javascript
// 1. PICK UP THE SAVED CALL. Nothing is sent yet.
//    'x_1569490_iplookup.IPInfo' = <app scope>.<REST Message name>
//    'Default GET'               = the method (a.k.a. function) inside that message
//    Have now: r = a prepared request. The URL still contains the blank,
//              https://ipinfo.io/${ipaddress}/json
var r = new sn_ws.RESTMessageV2('x_1569490_iplookup.IPInfo', 'Default GET');

// 2. FILL THE BLANK. First argument = which blank (must match the URL and the
//    Variable Substitutions row), second = the value.
//    "NoEscape" = send it exactly as written; the other version would URL-encode
//    the value (spaces become %20), which matters for free text, not for an IP.
//    Have now: a complete URL, https://ipinfo.io/8.8.8.8/json — still unsent.
r.setStringParameterNoEscape('ipaddress', '8.8.8.8');

// 3. SEND IT. The only line that touches the network, the only one that can be
//    slow, and the only one that can fail before any reply exists.
//    Have now: response = the whole reply (status code, headers, body).
//    Nothing has been read out of it yet.
var response = r.execute();

// 4. READ THE REPLY AS TEXT. It looks like an object on screen, but to
//    ServiceNow it is a string; you need JSON.parse() before you can ask for
//    responseBody.city.
//    Have now: responseBody = '{"ip":"8.8.8.8","hostname":"dns.google", ...}'
var responseBody = response.getBody();

// 5. SHOW IT. A Fix Script only displays something when the script reports a
//    problem, so we deliberately report the good answer. This is a testing
//    trick: real code logs it or writes it into a record instead.
//    Have now: a pop-up reading "Elton was here! Response: {...}"
throw "Elton was here! Response: " + responseBody;
```

### What you have after each line

| After | You have | You do **not** have yet |
|---|---|---|
| 1 | a prepared request object, pointing at a saved REST Message method | a value for the blank; no network activity has happened |
| 2 | a fully formed URL — read it back with `r.getEndpoint()` if you want proof | still nothing sent |
| 3 | the reply: `getStatusCode()`, `getHeaders()`, `getBody()`, `getErrorMessage()` | the body split into usable fields |
| 4 | the body as **text**, e.g. `{"ip":"8.8.8.8", …}` | `city`, `org`, `country` as things you can use — that needs `JSON.parse()` |
| 5 | the answer on screen | anything saved: a Fix Script writes nothing unless its own code does |

And what goes wrong at each stage:

| Line | What it is doing | What goes wrong |
|---|---|---|
| 1 | opens the saved REST Message and picks the method | name not scope-prefixed when called from another app's scope; misspelled method fails immediately |
| 2 | fills the blank with `8.8.8.8` | wrong name → the text `${ipaddress}` goes out literally; wrong value → ipinfo.io says it is not an address |
| 3 | actually visits the website (this is where the waiting happens) | no internet access from ServiceNow → status `0` or a timeout |
| 4 | grabs the reply as plain text | the reply can be an error page; text is not fields |
| 5 | puts the reply in the pop-up | a removed `throw` means the pop-up never appears |

### Why the script throws an "error" on purpose

A **Fix Script** is ServiceNow's "run this once and show me what happened" tool. The "show me" part
only appears when the script reports a problem — so the script reports the *good* answer on purpose,
purely to get it on the screen.

**This is a testing shortcut, not how you would write real code.** In something that runs all day
(a Business Rule, a Script Include, a scheduled job) you would save the answer into a record or a log,
and check first whether the call actually worked. The next section has three complete examples you can
copy.

---

## Real code: three complete examples

All three do the same job — call `IPInfo`, check whether it worked, then use the reply — and all three
are complete files, not fragments.

### Example 1 — a reusable Script Include

> **At a glance**
> - **You get:** one function, `lookup('8.8.8.8')`, callable from any server-side script, that always
>   returns the same shape — `{ ok, status, ip, city, country, org, loc, timezone }` on success,
>   `{ ok: false, error }` on any failure. It never throws, so one bad address cannot break a batch.
> - **You provide:** the Script Include record (name `IpInfoLookup`, *Accessible from: All application
>   scopes* only if other apps need it, **not** client-callable) and an IP address string.
> - **Why it is worth it:** the error handling is written once, not three times; changing ipinfo for
>   something else later means editing one file; every caller reports the same way.
> - **Still your job:** deciding what to do with a failure, and whether to retry. The include reports,
>   it does not act.

This is the piece to write once and call from everywhere else. Create a Script Include named
`IpInfoLookup`, **not** client-callable, and paste this in:

```javascript
var IpInfoLookup = Class.create();
IpInfoLookup.prototype = {

    initialize: function () {},

    /**
     * Look up one IP address.
     * Always returns an object, never throws, so the caller can decide what to do.
     *
     *   { ok: true,  status: 200, ip: '8.8.8.8', city: 'Mountain View', country: 'US',
     *     org: 'AS15169 Google LLC', loc: '37.4056,-122.0775', timezone: 'America/Los_Angeles' }
     *   { ok: false, status: 0,   error: 'timeout' }
     */
    lookup: function (ipAddress) {
        var result = { ok: false, status: 0, error: '' };

        if (!ipAddress) {
            result.error = 'no IP address supplied';
            return result;
        }

        // Cheap sanity check first: do not spend a web call on something
        // that cannot possibly be an address.
        var ipv4 = /^\d{1,3}(\.\d{1,3}){3}$/;
        if (!ipv4.test(ipAddress)) {
            result.error = 'not an IPv4 address: ' + ipAddress;
            return result;
        }

        try {
            var request = new sn_ws.RESTMessageV2('x_1569490_iplookup.IPInfo', 'Default GET');
            request.setStringParameterNoEscape('ipaddress', ipAddress);

            var response = request.execute();
            result.status = response.getStatusCode();

            // Status 0 means the call never made it out (DNS, network, timeout).
            if (result.status === 0) {
                result.error = 'could not reach ipinfo.io: ' + response.getErrorMessage();
                return result;
            }

            // Read the reply whatever the status is: on an error it still
            // contains the useful detail.
            var body = response.getBody() || '';

            if (result.status !== 200) {
                result.error = 'ipinfo.io replied ' + result.status + ': ' + body;
                return result;
            }

            var info;
            try {
                info = JSON.parse(body);
            } catch (parseError) {
                result.error = 'reply was not readable JSON: ' + body.substring(0, 200);
                return result;
            }

            if (info.error) {                // ipinfo reports a bad address this way
                result.error = 'lookup refused: ' +
                    (info.error.message || info.error.title || JSON.stringify(info.error));
                return result;
            }

            result.ok = true;
            result.ip = info.ip;
            result.city = info.city;
            result.country = info.country;
            result.org = info.org;
            result.loc = info.loc;
            result.timezone = info.timezone;
            return result;

        } catch (e) {
            result.error = 'unexpected failure: ' + (e.message || e);
            return result;
        }
    },

    type: 'IpInfoLookup'
};
```

Why it is shaped like this: every failure returns the same object with `ok: false`, so the caller never
has to check for `null`, and one bad address cannot break a whole batch.

### Example 2 — a Business Rule that fills in fields

> **At a glance**
> - **You get:** four fields filled in by themselves whenever someone saves a record with an address
>   (`u_city`, `u_country`, `u_org`, `u_geo`), a log line per lookup, and a warning instead of an error
>   when ipinfo is unreachable — the save still goes through.
> - **You provide:** a table with an `ip_address` field and those four fields, the Script Include from
>   example 1, and a rule on **before → insert and update**.
> - **Why it is worth it:** nobody has to press a button, and the "only if the address changed" guard
>   means a record edited ten times costs one call, not ten.
> - **Still your job:** knowing what a call costs. This runs while the user waits, so keep it off
>   high-volume tables and never let a slow website hold a form hostage.

Put this on a table with an `ip_address` field and somewhere to store the answer (`u_city`,
`u_country`, `u_org`). It runs only when the address is new or has changed, so you are not calling the
internet on every single save:

```javascript
(function executeRule(current, previous /* null when async */) {

    // Only when there is an address, and only when it changed.
    if (!current.ip_address) {
        return;
    }
    var changed = !previous ||
        previous.getValue('ip_address') != current.getValue('ip_address');
    if (!changed) {
        return;
    }

    var lookup = new IpInfoLookup();      // the Script Include from example 1
    var info = lookup.lookup(current.getValue('ip_address'));

    if (!info.ok) {
        gs.warn('[IpInfo BR] lookup failed for ' + current.ip_address + ': ' + info.error);
        return;                            // keep the record as it is; never block the save
    }

    current.u_city = info.city || '';
    current.u_country = info.country || '';
    current.u_org = info.org || '';
    current.u_geo = info.loc || '';

    gs.info('[IpInfo BR] ' + info.ip + ' → ' + info.city + ', ' + info.country);

})(current, previous);
```

Set the rule to **before** / **insert and update**. Two deliberate choices: a failed lookup logs a
warning instead of stopping the save (a website being down should never stop someone working), and the
`previous.ip_address != current.ip_address` guard keeps the call count low, because this call happens
while the user waits.

### Example 3 — a scheduled job that fills in a backlog

> **At a glance**
> - **You get:** an existing table of addresses filled in overnight, plus one log line saying
>   `updated=37 failed=2 skipped=11` — enough to tell "working" from "silently doing nothing".
> - **You provide:** the table and its field names, the Script Include, a schedule, and a batch size you
>   are comfortable with (50 here).
> - **Why it is worth it:** it is resumable by construction — it only selects rows where `u_city` is
>   still empty — so re-running it continues where it stopped instead of redoing work, and nobody waits
>   for it.
> - **Still your job:** staying inside the rate limit. Free ipinfo calls are throttled; 50 per run is
>   deliberate, and a bigger number will look like a broken job rather than a fast one.

Use this when you already have a table full of addresses with empty location columns. It runs on its
own, handles a batch, and logs a summary:

```javascript
(function fillMissingLocations() {

    var BATCH_LIMIT = 50;                  // stay gentle: 50 per run, not 5,000
    var TABLE = 'u_ip_registry';

    var gr = new GlideRecord(TABLE);
    gr.addQuery('ip_address', '!=', '');
    gr.addNullQuery('u_city');             // only the ones still empty
    gr.setLimit(BATCH_LIMIT);
    gr.query();

    var lookup = new IpInfoLookup();
    var done = 0, failed = 0, skipped = 0;

    while (gr.next()) {
        var info = lookup.lookup(gr.getValue('ip_address'));

        if (!info.ok) {
            failed++;
            gs.warn('[IpBackfill] ' + gr.getValue('ip_address') + ': ' + info.error);
            continue;
        }

        if (!info.city && !info.country) {    // private/internal address, nothing to find
            skipped++;
            continue;
        }

        gr.u_city = info.city || '';
        gr.u_country = info.country || '';
        gr.u_org = info.org || '';
        gr.u_geo = info.loc || '';
        gr.update();
        done++;
    }

    gs.info('[IpBackfill] updated=' + done + ' failed=' + failed + ' skipped=' + skipped);

})();
```

Schedule it daily at a quiet hour. The `setLimit` matters: free ipinfo.io calls are rate limited, and a
job that tries thousands at once gets throttled and looks broken. Private addresses (`10.x`, `192.168.x`)
are not on the internet at all, which is what `skipped` counts.

### What the API gives you, and what it does not

Worth being blunt about, because it decides how much of the three examples you actually need:

| ipinfo.io gives you | ipinfo.io does **not** give you |
|---|---|
| `ip`, `hostname` — what the address is called | a street address, or a person |
| `city`, `region`, `country`, `postal`, `loc` — approximate place | certainty: VPNs, mobile networks and corporate proxies all lie |
| `org` — the network owner (ASN + company) | a guarantee it answered: it can be slow, rate limited, or down |
| `timezone` | anything about addresses **inside** your network (`10.x`, `192.168.x`) — those are `bogon`, not on the internet |

So the sensible division of labour is: **ipinfo answers; your code remembers.** Store the answer on the
record the first time you get it (what examples 2 and 3 do), and only ask again when the address
changes. That one habit removes most of the rate-limit and latency problems before they start.

### Which one do I need?

| You want to… | Use |
|---|---|
| look an address up from several places | the **Script Include** (example 1) — write it once |
| fill fields while a person is editing a record | the **Business Rule** (example 2), with the "only if changed" guard |
| catch up an existing table of addresses | the **Scheduled Job** (example 3), in batches |
| just prove the connection works | the **Fix Script** at the top of this page |

---

## Run the original Fix Script yourself

**The script you are running** is the **Fix Script** record named **`Elton Test Integration`** — it is
the only script this app contains, and it is the five lines shown above.

1. In ServiceNow, open the filter box at the top left (or **All**), type `Fix Scripts`, and choose
   **System Definition → Fix Scripts**. You can also paste `sys_script_fix_list.do` into the URL.
2. Find the row **`Elton Test Integration`** — the only row in this app — and click its name to open
   the record.
3. The form shows a **Script** field containing the code above. At the top of the form, click the
   **Run Fix Script** button.
4. A pop-up appears containing `Elton was here! Response: {"ip":"8.8.8.8", …}`. That text is the
   reply from ipinfo.io, printed by the last line of the script.
5. Click **OK** to close it. A Fix Script saves nothing unless its own code writes something.

If there is no **Run Fix Script** button on the form, you are looking at a different kind of record —
a Script Include, for example, cannot be run on its own.

**Prefer the output panel to a pop-up?** Use **All → System Definition → Scripts - Background**
(paste `sys_script.do`), put this in it, and click **Run Script**:

```javascript
var r = new sn_ws.RESTMessageV2('x_1569490_iplookup.IPInfo', 'Default GET');
r.setStringParameterNoEscape('ipaddress', '8.8.8.8');
var response = r.execute();
gs.info('HTTP ' + response.getStatusCode());
gs.info(response.getBody());
```

Background scripts print `gs.info` lines straight into the panel below, which is easier to copy from
than a pop-up. It is the same call, with the "throw" replaced by a log line.

You should recognise the shape of the reply — it matches the sample above. If it does not, the next
table is the quick way out.

---

## If it does not work

| What you see | What it usually means | What to do |
|---|---|---|
| the text `${ipaddress}` appears in the reply | the two names do not match | make the variable name and the text in the web address identical, capitals included |
| a complaint that the IP is invalid | the value sent was not an address | check what you put in the fill-in line |
| `missingauth` in the reply | you are testing without a free account key | nothing is broken; get a key if you will call it often |
| "not authorized" or access denied | one app is trying to use another app's REST Message | mark the REST Message as usable by your app |
| "no such REST Message" or an empty result | the name needs its app prefix | use `x_1569490_iplookup.IPInfo` |
| nothing happens, or it times out | ServiceNow cannot reach the internet | test from that network; if outbound traffic is blocked, tick **Use MID Server** on the REST Message |
| the pop-up never appears | the `throw` line was removed | put it back, or use `gs.info()` and read **System Log → All** |

---

## Prove it is the website, not ServiceNow

Worth doing once, because it separates two very different problems ("the website is down" versus
"my script is wrong"). In a terminal:

```bash
curl -s https://ipinfo.io/8.8.8.8/json
curl -s https://ipinfo.io/1.1.1.1/json
```

If those work but ServiceNow does not, the problem is on the ServiceNow side — most often network
access, not the REST Message.

---

## Build it yourself in four steps

You do not need the files in this repo to recreate it:

1. **System Web Services → Outbound → REST Message.** Name it `IPInfo`, web address
   `https://ipinfo.io/`, no authentication.
2. Add a method: name `Default GET`, type `GET`, address `https://ipinfo.io/${ipaddress}/json`,
   authentication "inherit from parent".
3. On that method add a variable called `ipaddress`, with `8.8.8.8` as its default.
4. Test it: run **Elton Test Integration** (see [Run the original Fix Script yourself](#run-the-original-fix-script-yourself)) and the pop-up shows the reply.

When you eventually need a password or key, put it on the **REST Message** (step 1) and leave the
method on "inherit from parent" — that way changing the website later does not mean re-typing
credentials.

---

## Words you will bump into

| Word | What it means here |
|---|---|
| **REST** | a plain web request followed by a reply — what browsers do all day |
| **JSON** | the `{"name": "value"}` way of writing data; to you it is just text until something turns it into fields |
| **REST Message** | ServiceNow's saved record of a web address plus how to call it |
| **RESTMessageV2** | the ServiceNow code that makes the call from a script |
| **scope / namespace** | the app's own surname, so its names do not clash with other apps' |
| **Fix Script** | a "run once and tell me what happened" script record |
| **MID Server** | a helper that sits inside your network, needed when ServiceNow itself cannot reach a site |
| **200** | web short-hand for "that worked" (404 means "not found", 403 means "not allowed") |

---

## What is in this repo

```
79fe5701835fc750e958cd19feaad39d/       the app — every file here is written by ServiceNow
  sys_app_79fe….xml                     the app record
  update/sys_rest_message_687f….xml     REST Message (IPInfo)
  update/sys_rest_message_fn_b98f….xml  the Default GET method
  update/sys_rest_message_fn_parameters_….xml ×3   the ipaddress variable
  update/sys_script_fix_1b72….xml       the script shown above
  update/sys_scope_privilege_….xml ×4   access rules
  update/sys_user_role_….xml ×2         roles
  checksum.txt                          a fingerprint ServiceNow uses to spot tampering
  README.md                             ServiceNow's own note about importing and recovering
sn_source_control.properties            tells ServiceNow where the app sits in this repo
```

## Which branch to read

ServiceNow created this repository and keeps writing to its own branches. `main` is the one made for
reading:

| Branch | What it holds | Who writes it |
|---|---|---|
| **`main`** | this guide and a copy of the app, so the repo is readable | by hand |
| `latest` | the most recent copy ServiceNow exported | ServiceNow |
| `sn_instances/dev213089` | the branch the instance itself saves to | ServiceNow |

If the app changes in ServiceNow, the two ServiceNow branches update first and `main` follows when
someone re-publishes it.

## Two things not to do

- **Do not edit the app files by hand.** ServiceNow writes them and checks them; a hand edit stops
  the app importing cleanly. Change the app in ServiceNow instead.
- **Do not expect a MID Server here.** This app talks to the internet directly, so the site has to
  be reachable from ServiceNow itself. Some of the portfolio's other samples work the other way
  round and *need* a MID Server.

This app only calls out. It cannot be called from outside — if you want that direction, see the
Remote Table sample (#6) in the portfolio.
---

## Appendix: the three ways in and out, and tying this to a real database

Everything above is **outbound**: ServiceNow asks a website a question. Real integrations also need
the opposite direction, and they need to choose between three patterns. This appendix answers:

- what a REST Message actually gives you (and what it does not),
- how to wire this to the Toronto events database, including what the PHP wrapper *is*,
- a step-by-step lab for a **Scripted REST API** (ServiceNow serves),
- a step-by-step lab for **SOAP** (ServiceNow consumes, and briefly serves),
- when to use which, what triggers each one, and where the errors go.

> Menu names drift between releases (Vancouver, Washington, Xanadu…). Where a module name is
> version-dependent I say so instead of pretending it is fixed.
>
> Every section below carries a **Breadcrumbs** block — the exact clicks involved in setting that
> thing up, so you can follow along in your own instance. The `sys_…_list.do` in brackets is the
> table behind the module: paste it after your instance address when you cannot find a menu item.

### A1. What a REST Message actually gives you

Two things, at two different times:

**At design time — a saved definition.** The address, the method (`GET`/`POST`), headers, the
authentication profile, the variables with their defaults, and optionally which MID Server carries
the call. It is a record, so it shows up in the menu, can be edited without touching code, and can be
reused by any script.

**At run time — a response object.** After `execute()` you get:

| You call | You get |
|---|---|
| `getStatusCode()` | `200` worked, `404` not found, `401/403` not allowed, `500` the far end broke, `0` the call never left |
| `getBody()` | the reply as **text** (JSON, XML, HTML — whatever they sent) |
| `getHeaders()` | the reply headers (`Content-Type`, rate-limit hints, cookies) |
| `getErrorMessage()` | why it failed, when there was no HTTP reply at all |

**What it does not give you, no matter what the marketing implies:**

| Not included | So you handle it |
|---|---|
| retries | a table of "things still to send", or a scheduled job that re-runs |
| a queue | the call is synchronous: the user (or job) waits |
| schema validation | `JSON.parse` may still throw; check `info.error` too |
| rate limiting | free APIs throttle you; batch and pause (see example 3 in the main guide) |
| guaranteed delivery | if the job dies mid-loop, nothing retries it |

That is the honest trade: low ceremony, no guarantees. If you need guarantees, use Flow Designer
(retries, error branches) or a "to-send" table you drive yourself.

**Breadcrumbs** — where each of those things actually lives:

| To do this | Click path |
|---|---|
| Create or open a REST Message | **All → System Web Services → Outbound → REST Messages** (`sys_rest_message_list.do`) |
| See the method, its endpoint and headers | open the message → **HTTP Methods** related list (one row per `GET`/`POST`) |
| See the variables and their defaults | open the method → **HTTP Request Parameters** (`sys_rest_message_fn_parameters_list.do`) |
| Keep the token/password out of the script | **Authentication** section on the message → *Basic* profile (older releases: the *Authentication profile* field; profiles are `sys_auth_profile_basic`) |
| Route the call through a MID Server | message → **Use MID Server** → pick one from **All → MID Server → Servers** |
| Run it without writing anything permanent | **All → System Definition → Scripts - Background** — paste the 5-line fix script from the main guide |

Note what is *not* in that list: there is no "response" record anywhere. The response only exists
while your script is running, which is why you log it if you want to look at it later.

### A2. Making it practical: your Toronto events database

There are two directions, and they are mirror images. Naming both sides properly is the part people
get stuck on:

| | **ServiceNow pulls** (events → ServiceNow) | **ServiceNow serves** (ServiceNow → website) |
|---|---|---|
| Who starts it | a scheduled job in ServiceNow | your PHP page |
| Website's role | **service provider** — it publishes a REST endpoint | **service consumer** — it is the client |
| ServiceNow's role | **service consumer** — outbound REST client | **service provider** — via a Scripted REST API |
| What you build | REST Message + scheduled job | Scripted REST API + one resource script |
| Data moves | website → ServiceNow | ServiceNow → website |
| Auth | token/header or basic auth **on the REST Message** | OAuth or basic auth + a role on the resource |

So, directly answering the question: **yes, the website needs a small PHP wrapper, and that wrapper is
a web service** — specifically a REST API endpoint (it answers `GET` with JSON). It is *not* "a REST
Message"; a REST Message is the ServiceNow-side record that calls it. On the ServiceNow side the thing
that does the calling is the **outbound REST client**, and the record plus `RESTMessageV2` is how you
configure and run it.

### The pull direction, end to end

```
Scheduled job (every 15 min)
   └─ RESTMessageV2('Toronto Events', 'getSince')
        ├─ GET https://findtorontoevents.ca/api/events.php?since=2026-11-04T20:00:00Z
        └─ header  X-API-Token: ****            ← secret lives on the REST Message, not in code
             │
             ▼
        {"count":2,"events":[{"event_id":"evt_123","title":"…","start":"…","venue":"…"}]}
             │
             ▼
   loop → GlideRecord upsert keyed on event_id (coalesce) → u_torontoeventsfinal
```

A minimal endpoint — this is the whole "PHP wrapper" idea, ~25 lines:

```php
<?php
// api/events.php — a REST endpoint the ServiceNow instance calls.
header('Content-Type: application/json');

$token = getenv('EVENTS_API_TOKEN') ?: '';
$sent  = $_SERVER['HTTP_X_API_TOKEN'] ?? '';
if ($token !== '' && !hash_equals($token, $sent)) {
    http_response_code(401);
    echo json_encode(['error' => 'bad or missing token']);
    exit;
}

$since = $_GET['since'] ?? '1970-01-01 00:00:00';

try {
    $pdo = new PDO('mysql:host=localhost;dbname=events;charset=utf8mb4', 'user', 'pass', [
        PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
    ]);
    $stmt = $pdo->prepare(
        'SELECT event_id, title, start_dt, venue, url
           FROM events
          WHERE updated_at >= :since
          ORDER BY updated_at ASC
          LIMIT 500'
    );
    $stmt->execute([':since' => $since]);

    $out = [];
    foreach ($stmt as $r) {
        $out[] = [
            'event_id' => $r['event_id'],
            'title'    => $r['title'],
            'start'    => $r['start_dt'],
            'venue'    => $r['venue'],
            'url'      => $r['url'],
        ];
    }
    echo json_encode(['count' => count($out), 'events' => $out]);
} catch (Throwable $e) {
    http_response_code(500);
    error_log('events.php: ' . $e->getMessage());       // details in the server log, not to the caller
    echo json_encode(['error' => 'lookup failed']);
}
```

Four things that matter in that file, and they are the same four that matter on the ServiceNow side:

1. **A secret, not a query-string password.** The token travels in a header; the ServiceNow REST
   Message holds it, and the script never does.
2. **A delta, not "everything".** `?since=` plus an `updated_at` index, so a 15-minute job moves
   hundreds of rows, not hundreds of thousands.
3. **A limit.** `LIMIT 500` and a `next` cursor keep one slow query from timing out the caller. When
   the reply has 500 rows, the job asks again with the newest `updated_at` it saw.
4. **Errors that do not leak.** The caller gets "lookup failed"; the stack trace goes to the web
   server's error log.

**Which is better here — this, or the JDBC route (#3 in the portfolio)?**

| | HTTP endpoint (this) | JDBC data source (#3) |
|---|---|---|
| Needs DB credentials on the MID Server | no | yes |
| Works with any host / language | yes (PHP, Node, Go…) | MySQL/MariaDB only |
| Can reshape or filter the payload | yes, in PHP | no, raw SQL |
| Bulk volume | slower, page by page | much faster |
| Firewall friendliness | one HTTPS URL | a database port through the MID |

Rule of thumb: HTTP when you need a *shaped, filtered* feed across the internet; JDBC when you need
*bulk* from a database you already trust.

**Breadcrumbs — the pull direction (events → ServiceNow):**

| Step | Click path |
|---|---|
| 1. Describe the endpoint | **All → System Web Services → Outbound → REST Messages → New** — name `Toronto Events`, endpoint `https://findtorontoevents.ca/api/events.php`, one method `getSince`, one parameter `since` (same tabs as A1) |
| 2. Hold the token | message → **Authentication** section (or the method's **HTTP Headers**, if you must — but then the secret sits in a field people screenshot) |
| 3. Schedule the pull | **All → System Definition → Scheduled Jobs → New → Automatically run a script of your choosing**; the script is stored in `sysauto_script`, the run-time rows you can watch are in `sys_trigger` (`sys_trigger_list.do`) |
| 4. Land the rows somewhere | **All → System Definition → Tables** (`sys_db_object_list.do`) → `u_torontoeventsfinal`; field types are in **System Definition → Dictionary** |
| 5. Or land them via an import set | **All → System Import Sets → Load Data**; the dedupe key is on the import set's **Coalesce** related list |
| 6. Check the job ran | **All → System Definition → Scheduled Jobs** (last run / next run), or `sys_trigger_list.do` filtered to your job |

The PHP file itself has no ServiceNow breadcrumb — it lives on the web host, not in the instance.
Only its token and its URL ever appear in ServiceNow.

**Breadcrumbs — the serve direction (ServiceNow → website):** that is A3, and it is a different set of
records (`sys_ws_definition` + `sys_ws_operation`), not a REST Message.

### A3. Scripted REST API — the lab (ServiceNow *serves*)

This is the reverse of the app: instead of ServiceNow calling out, you publish an address **outside
systems call**. Your PHP page, a mobile app, Power BI, Postman — anything that speaks HTTP.

**What it is:** a REST API defined in two records — the API (name, namespace, base path) and one or
more resources (HTTP method + script). Your script receives `request` and fills in `response`.

**Breadcrumbs** — the whole build, in six clicks:

| To do this | Click path |
|---|---|
| Create the API | **All → System Web Services → Scripted REST APIs → New** (`sys_ws_definition_list.do`); in some releases it sits under *System Web Services → Scripted Web Services → Scripted REST APIs* |
| Add a resource | the API record → **Resources** related list → **New** (`sys_ws_operation_list.do`) |
| Make the role it requires | **All → User Administration → Roles → New** → `x_1569490_iplookup.api_user` (`sys_user_role_list.do`) |
| Give a caller that role | **All → User Administration → Users** → the integration user → **Roles** tab |
| Test without a client | **All → System Web Services → REST API Explorer** |
| Read the errors | **All → System Log → All** (`syslog_list.do`), filtered to this scope |

One thing that trips people up before they even start: both records are **scoped**. Click the
scope/app picker next to the banner (or **All → System Applications → My Company Applications**) and
switch to `x_1569490_iplookup` first, or you will build the API in Global and then wonder why the
app's other files cannot see it.

### Step by step

**Step 1 — create the API.** Row 1 of the table above; the fields are what matter here.

| Field | Value to use | Mandatory? | Why it matters |
|---|---|---|---|
| **Name** | `IP Info API` | yes | the human label |
| **API ID** | `ipinfo` | yes | becomes the URL segment |
| **API namespace** | `x_1569490_iplookup` (the app scope) | yes | also in the URL, and decides which scripts can see it |
| **Short description** | what it serves | no | the only documentation your consumers will read |
| **Requires authentication** | leave **true** | — | turning this off makes a public, unauthenticated endpoint |

The resulting base path is `/api/<namespace>/<API ID>`, i.e.
`https://<your-instance>.service-now.com/api/x_1569490_iplookup/ipinfo`.

**Step 2 — add a resource.** Row 2 of the table above.

| Field | Value | Mandatory? | Notes |
|---|---|---|---|
| **Name** | `lookup` | yes | label only |
| **HTTP method** | `GET` | yes | `GET` reads, `POST` creates, `PUT/PATCH` update, `DELETE` removes |
| **Relative path** | `/lookup` | yes | appended to the base path |
| **Requires authentication** | true | — | inherit unless you have a reason |
| **Requires role** | e.g. `x_1569490_iplookup.api_user` | practically yes | **this is the usual cause of a 403** |
| **Script** | below | yes | runs as the *calling* user unless you say otherwise |

**Step 3 — the script.** It is a function with two arguments, and you must call it:

```javascript
(function process(request, response) {

    var ip = request.queryParams.ip;

    if (!ip) {
        response.setStatus(400);
        response.setHeader('Content-Type', 'application/json');
        response.setBody(JSON.stringify({ error: 'pass ?ip=8.8.8.8' }));
        return;
    }

    var info = new IpInfoLookup().lookup(ip);      // the Script Include from the main guide

    if (!info.ok) {
        response.setStatus(502);                    // 502 = the far end misbehaved, not you
        response.setHeader('Content-Type', 'application/json');
        response.setBody(JSON.stringify({ error: info.error, upstreamStatus: info.status }));
        return;
    }

    response.setStatus(200);
    response.setHeader('Content-Type', 'application/json');
    response.setBody(JSON.stringify({
        ip: info.ip, city: info.city, country: info.country, org: info.org
    }));

})(request, response);
```

Three habits worth copying: **`JSON.stringify` explicitly** (passing an object to `setBody` is not a
reliable way to get JSON), **a different status for "you asked wrong" (400) versus "the far end
failed" (502)**, and **never the stack trace in the body**.

**Step 4 — test before writing a client.**
Breadcrumb: **All → System Web Services → REST API Explorer** → pick `IP Info API` → `lookup` → enter
`8.8.8.8` → **Send**. The explorer shows the request, the response and a `curl` you can copy. That
curl is the breadcrumb to hand a PHP developer.

**Step 5 — call it from outside.**

```bash
curl -u 'integration.user:password' \
     'https://your-instance.service-now.com/api/x_1569490_iplookup/ipinfo/lookup?ip=8.8.8.8'
```

Basic auth is fine for a first test; for anything real use OAuth (or a dedicated integration user
with a strong password and exactly one role). Never reuse an admin account.

### What will bite you

| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` | no credentials, or basic auth disabled | check the API's *Requires authentication*; confirm the user can log in |
| `403 Forbidden` | authenticated, but missing the role — or missing the platform's `snc_internal` / `snc_external` role | add the resource role to the integration user |
| `400` with a JSON body | your own validation branch | read your own message; the caller did something wrong |
| `500`, empty body | your resource script threw | read **System Log → All** filtered to your scope: the caller never sees the stack trace |
| `404` | wrong path — usually the namespace or API ID | compare with the URL shown in the REST API Explorer |
| works in the explorer, fails outside | the explorer runs as *you* (admin); the outside caller does not | test with the integration user's credentials |
| slow, or `timeout` | work inside the request | keep the handler under a second: return an id, do the heavy part in a scheduled job |

**Debugging, concretely:** add `gs.info('[ipinfo api] ' + ip)` at the top of the resource and
`gs.error(...)` in each failure branch; read **System Log → All** with the scope filter; use
**Session Debug → Debug** only if you need line-level tracing (it is slow and noisy). Where errors go
is split by design: **the caller gets a status code and a short message; the server gets the stack
trace.**

**Performance, concretely:** a request holds a transaction slot for its whole duration, so the
expensive things to avoid are loops over `GlideRecord` with per-row queries, unbounded queries (no
`setLimit`), and calling another slow API inline. If the answer is not instant, store the request in a
table, return `202 Accepted` with an id, and let a job do the work.

### A4. SOAP — the lab

SOAP is the older XML protocol: a rigid envelope, a strict contract (**WSDL**) describing every
operation, and its own error shape (`soap:Fault`). It is still everywhere in enterprise land (SAP,
older .NET services, government systems). Prefer REST when you have the choice; use SOAP when the
other side only offers it.

### Consume SOAP — the path that works on every release

You do not need any ServiceNow records: POST the envelope as if it were a REST call with an XML body.
Using W3Schools' public temperature converter (a real service, handy for practice):

```javascript
var env = '<?xml version="1.0" encoding="utf-8"?>' +
  '<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"' +
  ' xmlns:xsd="http://www.w3.org/2001/XMLSchema"' +
  ' xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">' +
  '<soap:Body><FahrenheitToCelsius xmlns="https://www.w3schools.com/xml/">' +
  '<Fahrenheit>100</Fahrenheit></FahrenheitToCelsius></soap:Body></soap:Envelope>';

var msg = new sn_ws.RESTMessageV2();          // no saved record: an ad-hoc call
msg.setHttpMethod('post');
msg.setEndpoint('https://www.w3schools.com/xml/tempconvert.asmx');
msg.setRequestHeader('Content-Type', 'text/xml; charset=utf-8');
msg.setRequestHeader('SOAPAction', 'https://www.w3schools.com/xml/FahrenheitToCelsius');
msg.setRequestBody(env);

var res = msg.execute();
gs.info('SOAP ' + res.getStatusCode() + ' ' + res.getBody());
```

Reading the value out (the reply is XML, so XML-parsing, not `JSON.parse`):

```javascript
var doc = new XMLDocument2();
doc.parseXML(res.getBody());
var celsius = doc.getNodeText('//*[local-name()="FahrenheitToCelsiusResult"]');
gs.info('100F = ' + celsius + 'C');
```

If `XMLDocument2` is not available in your release, import the WSDL instead (below) and let ServiceNow
do the parsing for you — that is the trade.

**The WSDL route** (row 2 of the table above; in older instances the module is simply *Import WSDL*
under System Web Services). Add the WSDL URL, and ServiceNow generates a message record plus one
function per operation, with typed arguments. Your script then calls it by name instead of
hand-writing envelopes.
Confirm the class name for your release before relying on it (some expose a SOAP-specific message
class, others generate code that calls the message record).

### Publish SOAP from ServiceNow — and why you probably should not

ServiceNow has a legacy *Scripted SOAP Web Service* module that publishes a WSDL you write by hand.
It works, but compared with a Scripted REST API you get: manual WSDL, no explorer, no easy versioning,
awkward JSON conversion (none — it is XML in both directions), and very little community material.
Publish REST unless a specific consumer only speaks SOAP; if so, put a tiny REST→SOAP bridge next to
it (a PHP or Node service) rather than maintaining a WSDL inside ServiceNow.

**Breadcrumbs — the three SOAP setups:**

| To do this | Click path |
|---|---|
| Send a hand-built envelope (no records at all) | **All → System Definition → Scripts - Background** — paste the snippet above and read the log |
| Import a WSDL and let ServiceNow write the code | **All → System Web Services → Outbound → Import WSDL** (some releases: **Outbound → SOAP Messages → New → Import WSDL**) → paste the `.asmx?WSDL` URL → **Generate** |
| See what the import produced | **All → System Web Services → Outbound → SOAP Messages** — one message record, plus one function per WSDL operation, with typed arguments |
| Look at the WSDL again | open the message record → the **WSDL** link/related link it carries |
| Publish a SOAP service from ServiceNow (not recommended) | **All → System Web Services → Scripted SOAP Web Services → New** — older releases keep this under *System Web Services → Scripted Web Services* |
| Test from outside | not in ServiceNow: Postman, SoapUI, or `curl` with the envelope in `--data` |

If the *Import WSDL* module is missing entirely, that instance has SOAP outbound disabled or the
plugin is not active — check with your admin rather than hunting the menu for ten minutes.

### SOAP's three classic failures

| Symptom | Cause | Fix |
|---|---|---|
| `500` with `<soap:Fault>` in the body | the far end rejected the request | read the **fault string inside the body**, not just the status code |
| "Server did not recognize the value of HTTP Header SOAPAction" | `SOAPAction` missing or misspelled | copy it character-for-character from the WSDL |
| Everything else fails, envelope looks right | malformed XML (one unclosed tag invalidates the whole message) | paste the envelope into any XML validator before blaming ServiceNow |

### A5. Side by side

| | **`RESTMessageV2`** | **Scripted REST API** | **SOAP message** |
|---|---|---|---|
| Direction | outbound | inbound | outbound, and (legacy) inbound |
| Who is the client | ServiceNow | the outside system | ServiceNow |
| What you create | REST Message + method + variables | API record + resource script | WSDL import (or hand-built envelope) |
| Data format | JSON (usually) | JSON | XML only |
| Auth | on the message / profile | role on the resource, OAuth or basic | WS-Security, basic, or none |
| Test tool | Fix Script / log | REST API Explorer | Postman / SoapUI / curl |
| Best at | asking another system a question | letting another system ask *you* | talking to a system that only speaks SOAP |
| Ceremony | low | medium | high |
| Modern choice? | yes | yes | only when forced |
| Where you build it | **System Web Services → Outbound → REST Messages** | **System Web Services → Scripted REST APIs** | **System Web Services → Outbound → Import WSDL / SOAP Messages** |
| Where you test it | **System Definition → Scripts - Background** | **System Web Services → REST API Explorer** | Postman / SoapUI, or Scripts - Background for a hand-built envelope |
| The record behind it | `sys_rest_message` + `sys_rest_message_fn` | `sys_ws_definition` + `sys_ws_operation` | the SOAP message record created by the WSDL import |

**Breadcrumbs:** the last three rows above are the ones you click — where to build it, where to test
it, and the record to check when the menu hides the module. All three live under the same **System Web
Services** menu, which is why "which one am I looking at?" is a fair question — the table names in
that last row are the reliable answer.

### A6. What triggers what, and why

| Trigger | Right when | What it costs you |
|---|---|---|
| **Business Rule** (before/after insert/update) | the call must happen as part of the save, the payload is small, and the user can wait under a second | it blocks the transaction: a slow website makes the form feel broken, and an exception can roll the save back |
| **Scheduled job** | delta pulls, backfills, retries, anything batched | not real-time; you need a watermark and a page size |
| **Flow Designer action** | a business analyst should own the automation, or you need retries and error branches | licence/entitlement, harder to unit test |
| **UI Action** | a human clicks "look this up now" | runs as that user; synchronous; needs its own permission thinking |
| **Async Business Rule / event queue** | save must be fast, the call can happen a moment later | more moving parts (event registry, script action) — but this is the safe way to call out on save |
| **External HTTP call** (someone calls your Scripted REST API) | the website needs ServiceNow's data | you own auth, rate limits and a stable contract |

The pairing that covers most real work: **Scripted REST API for inbound, `RESTMessageV2` inside a
scheduled job for outbound**, with Flow Designer where a non-developer must own it.

**Breadcrumbs — where you create each trigger:**

| Trigger | Click path | Table behind it |
|---|---|---|
| Business Rule | **All → System Definition → Business Rules → New** (or on the table: **Configure → Business Rules**) | `sys_script` |
| Scheduled job | **All → System Definition → Scheduled Jobs → New** | `sysauto_script` (runs in `sys_trigger`) |
| Flow Designer action | **All → Process Automation → Flow Designer → New → Action** | `sys_hub_action_type_definition` |
| UI Action | **All → System Definition → UI Actions → New** (or on the form: **Configure → UI Actions**) | `sys_ui_action` |
| Async Business Rule / event queue | **System Definition → Business Rules** (tick *asynchronous*), plus **All → System Policy → Events → Registry** and **→ Script Actions** (older releases: *System Definition → Events → Registry*) | `sysevent_register`, `sysevent_script_action` |
| Inbound endpoint | see A3 | `sys_ws_definition` |

### A7. One debug checklist for all three

| Question | Where to look |
|---|---|
| Did the call even leave? | outbound: `getStatusCode() === 0` means no reply at all; check network/MID |
| What exactly was sent? | log the substituted address and the body, not just "it failed" |
| What did they say? | the body — always log the body on failure, not just the code |
| Why did my resource 500? | **System Log → All**, filtered to your scope; the caller never sees it |
| Is it slow, or dead? | time one call in the explorer / with curl before blaming ServiceNow |
| Is it a permission or a logic problem? | strip the logic to a "ping" resource or a bare call; if ping works, it is logic |
| Did it work yesterday and not today? | check for a changed token, an expired certificate, a new role requirement, or a rate limit |

Rules that save the most time: **log the request and the response body on failure**, **use a distinct
status code per cause**, **never put secrets in a URL**, and **never call a slow system from a
Business Rule** without an async step between them.

**Breadcrumbs — the tools behind each question:**

| Tool | Click path |
|---|---|
| See your own log lines | **All → System Log → All** (`syslog_list.do`) — filter **Source** to your script, or search for the tag you put in the message (`[ipinfo api]`) |
| See how long a request took | **All → System Log → Transactions** (`syslog_transaction_list.do`) — this is where "it was slow" becomes a number |
| Get line-by-line tracing | **All → System Diagnostics → Session Debug → Debug** (older: *System Diagnostics → Session Debug → Enable Debug*); turn it off afterwards, it is noisy and slow |
| Re-run one call in isolation | **All → System Definition → Scripts - Background** — the smallest possible repro |
| Test an inbound resource | **All → System Web Services → REST API Explorer** — same call, as you, with the request visible |
| Check a MID Server | **All → MID Server → Servers** (status must be *Up*), then **→ Capabilities**; also `ecc_agent_list.do` |
| Check an expired certificate | `sys_certificate_list.do` — look for a *Valid to* date in the past |

If the same call works in *Scripts - Background* and fails from the website, the problem is auth or
scope, not your code — the breadcrumb to compare is the *System Log → All* rows from each attempt.
