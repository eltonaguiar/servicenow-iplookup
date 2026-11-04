# ServiceNow IP Lookup — explained simply

> **In one sentence:** this app asks a website called ipinfo.io *"where is the IP address 8.8.8.8?"*
> and shows you the answer.

It is a small ServiceNow app (`IPLookup`) that demonstrates the most common integration task there
is: **ServiceNow calling somebody else's website and reading the reply.**

There is no login, no server to install, and no password. If you can read a web address and run a
script, you can follow this.

Part of [eltonaguiar/ServiceNow_Portfolio](https://github.com/eltonaguiar/ServiceNow_Portfolio)
(entry **#7**).

**Jump to:** [the 60-second version](#start-here-the-60-second-version) ·
[fundamentals: why this way](#fundamentals-why-call-a-website-at-all-and-why-this-way) ·
[why the address has a blank](#why-the-address-has-a-blank-in-it) ·
[what the answer looks like](#what-the-answer-looks-like) ·
[the code line by line](#the-script-explained-line-by-line) ·
[three complete examples](#real-code-three-complete-examples) ·
[troubleshooting](#if-it-does-not-work) · [glossary](#words-you-will-bump-into)

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

1. **The name has to match on both sides.** The blank is called `ipaddress` in the address, so the
   app must also have a variable called `ipaddress`. If those two names differ by even one letter,
   ServiceNow sends the text `${ipaddress}` to ipinfo.io instead of an address — and ipinfo.io
   politely says that is not an IP address.
2. **There is a "fill in exactly as written" version** of the fill-in command
   (`setStringParameterNoEscape`). The other version tidies your value for use inside a web address
   (turning spaces into `%20`). For something like `8.8.8.8` either works, and the app uses the
   "exactly as written" one.

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

```javascript
var r = new sn_ws.RESTMessageV2('x_1569490_iplookup.IPInfo', 'Default GET');
r.setStringParameterNoEscape('ipaddress', '8.8.8.8');
var response = r.execute();
var responseBody = response.getBody();

// Show the answer on screen. A Fix Script only displays something when
// the script reports a problem, so we deliberately report the answer.
throw "Elton was here! Response: " + responseBody;
```

| Line | What it is doing | What goes wrong |
|---|---|---|
| first line | opens the saved REST Message `IPInfo` and picks the method `Default GET` | if the app lives in another app's namespace, the name needs that prefix; a misspelled method fails straight away |
| second line | fills the blank with `8.8.8.8` | wrong name → the text `${ipaddress}` gets sent; wrong value → ipinfo.io says it is not an address |
| third line | actually visits the website (this is where the waiting happens) | with no internet access from ServiceNow this fails or times out |
| fourth line | grabs the reply as plain text | it is text, not a ready-made list of fields — see the next section |
| last line | puts the reply in the pop-up | see below |

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

### Which one do I need?

| You want to… | Use |
|---|---|
| look an address up from several places | the **Script Include** (example 1) — write it once |
| fill fields while a person is editing a record | the **Business Rule** (example 2), with the "only if changed" guard |
| catch up an existing table of addresses | the **Scheduled Job** (example 3), in batches |
| just prove the connection works | the **Fix Script** at the top of this page |

---

## Run the original Fix Script yourself

1. Open **All → System Definition → Fix Scripts → Elton Test Integration**.
2. Click **Run Fix Script**.
3. A pop-up shows `Elton was here! Response: {"ip":"8.8.8.8", …}`.

You should recognise the shape of that reply — it is the same as the sample above. If you get
something else, the next table is the quick way out.

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
4. Run the script and watch the pop-up.

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
