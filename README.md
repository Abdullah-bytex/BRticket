# -*- coding: utf-8 -*-
import json
import os
import re
import shlex
import threading
import time
import uuid
import urllib.error
import urllib.parse
import urllib.request
import webbrowser
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

DEFAULT_URL = "https://railspaapi.shohoz.com/v1.0/web/bookings/search-trips-v2"
MON = ["Jan", "Feb", "Mar", "Apr", "May", "Jun", "Jul", "Aug", "Sep", "Oct", "Nov", "Dec"]
SKIP_HEADERS = {"host", "content-length", "accept-encoding", "connection"}
TO_KEYS = {"to", "to_city", "tocity", "to_station", "dest", "destination"}
TTL = 20
cache = {}
PROFILE = None

LOGIN_URLS = [os.environ["BR_LOGIN_URL"]] if os.environ.get("BR_LOGIN_URL") else [
    "https://railspaapi.shohoz.com/v1.0/web/auth/sign-in",
    "https://railspaapi.shohoz.com/v1.0/app/auth/sign-in",
]
SESSION = {"token": "", "mobile": "", "device_id": "", "device_key": "", "at": 0}
CREDS = {"mobile": "", "password": ""}

DEFAULT_UUDID = 'a6f37f0241c43754df530241996633e0'
DEFAULT_SSDK = '84735db53f5e8ac14220564eb882d35050f7b0a8de0f0b2ed39c5c802fce3dda6edb4198a9acc5f5e1895cf0754eeea8c54924fbb09df388ea41965e99a81c8bd5540122da1c16bc0f6e8b1dcf905efb'
DEFAULT_TOKEN = 'eyJhbGciOiJSUzI1NiIsInR5cCI6ImF0K2p3dCJ9.eyJuYmYiOjE3OTI1NjI1MzEsImV4cCI6MTc5MjYwNSczMSwiaXNzIjoiaHR0cDovL3RyYWluLWlhbS5zaG9ob3ouY29tIiwiYXVkIjoic2hvaG96LmlhbS50cmFpbiIsImNsaWVudF9pZCI6InRyYWluLXRpY2tldC11c2VyIiwiY2xpZW50X3RlbmFudF9pZCI6IjY3YTUxNjNjLTU2NWQtNGNhMC1iMTE3LTg3YzcwMWY2NTRmOSIsImNsaWVudF92ZXJ0aWNhbF9pZCI6IjA2OGU4MDRmLTJkYjctNDQ4OS05OTYzLTc4OWZiN2UwMmJkZiIsInN1YiI6ImQ0MTcyZDM1LTBmYjgtNGM4Mi1hZWRjLWE2ZDNkOGM4NWM3OCIsImF1dGhfdGltZSI6MTc5MjU2MjUzMSwiaWRwIjoibG9jYWwiLCJwaG9uZV9udW1iZXIiOiIwMTU3MjkwMjYxMiIsImVtYWlsIjoiYWJkdWxsYWg4NTgwMTVAZ21haWwuY29tIiwidXNlcm5hbWUiOiIwMTU3MjkwMjYxMiIsImRpc3BsYXlfbmFtZSI6IkFiZHVsbGFoIiwibG9jYWxlIjoiYm4tQkQiLCJuaWRuIjoiMjg4MDkzMDUxMiIsIm5pZG50IjoiTklEIiwicm9sZSI6WyJ1c2VyIiwiYW5vbnltb3VzIl0sInNjb3BlIjpbInNob2hvei5pYW0udHJhaW4iLCJvZmZsaW5lX2FjY2VzcyJdLCJhbXIiOlsicHdkIl19.Bygd4oNNWWdF-sYfpNPJjePAsuncZss25XWC6zkl_RIM9IuzjD7g_Uyy-CAVhvL9Pzk7lF9p00Ys9TxNiReUpNTNqlV16aKESHo01zSSKbKgz1J6nzRCrXuHmSPBm3SgpqeY-zg_GbKnUBenF0YvD1lAhkDc29lKzes9MJ0SnwbAPry_ux_WWsO2HYuDg-qIYShtcrc9Kgd_weCXmRMHOPxw8i9CmzFGo2fL74ZmtgJ0X3x6cx6BzIlKzuffoVy54F5FxKzRHpKrpmvwuwnF4kKZWTrPCQXsRlf6-9qPXEhqXJFMnfHLsbsHJznYCtQ-fPQJ4GBntcQjIndWXn1fxA'

CLEAR_LOGIN_ON_START = True


def load_device():
    return {"device_id": DEFAULT_UUDID or str(uuid.uuid4()), "device_key": DEFAULT_SSDK, "token": DEFAULT_TOKEN}


def save_device():
    pass


def find_key(o, names):
    if isinstance(o, dict):
        for k, v in o.items():
            if str(k).lower().replace("-", "_") in names and isinstance(v, str) and v:
                return v
        for v in o.values():
            r = find_key(v, names)
            if r:
                return r
    elif isinstance(o, list):
        for v in o:
            r = find_key(v, names)
            if r:
                return r
    return ""


def token_expiry(token):
    try:
        import base64
        part = token.split(".")[1]
        part += "=" * (-len(part) % 4)
        exp = json.loads(base64.urlsafe_b64decode(part)).get("exp")
        return int(exp - time.time()) if exp else None
    except Exception:
        return None


def auth_headers():
    h = {}
    if SESSION["token"]:
        h["Authorization"] = "Bearer " + SESSION["token"]
    if SESSION["device_id"]:
        h["x-device-id"] = SESSION["device_id"]
    if SESSION["device_key"]:
        h["x-device-key"] = SESSION["device_key"]
    return h


def find_token(o):
    if isinstance(o, dict):
        for k, v in o.items():
            if str(k).lower() in ("token", "access_token", "accesstoken", "jwt") and isinstance(v, str) and len(v) > 20:
                return v
        for v in o.values():
            t = find_token(v)
            if t:
                return t
    elif isinstance(o, list):
        for v in o:
            t = find_token(v)
            if t:
                return t
    return ""


def password_login(mobile, password):
    body = json.dumps({"mobile_number": mobile, "password": password}).encode("utf-8")
    hdr = {"Content-Type": "application/json", "Accept": "application/json", "User-Agent": "Mozilla/5.0",
           "Origin": "https://eticket.railway.gov.bd", "Referer": "https://eticket.railway.gov.bd/"}
    if SESSION["device_id"]:
        hdr["x-device-id"] = SESSION["device_id"]
    if SESSION["device_key"]:
        hdr["x-device-key"] = SESSION["device_key"]
    last = ("login failed", "")
    for url in LOGIN_URLS:
        try:
            req = urllib.request.Request(url, data=body, method="POST", headers=hdr)
            with urllib.request.urlopen(req, timeout=20) as r:
                raw = r.read()
                reply_headers = r.headers
            try:
                parsed = json.loads(raw)
            except ValueError:
                parsed = {}
            token = find_token(parsed)
            if not token:
                last = ("the railway server replied but no login token was found in it", raw[:800].decode("utf-8", "replace"))
                continue
            key = find_key(parsed, ("ssdk", "device_key", "devicekey")) or reply_headers.get("x-device-key") or ""
            if key:
                SESSION["device_key"] = key
            SESSION.update(token=token, mobile=mobile)
            CREDS.update(mobile=mobile, password=password)
            save_device()
            print("Logged in as", mobile, "| device id: yes | device key:", "yes" if SESSION["device_key"] else "not given by server")
            return True, "", ""
        except urllib.error.HTTPError as e:
            print("Login refused", e.code, url)
            last = ("the railway server refused the login (%s)" % e.code, e.read()[:800].decode("utf-8", "replace"))
        except Exception as e:
            print("Login failed:", e)
            last = ("cannot reach railway server (%s)" % e, "")
    return False, last[0], last[1]


BROWSER_DIR = os.path.join(os.path.dirname(os.path.abspath(__file__)), "br_browser_profile")
SITE = "https://eticket.railway.gov.bd/"
USE_BROWSER = os.environ.get("BR_BROWSER") == "1"
browser_lock = threading.Lock()
browser_failed_at = [0.0]


def token_phone(token):
    try:
        import base64
        part = token.split(".")[1]
        part += "=" * (-len(part) % 4)
        return str(json.loads(base64.urlsafe_b64decode(part)).get("phone_number") or "")
    except Exception:
        return ""


def read_storage(page):
    try:
        return page.evaluate("() => { const o = {}; for (let i = 0; i < localStorage.length; i++) { const k = localStorage.key(i); o[k] = localStorage.getItem(k); } return o; }")
    except Exception:
        return {}


def pick_from_storage(items):
    want = {"token": ("token", "access_token", "accesstoken"),
            "device_id": ("uudid", "device_id", "deviceid"),
            "device_key": ("ssdk", "device_key", "devicekey")}
    out = {"token": "", "device_id": "", "device_key": ""}
    for k, v in (items or {}).items():
        kl = str(k).lower().replace("-", "_")
        try:
            parsed = json.loads(v)
        except (ValueError, TypeError):
            parsed = v
        for field, names in want.items():
            if out[field]:
                continue
            if kl in names and isinstance(parsed, str) and parsed:
                out[field] = parsed
            elif isinstance(parsed, (dict, list)):
                out[field] = find_key(parsed, names) or out[field]
    if not out["token"]:
        for v in (items or {}).values():
            m = re.search(r"eyJ[\w-]+\.[\w-]+\.[\w-]+", str(v))
            if m:
                out["token"] = m.group(0)
                break
    return out


def fill_login(page, mobile, password):
    try:
        pw = page.locator('input[type="password"]').first
        pw.wait_for(state="visible", timeout=10000)
        mob = page.locator('input[type="tel"], input[name*="mobile" i], input[name*="phone" i], '
                           'input[placeholder*="mobile" i], input[placeholder*="phone" i], input[type="text"]').first
        mob.fill(mobile)
        pw.fill(password)
        print("Form filled. If the window shows a Cloudflare check, click it: the program waits for you.")
        page.get_by_role("button", name=re.compile(r"log\s*in|sign\s*in", re.I)).first.click(timeout=120000)
    except Exception as e:
        print("Could not fill the login form by itself, type it in the browser window:", type(e).__name__)


def browser_login(mobile, password, wait=180):
    try:
        from playwright.sync_api import sync_playwright
    except ImportError:
        return False, "automatic browser login needs Playwright. In a terminal run:  pip install playwright   then   playwright install chromium", ""
    digits = re.sub(r"\D", "", mobile)[-10:]

    def good(g):
        t = g["token"]
        if not t or (token_expiry(t) or 0) < 60:
            return False
        ph = token_phone(t)
        return ph == "" or ph.endswith(digits)

    with browser_lock:
        try:
            with sync_playwright() as p:
                opts = dict(headless=os.environ.get("BR_HEADLESS") == "1", viewport={"width": 1100, "height": 800})
                try:
                    ctx = p.chromium.launch_persistent_context(BROWSER_DIR, channel="chrome", **opts)
                except Exception:
                    print("Google Chrome not found, using the test browser instead.")
                    ctx = p.chromium.launch_persistent_context(BROWSER_DIR, **opts)
                try:
                    page = ctx.pages[0] if ctx.pages else ctx.new_page()
                    page.goto(SITE, wait_until="domcontentloaded", timeout=45000)
                    page.wait_for_timeout(2500)
                    got = pick_from_storage(read_storage(page))
                    if not good(got):
                        page.goto(SITE + "login", wait_until="domcontentloaded", timeout=45000)
                        fill_login(page, mobile, password)
                        end = time.time() + wait
                        while time.time() < end and not good(got):
                            page.wait_for_timeout(1500)
                            got = pick_from_storage(read_storage(page))
                    if not good(got):
                        return False, "no login token appeared in the browser in %d seconds" % wait, ""
                    SESSION["token"], SESSION["mobile"] = got["token"], mobile
                    if got["device_id"]:
                        SESSION["device_id"] = got["device_id"]
                    if got["device_key"]:
                        SESSION["device_key"] = got["device_key"]
                    CREDS.update(mobile=mobile, password=password)
                    return True, "", ""
                finally:
                    ctx.close()
        except Exception as e:
            msg = str(e)
            return False, "browser login failed (%s)" % type(e).__name__, msg[:400]


def full_login(mobile, password, auto=False):
    errs = []
    ok, err, detail = password_login(mobile, password)
    if ok:
        return True, "", ""
    errs.append(err)
    if USE_BROWSER and not (auto and time.time() - browser_failed_at[0] < 180):
        ok2, err2, detail2 = browser_login(mobile, password)
        if ok2:
            return True, "", ""
        browser_failed_at[0] = time.time()
        errs.append(err2)
        detail = detail or detail2
    return False, "; ".join(e for e in errs if e), detail


def clean_cmd(text):
    if re.search(r"\^\r?\n", text) or '^"' in text:
        text = re.sub(r"\^\r?\n", " ", text)
        text = re.sub(r"\^(.)", r"\1", text, flags=re.S)
    return text


def parse_curl(text):
    text = clean_cmd(text.strip())
    text = re.sub(r"\\\r?\n", " ", text)
    text = re.sub(r"(?<=\s)\$'", "'", text)
    try:
        parts = shlex.split(text)
    except ValueError:
        raise ValueError("could not read it. In DevTools use right-click > Copy > Copy as cURL")
    url, headers, method, body, i = "", {}, "", None, 0
    while i < len(parts):
        a = parts[i]
        nxt = parts[i + 1] if i + 1 < len(parts) else ""
        if a in ("-H", "--header"):
            k, _, v = nxt.partition(":")
            if k.strip() and k.strip().lower() not in SKIP_HEADERS:
                headers[k.strip()] = v.strip()
            i += 2
        elif a in ("-b", "--cookie"):
            headers["Cookie"] = nxt
            i += 2
        elif a in ("-X", "--request"):
            method = nxt.upper()
            i += 2
        elif a in ("-d", "--data", "--data-raw", "--data-binary", "--data-urlencode"):
            body = nxt
            i += 2
        else:
            if a.startswith("http") and not url:
                url = a
            i += 1
    if not url:
        raise ValueError("no web address found in the pasted text")
    if body is not None and not method:
        method = "POST"
    return {"url": url, "headers": headers, "method": method or "GET", "body": body}


def profile_to_curl(prof):
    if not prof:
        return ""
    parts = ["curl " + shlex.quote(prof["url"])]
    if prof.get("method") and prof["method"] != "GET":
        parts.append("-X " + prof["method"])
    for k, v in prof["headers"].items():
        parts.append("-H " + shlex.quote("%s: %s" % (k, v)))
    if prof.get("body") is not None:
        parts.append("--data-raw " + shlex.quote(prof["body"]))
    return " \\\n  ".join(parts)


def fmt_date(iso, sample):
    y, m, d = iso.split("-")
    if re.fullmatch(r"\d{4}-\d{2}-\d{2}", sample or ""):
        return iso
    return "%s-%s-%s" % (d, MON[int(m) - 1], y)


def swap(key, val, frm, to, date, cls):
    k = str(key).lower()
    if "from" in k:
        return frm
    if k in TO_KEYS or k.startswith(("to_", "tocity")):
        return to
    if "date" in k or k == "doj":
        return fmt_date(date, str(val))
    if "class" in k and cls:
        return cls
    return val


def walk(o, *a):
    if isinstance(o, dict):
        return {k: (walk(v, *a) if isinstance(v, (dict, list)) else swap(k, v, *a)) for k, v in o.items()}
    if isinstance(o, list):
        return [walk(v, *a) for v in o]
    return o


def build_request(frm, to, date, cls):
    a = (frm, to, date, cls)
    if not PROFILE:
        q = {"from_city": frm, "to_city": to, "date_of_journey": fmt_date(date, "")}
        if cls:
            q["seat_class"] = cls
        hdr = {"Accept": "application/json", "User-Agent": "Mozilla/5.0",
               "Origin": "https://eticket.railway.gov.bd", "Referer": "https://eticket.railway.gov.bd/"}
        hdr.update(auth_headers())
        return DEFAULT_URL + "?" + urllib.parse.urlencode(q), hdr, "GET", None
    u = urllib.parse.urlsplit(PROFILE["url"])
    pairs = [(k, swap(k, v, *a)) for k, v in urllib.parse.parse_qsl(u.query, keep_blank_values=True)]
    url = urllib.parse.urlunsplit((u.scheme, u.netloc, u.path, urllib.parse.urlencode(pairs), ""))
    data, body = None, PROFILE["body"]
    if body is not None:
        try:
            data = json.dumps(walk(json.loads(body), *a)).encode("utf-8")
        except ValueError:
            data = urllib.parse.urlencode(
                [(k, swap(k, v, *a)) for k, v in urllib.parse.parse_qsl(body, keep_blank_values=True)]).encode("utf-8")
    return url, dict(PROFILE["headers"]), PROFILE["method"], data


CAPTURE_HOSTS = ("shohoz.com", "railway.gov.bd")
ALLOWED_ORIGIN = "https://eticket.railway.gov.bd"
PROFILE_AT = [0.0]


def set_captured(rec):
    global PROFILE
    if not isinstance(rec, dict):
        raise ValueError("bad capture data")
    url = str(rec.get("url") or "")
    host = (urllib.parse.urlsplit(url).hostname or "").lower()
    if not any(host == h or host.endswith("." + h) for h in CAPTURE_HOSTS):
        raise ValueError("refused: %r is not a railway address" % host)
    headers = {}
    for k, v in (rec.get("headers") or {}).items():
        k = str(k).strip()
        if k and not k.startswith(":") and k.lower() not in SKIP_HEADERS:
            headers[k] = str(v)
    if not any(k.lower() == "authorization" for k in headers):
        raise ValueError("the captured request has no login (authorization) header")
    body = rec.get("body")
    PROFILE = {"url": url, "headers": headers, "method": str(rec.get("method") or "GET").upper(),
               "body": body if isinstance(body, str) else None}
    PROFILE_AT[0] = time.time()
    cache.clear()


HOOK_JS = r"""function (o) {
  var W = (o.mode === "gm" && typeof unsafeWindow !== "undefined") ? unsafeWindow : window;
  if (W.__brHooked) return;
  W.__brHooked = true;
  var RE = /search-trips-v2/i, sent = false;
  var F = W.fetch, XP = W.XMLHttpRequest && W.XMLHttpRequest.prototype;
  function toast(msg, bad) {
    function put() {
      var d = document.createElement("div");
      d.textContent = msg;
      d.style.cssText = "position:fixed;z-index:2147483647;top:12px;right:12px;padding:10px 14px;border-radius:8px;font:600 14px system-ui,sans-serif;color:#fff;background:" + (bad ? "#C8102E" : "#00573F");
      (document.body || document.documentElement).appendChild(d);
      setTimeout(function () { d.remove(); }, 7000);
    }
    if (document.body) put(); else document.addEventListener("DOMContentLoaded", put);
  }
  function done() {
    toast("Sent to your site. " + (o.close ? "This tab will close." : "You can go back to it."));
    if (o.close) setTimeout(function () { try { window.close(); } catch (e) {} }, 1800);
  }
  function fail(why) { sent = false; toast(why || ("Could not reach your site at " + o.server + ". Is server.py running?"), true); }
  function post(rec) {
    var json = JSON.stringify(rec);
    if (o.mode === "gm") {
      GM_xmlhttpRequest({ method: "POST", url: o.server + "/api/capture", data: json, headers: { "Content-Type": "application/json" },
        onload: function (r) { if (r.status < 300) done(); else fail("Your site refused it: " + r.responseText); }, onerror: function () { fail(); } });
    } else {
      F.call(W, o.server + "/api/capture", { method: "POST", mode: "no-cors", body: json }).then(done, function () {
        var i = new Image();
        i.onload = i.onerror = done;
        i.src = o.server + "/api/capture?d=" + encodeURIComponent(btoa(unescape(encodeURIComponent(json))));
      });
    }
  }
  function hdrs(h, into) {
    if (!h) return;
    if (Array.isArray(h)) h.forEach(function (p) { into[p[0]] = p[1]; });
    else if (typeof h.forEach === "function") h.forEach(function (v, k) { into[k] = v; });
    else for (var k in h) into[k] = h[k];
  }
  function emit(method, url, h, body) {
    url = String(url);
    if (sent || !RE.test(url)) return;
    var low = {};
    for (var k in h) low[k.toLowerCase()] = 1;
    if (!low["authorization"]) return;
    try { url = new URL(url, location.href).href; } catch (e) {}
    var d = { "user-agent": navigator.userAgent, "accept": "application/json, text/plain, */*", "origin": location.origin, "referer": location.origin + "/", "accept-language": navigator.language };
    for (var n in d) if (!low[n]) h[n] = d[n];
    sent = true;
    post({ url: url, method: method || "GET", headers: h, body: typeof body === "string" ? body : null });
  }
  if (F) W.fetch = function (input, init) {
    try {
      var h = {}, isReq = typeof Request !== "undefined" && input instanceof Request;
      if (isReq) hdrs(input.headers, h);
      hdrs(init && init.headers, h);
      emit((init && init.method) || (isReq && input.method) || "GET", isReq ? input.url : input, h, init && init.body);
    } catch (e) {}
    return F.apply(W, arguments);
  };
  if (XP) {
    var open = XP.open, setH = XP.setRequestHeader, snd = XP.send;
    XP.open = function (m, u) { this.__br = { m: m, u: u, h: {} }; return open.apply(this, arguments); };
    XP.setRequestHeader = function (k, v) { if (this.__br) this.__br.h[k] = v; return setH.apply(this, arguments); };
    XP.send = function (b) { try { var r = this.__br; if (r) emit(r.m, r.u, r.h, b); } catch (e) {} return snd.apply(this, arguments); };
  }
  if (o.mode === "page") toast("Capture is on. Now click SEARCH TRAINS on this page.");
  if (o.auto) {
    var CAPKEY = "__brcap";
    try { if (/[?&]autocap=1/.test(location.search)) sessionStorage.setItem(CAPKEY, location.href); } catch (e) {}
    var want = "";
    try { want = sessionStorage.getItem(CAPKEY) || ""; } catch (e) {}
    if (want) {
      var n = 0, t = setInterval(function () {
        if (sent) { try { sessionStorage.removeItem(CAPKEY); } catch (e) {} return clearInterval(t); }
        if (++n > 300) return clearInterval(t);
        var onSearch = /\/booking\/train\/search/.test(location.pathname);
        if (!onSearch) {
          var loggedIn = false;
          try { for (var i = 0; i < localStorage.length; i++) if (/token/i.test(localStorage.key(i))) loggedIn = true; } catch (e) {}
          if (loggedIn && !/login/i.test(location.pathname)) location.href = want;
          return;
        }
        var b = [].slice.call(document.querySelectorAll("button,a")).filter(function (x) { return /search\s*trains/i.test(x.textContent); })[0];
        if (n > 4 && n % 6 === 0 && b && !b.disabled) b.click();
      }, 1000);
    }
  }
}"""


def userscript(port):
    head = ("// ==UserScript==\n// @name         BR ticket: send my search request to my site\n// @namespace    br-local\n"
            "// @version      2\n// @match        https://eticket.railway.gov.bd/*\n// @run-at       document-start\n"
            "// @grant        GM_xmlhttpRequest\n// @grant        unsafeWindow\n// @connect      localhost\n"
            "// @connect      127.0.0.1\n// ==/UserScript==\n")
    return head + "(" + HOOK_JS + ")({mode: \"gm\", server: \"http://localhost:%d\", close: true, auto: true});\n" % port


LOGIN_JS = r"""(function () {
  var SERVER = "__SERVER__";
  var q = new URLSearchParams(location.search);
  if (q.get("brsend") === "1") { try { sessionStorage.setItem("__brsend", "1"); } catch (e) {} }
  var armed = false;
  try { armed = sessionStorage.getItem("__brsend") === "1"; } catch (e) {}
  if (!armed) return;
  var sent = false, tries = 0;
  function toast(msg, bad) {
    var d = document.createElement("div");
    d.textContent = msg;
    d.style.cssText = "position:fixed;z-index:2147483647;top:12px;right:12px;padding:10px 14px;border-radius:8px;font:600 14px system-ui,sans-serif;color:#fff;background:" + (bad ? "#C8102E" : "#00573F");
    (document.body || document.documentElement).appendChild(d);
    setTimeout(function () { d.remove(); }, 6000);
  }
  function grab() {
    var o = {}, re = /token|uudid|ssdk|device|auth|user/i;
    for (var i = 0; i < localStorage.length; i++) {
      var k = localStorage.key(i);
      if (re.test(k)) o[k] = localStorage.getItem(k);
    }
    return o;
  }
  function token(o) {
    for (var k in o) if (/token/i.test(k)) return String(o[k]).replace(/^"|"$/g, "").replace(/^bearer\s+/i, "");
    return "";
  }
  function expired(t) {
    try {
      var p = JSON.parse(atob(t.split(".")[1].replace(/-/g, "+").replace(/_/g, "/")));
      return p.exp && p.exp * 1000 < Date.now() + 60000;
    } catch (e) { return false; }
  }
  var timer = setInterval(function () {
    if (sent) return;
    if (++tries > 900) { clearInterval(timer); return; }
    var o = grab(), t = token(o);
    if (!t || expired(t)) return;
    sent = true;
    GM_xmlhttpRequest({
      method: "POST", url: SERVER + "/api/login",
      data: JSON.stringify({ storage: o }), headers: { "Content-Type": "application/json" },
      onload: function (r) {
        if (r.status < 300) {
          clearInterval(timer);
          try { sessionStorage.removeItem("__brsend"); } catch (e) {}
          toast("Logged in on your site. This tab will close.");
          setTimeout(function () { try { window.close(); } catch (e) {} }, 1500);
        } else { sent = false; toast("Your site refused it: " + r.responseText, true); }
      },
      onerror: function () { sent = false; toast("Could not reach your site at " + SERVER + ". Is server.py running?", true); }
    });
  }, 1000);
})();"""


def login_userscript(port):
    head = ("// ==UserScript==\n// @name         BR ticket: auto login to my site\n// @namespace    br-local\n"
            "// @version      1\n// @match        https://eticket.railway.gov.bd/*\n// @run-at       document-idle\n"
            "// @grant        GM_xmlhttpRequest\n// @connect      localhost\n"
            "// @connect      127.0.0.1\n// ==/UserScript==\n")
    return head + LOGIN_JS.replace("__SERVER__", "http://localhost:%d" % port) + "\n"


PAGE = r"""<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>BR ticket info</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=Barlow+Condensed:wght@500;600;700&family=Hind+Siliguri:wght@400;500;600;700&display=swap">
<style>
:root{--display:"Barlow Condensed","Arial Narrow",system-ui,sans-serif;--body:"Hind Siliguri","Segoe UI","Nirmala UI",system-ui,sans-serif;--bg:#EEF3EC;--surface:#fff;--ink:#0F1C16;--mute:#56685F;--line:#D3DED5;--green:#0A4A36;--green2:#0E7C57;--on-green:#fff;--red:#D7263D;--sun:#F42A41;--amber:#8A5A00;--gold:#FFB81C;--ok:#0A7A4F;--band1:#0C5A40;--band2:#052B1D;--shadow:0 18px 38px -24px rgba(5,43,29,.6);color-scheme:light}
@media (prefers-color-scheme:dark){:root{--bg:#07110C;--surface:#101E17;--ink:#E6EFE9;--mute:#8FA298;--line:#25372D;--green:#43BE8E;--green2:#1E8F69;--on-green:#04130C;--red:#FF6B7F;--sun:#E8344A;--amber:#E3B341;--gold:#FFB81C;--ok:#4CD39A;--band1:#0B3A2B;--band2:#04150E;--shadow:0 18px 38px -22px rgba(0,0,0,.85);color-scheme:dark}}
*{box-sizing:border-box}
html{scroll-behavior:smooth}
body{margin:0;position:relative;color:var(--ink);font:400 16px/1.5 var(--body);background:var(--bg);min-height:100vh}
body::before{--r:clamp(52px,11vw,92px);content:"";position:absolute;inset:0 0 auto 0;height:372px;z-index:-1;background:radial-gradient(circle at 86% var(--sy),var(--sun) 0 var(--r),transparent calc(var(--r) + 1px)),radial-gradient(circle at 86% var(--sy),rgba(244,42,65,.2) 0 calc(var(--r) * 1.7),transparent calc(var(--r) * 1.7 + 1px)),linear-gradient(165deg,var(--band1),var(--band2))}
main{position:relative;max-width:920px;margin:0 auto;padding:18px 16px 72px}
.eyebrow{display:flex;align-items:center;gap:10px;font-size:15px;font-weight:500;color:rgba(255,255,255,.9);margin:20px 0 6px}
.eyebrow i{width:10px;height:10px;border-radius:50%;background:var(--gold);box-shadow:0 0 0 0 var(--gold);animation:ping 1.8s infinite}
@keyframes ping{0%{box-shadow:0 0 0 0 rgba(255,184,28,.65)}100%{box-shadow:0 0 0 11px transparent}}
h1{margin:0 0 10px;color:#fff;font:700 clamp(46px,9vw,80px)/.95 var(--display);letter-spacing:-.005em}
h1 .bn{display:block;margin-top:8px;color:var(--gold);font:600 clamp(20px,4vw,28px)/1.3 var(--body);letter-spacing:0}
.sub{color:var(--mute);margin:0 0 22px;max-width:52ch;font-size:17px}
.lede{color:rgba(255,255,255,.84)}
form{position:relative;margin-top:26px;background:var(--surface);color:var(--ink);border:1px solid var(--line);border-radius:16px;padding:24px 22px 22px 28px;display:grid;gap:14px;grid-template-columns:1fr auto 1fr;box-shadow:var(--shadow)}
form::before{content:"";position:absolute;left:0;top:0;bottom:0;width:6px;border-radius:16px 0 0 16px;background:linear-gradient(180deg,var(--green2) 0 72%,var(--sun) 72%)}
form label{display:grid;gap:5px;font-weight:600;font-size:14px;color:var(--mute)}
form input,form select{font:inherit;font-size:17px;font-weight:500;color:var(--ink);background:var(--bg);border:2px solid var(--line);border-radius:10px;padding:10px 14px;width:100%;min-height:50px;transition:border-color .15s}
form input:hover,form select:hover,form input:focus,form select:focus{border-color:var(--green2)}
:focus-visible{outline:3px solid var(--gold);outline-offset:2px}
.swap{align-self:end;min-height:48px;width:48px;border-radius:50%;cursor:pointer;border:2px solid var(--line);background:var(--surface);color:var(--green2);font-size:20px;transition:transform .35s,border-color .2s}
.swap:hover{transform:rotate(180deg);border-color:var(--green2)}
.row2{grid-column:1/-1;display:grid;gap:14px;grid-template-columns:1.2fr 1fr auto}
.go{align-self:end;min-height:50px;padding:0 30px;border:0;border-radius:10px;cursor:pointer;background:var(--gold);color:#1b1200;font:700 21px var(--display);letter-spacing:.01em;box-shadow:0 3px 0 rgba(0,0,0,.28);transition:transform .1s,box-shadow .1s}
.go:hover{transform:translateY(-1px);box-shadow:0 5px 0 rgba(0,0,0,.28)}
.go:active{transform:translateY(2px);box-shadow:0 1px 0 rgba(0,0,0,.28)}
.go:disabled{opacity:.6;cursor:wait}
.chips{display:flex;flex-wrap:wrap;gap:8px;margin:18px 0 0}
.chips button{border:1.5px solid var(--line);background:var(--surface);color:var(--ink);border-radius:8px;padding:6px 14px;font:inherit;font-size:14px;font-weight:600;cursor:pointer;transition:border-color .15s,background .15s}
.chips button:hover{border-color:var(--green2);background:var(--bg)}
.opts{display:flex;flex-wrap:wrap;gap:6px 22px;margin-top:14px;color:var(--mute);font-size:14px}
.opts label{display:flex;gap:8px;align-items:center;cursor:pointer}
input[type=checkbox]{accent-color:var(--green2);width:16px;height:16px}
#status{margin:16px 0 6px;color:var(--mute);min-height:1.5em}
#status.err{color:var(--red);background:color-mix(in srgb,var(--red) 10%,transparent);border:1px solid color-mix(in srgb,var(--red) 30%,transparent);border-radius:10px;padding:10px 14px}
#demoBanner{background:var(--red);color:#fff;border-radius:10px;padding:8px 14px;font-weight:600;margin:10px 0}
.route{display:flex;align-items:center;gap:16px;margin:32px 0 4px}
.route b{font:700 clamp(32px,6vw,48px)/1 var(--display)}
.rail{position:relative;flex:1;height:3px;background:repeating-linear-gradient(90deg,color-mix(in srgb,var(--ink) 55%,transparent) 0 9px,transparent 9px 14px)}
.loco{--tw:min(124px,52%);position:absolute;bottom:0;left:0;width:var(--tw);line-height:0}
.loco.go{background:none;box-shadow:none;padding:0;min-height:0;border-radius:0;font-size:26px;animation:ride 1.5s cubic-bezier(.3,.7,.3,1) forwards}
@keyframes ride{from{left:0}to{left:calc(100% - var(--tw))}}
@media (prefers-reduced-motion:reduce){.loco.go{animation:none;left:calc(100% - var(--tw))}.ticket,.eyebrow i,.pulse{animation:none!important}}
#meta{color:var(--mute);margin:0 0 10px}
.filters{display:flex;flex-wrap:wrap;gap:10px 18px;align-items:center;margin:8px 0 16px;font-size:14px;color:var(--mute)}
.filters label{display:flex;gap:8px;align-items:center;cursor:pointer}
.filters input[type=search]{font:inherit;padding:9px 14px;border:1.5px solid var(--line);border-radius:8px;background:var(--surface);color:var(--ink);min-width:220px}
#bookbox{display:flex;flex-wrap:wrap;gap:12px;align-items:center;justify-content:space-between;margin:0 0 18px}
#bookbox[hidden],.book[hidden]{display:none!important}
.book{display:inline-block;margin:0;padding:11px 18px;border-radius:10px;background:var(--green);color:var(--on-green);font-weight:600;text-decoration:none;transition:filter .15s}
.book:hover{filter:brightness(1.15)}
.deep-search-btn{display:inline-flex;align-items:center;gap:8px;padding:11px 18px;border-radius:10px;background:var(--ink);color:var(--gold);border:0;font:600 15px var(--body);cursor:pointer;transition:background .15s,color .15s}
.deep-search-btn:hover{background:var(--gold);color:#1b1200}
.deep-search-btn svg{width:18px;height:18px;fill:currentColor}
.ticket{position:relative;display:grid;grid-template-columns:1.25fr 1fr;background:var(--surface);border:1px solid var(--line);border-radius:14px;margin-bottom:16px;-webkit-mask:radial-gradient(circle 11px at 55.55% 0,#0000 97%,#000) top/100% 51% no-repeat,radial-gradient(circle 11px at 55.55% 100%,#0000 97%,#000) bottom/100% 51% no-repeat;mask:radial-gradient(circle 11px at 55.55% 0,#0000 97%,#000) top/100% 51% no-repeat,radial-gradient(circle 11px at 55.55% 100%,#0000 97%,#000) bottom/100% 51% no-repeat}
.ticket:not(:has(.seat.ok,.seat.low)){opacity:.68}
.ticket.animate-rise{animation:rise .5s both}
.ticket::before{content:"";position:absolute;left:0;top:0;bottom:0;width:6px;background:linear-gradient(180deg,var(--green2) 0 78%,var(--sun) 78%)}
.ticket:nth-child(2).animate-rise{animation-delay:.05s}.ticket:nth-child(3).animate-rise{animation-delay:.1s}.ticket:nth-child(4).animate-rise{animation-delay:.15s}.ticket:nth-child(5).animate-rise{animation-delay:.2s}.ticket:nth-child(n+6).animate-rise{animation-delay:.25s}
@keyframes rise{from{opacity:0;transform:translateY(12px)}to{opacity:1;transform:none}}
.main{padding:20px 20px 20px 28px}
.stub{padding:20px;border-left:2px dashed var(--line);display:grid;gap:12px;align-content:start}
.name{font:700 28px/1.1 var(--display);margin:0 0 8px;letter-spacing:.005em}
.badges{display:flex;flex-wrap:wrap;gap:6px;margin:0 0 14px;min-height:4px}
.badge{font:600 13px var(--body);padding:2px 10px;border-radius:6px;background:var(--gold);color:#1b1200}
.badge.b2{background:var(--ink);color:var(--surface)}
.times{display:flex;align-items:center;gap:16px;font:700 44px/1 var(--display);font-variant-numeric:tabular-nums}
.line{flex:1;height:2px;background:repeating-linear-gradient(90deg,var(--ink) 0 8px,transparent 8px 12px);position:relative;min-width:40px}
.line::before,.line::after{content:"";position:absolute;top:-4px;width:10px;height:10px;border-radius:50%;border:2px solid var(--green2);background:var(--surface)}
.line::before{left:-6px}.line::after{right:-6px}
.line small{position:absolute;top:12px;left:50%;transform:translateX(-50%);font:500 13px var(--body);color:var(--mute);white-space:nowrap}
.leaves{margin-top:24px;font-size:14px;color:var(--mute)}
.seat{display:grid;gap:5px;font-size:15px}
.seat .top{display:flex;justify-content:space-between;gap:10px;align-items:baseline}
.fare{color:var(--mute);font-size:13px}
.bar{display:block;height:9px;background:var(--line);overflow:hidden;-webkit-mask:repeating-linear-gradient(90deg,#000 0 7px,#0000 7px 10px);mask:repeating-linear-gradient(90deg,#000 0 7px,#0000 7px 10px)}
.bar i{display:block;height:100%;background:var(--ok);transition:width .5s}
.low .bar i{background:var(--gold)}.out .bar i{background:var(--red)}
.out{opacity:.72}
.n{font-weight:700}.ok .n{color:var(--ok)}.low .n{color:var(--amber)}.out .n{color:var(--red)}
details{margin-top:14px;font-size:14px;color:var(--mute)}
summary{cursor:pointer;min-height:32px}
textarea,pre{width:100%;font:13px/1.4 ui-monospace,Consolas,monospace;border:1px solid var(--line);border-radius:12px;background:var(--surface);color:var(--ink);padding:10px}
textarea{min-height:110px}
pre{max-height:260px;overflow:auto;white-space:pre-wrap;word-break:break-all}
.small{padding:8px 16px;border:0;border-radius:10px;background:linear-gradient(135deg,var(--green),var(--green2));color:var(--on-green);font:700 14px system-ui,sans-serif;cursor:pointer;margin:8px 8px 0 0;transition:transform .15s,filter .15s}
.small:hover{transform:translateY(-1px);filter:brightness(1.08)}
.acct{display:flex;flex-wrap:wrap;justify-content:flex-end;text-align:right;align-items:center;gap:8px;color:var(--mute);font-size:14px;margin-bottom:6px}
.acct .small{margin:0}
.adv{margin-top:28px;border:1px dashed var(--line);border-radius:16px;padding:6px 14px 12px;background:color-mix(in srgb,var(--surface) 55%,transparent)}
.advbar{display:flex;align-items:center;justify-content:space-between;gap:12px;flex-wrap:wrap;color:var(--mute);font-size:14px;padding-top:6px}
.advbar b{color:var(--ink)}
.advbar .small{margin:0;min-width:74px}
.pulse{animation:glow 1.4s ease-in-out infinite}
@keyframes glow{0%,100%{box-shadow:0 0 0 0 color-mix(in srgb,var(--gold) 70%,transparent)}50%{box-shadow:0 0 0 8px transparent}}
#connect{margin-top:10px}
#curl{min-height:0;height:46px;resize:none;transition:height .2s ease}
#curl:not(:focus){white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
#curl:focus{height:200px;resize:vertical}
.pastetitle{margin:18px 0 6px;font-size:19px;color:var(--ink)}
.btnrow{display:flex;align-items:center;flex-wrap:wrap;gap:0}
.btnrow .spacer{flex:1}
.btnrow .small.ghost{margin-right:0;background:transparent;border:1px solid var(--line);color:var(--ink)}
#curlSteps{margin:10px 0 0;padding:12px 14px 12px 14px;border:1px dashed var(--line);border-radius:12px}
#curlSteps ol{margin:0;padding-left:22px;line-height:1.9}
#curlSteps code{background:var(--surface);border:1px solid var(--line);border-radius:6px;padding:1px 6px;font-size:13px}
.autorow{display:flex;align-items:center;justify-content:space-between;gap:10px;flex-wrap:wrap}
.autorow .small{margin:0}
dialog{border:1px solid var(--line);border-radius:20px;background:var(--surface);color:var(--ink);padding:22px;width:min(440px,92vw);box-shadow:var(--shadow)}
dialog::backdrop{background:rgba(0,0,0,.55);backdrop-filter:blur(3px)}
dialog h2{margin:0 0 6px;font-size:21px}
dialog label{display:grid;gap:4px;font-size:14px;font-weight:500;margin:0 0 12px}
dialog input{font:inherit;color:var(--ink);background:var(--bg);border:1px solid var(--line);border-radius:10px;padding:10px 12px;width:100%;min-height:44px}
dialog hr{border:0;border-top:1px solid var(--line);margin:16px 0}
.fine{font-size:13px;color:var(--mute);margin:0 0 12px}
#loginNote{min-height:1.4em;font-size:14px;color:var(--amber);margin:10px 0 4px}
.steps{padding-left:20px;line-height:1.9}
.autosteps{list-style:none;margin:12px 0 4px;padding:0;font-size:14px;line-height:1.7}
.autosteps li{padding-left:26px;position:relative;color:var(--mute)}
.autosteps li::before{position:absolute;left:0;width:20px;text-align:center}
.autosteps li.todo::before{content:"\25CB"}
.autosteps li.now{color:inherit;font-weight:600}
.autosteps li.now::before{content:"\25CF";color:var(--gold);animation:stepPulse 1s infinite}
.autosteps li.done{color:inherit}
.autosteps li.done::before{content:"\2713";color:#3ecf6e}
.autosteps li.fail{color:var(--amber);font-weight:600}
.autosteps li.fail::before{content:"\2717";color:var(--amber)}
.autosteps li span{display:block;font-weight:400;font-size:13px;color:var(--mute)}
@keyframes stepPulse{50%{opacity:.25}}
.bm{display:inline-block;padding:6px 12px;border-radius:10px;background:var(--gold);color:#1b1500;font-weight:700;text-decoration:none;cursor:grab}
code{font:12px ui-monospace,Consolas,monospace;word-break:break-all}
@media (max-width:640px){form{grid-template-columns:1fr}.swap{justify-self:center}.row2{grid-template-columns:1fr}.ticket{grid-template-columns:1fr}.ticket::after{display:none}.stub{border-left:0;border-top:2px dashed var(--line)}}
.deepbar{display:flex;flex-wrap:wrap;gap:10px;align-items:center}
.deepbar input{font:inherit;font-size:15px;padding:11px 14px;border:1px solid var(--line);border-radius:12px;background:var(--surface);color:var(--ink);min-width:210px}
.small.ghost{background:transparent;border:1px solid var(--line);color:var(--ink)}
#deepOut{margin-top:6px}
.dtop{display:flex;flex-wrap:wrap;gap:10px 16px;align-items:center;justify-content:space-between;margin:22px 0 8px}
.dtop h2{margin:0;font-size:clamp(20px,4.4vw,28px);letter-spacing:-.02em}
.dctl{display:flex;flex-wrap:wrap;gap:8px 16px;align-items:center;font-size:14px;color:var(--mute)}
.dctl label{display:flex;gap:8px;align-items:center;cursor:pointer}
.dctl .small{margin:0}
.dprog{height:6px;border-radius:4px;background:var(--line);overflow:hidden;margin:8px 0 6px}
.dprog i{display:block;height:100%;width:0;background:linear-gradient(90deg,var(--ok),var(--gold));transition:width .25s}
#deepStatus{color:var(--mute);margin:6px 0 12px;min-height:1.5em}
#deepStatus.err{color:var(--red);background:color-mix(in srgb,var(--red) 10%,transparent);border:1px solid color-mix(in srgb,var(--red) 30%,transparent);border-radius:12px;padding:10px 14px}
.dgroup{margin:24px 0 6px}
.dhead{display:flex;flex-wrap:wrap;align-items:center;gap:8px 12px;margin:0 0 4px}
.dhead b{font-size:20px;letter-spacing:-.01em}
.dtag{font-size:12px;font-weight:800;padding:3px 11px;border-radius:999px;background:var(--green2);color:#fff}
.dtag.alias{background:var(--gold);color:#1b1500}
.dtag.wide{background:#0284c7;color:#fff}
.dtag.partial{background:transparent;border:1px solid var(--amber);color:var(--amber)}
.dnote{font-size:14px;color:var(--mute);margin:0 0 10px}
.deepbar input{border:1.5px solid var(--line);border-radius:10px;font-family:var(--body)}
.deepbar input:focus{border-color:var(--green2)}
.small.ghost{border:1.5px solid var(--line)}
.dtop h2{font:700 clamp(30px,5vw,42px)/1 var(--display);letter-spacing:0}
.dhead b{font:700 26px/1.1 var(--display);letter-spacing:.005em}
.dtag{font:600 12px var(--body);padding:2px 10px;border-radius:6px}
.dtag.wide{background:var(--ink);color:var(--surface)}
.dprog{height:9px;border-radius:0;-webkit-mask:repeating-linear-gradient(90deg,#000 0 7px,#0000 7px 10px);mask:repeating-linear-gradient(90deg,#000 0 7px,#0000 7px 10px)}
.dprog i{background:var(--ok)}
.acct{color:rgba(255,255,255,.88)}
.acct .small{background:rgba(255,255,255,.12);border:1px solid rgba(255,255,255,.45);color:#fff}
@media (max-width:640px){.ticket{-webkit-mask:none;mask:none}.times{font-size:38px}.main{padding:18px 16px 18px 24px}}
@property --sy{syntax:"<percentage>";inherits:false;initial-value:42%}
body::before{animation:sunrise 1.8s cubic-bezier(.2,.8,.2,1) both}
@keyframes sunrise{from{--sy:120%}}
@keyframes up{from{opacity:0;transform:translateY(18px)}}
@keyframes pop{from{opacity:0;transform:scale(.6)}}
@keyframes fill{from{transform:scaleX(0)}}
@keyframes flow{to{background-position:12px 0}}
@keyframes flowr{to{background-position:-14px 0}}
@keyframes cross{from{transform:translateX(-340px)}to{transform:translateX(100vw)}}
@keyframes stripes{to{background-position:34px 0}}
@keyframes live{0%{box-shadow:0 0 0 0 rgba(62,207,110,.7)}100%{box-shadow:0 0 0 9px transparent}}
.eyebrow{animation:up .7s .25s both cubic-bezier(.2,.7,.2,1)}
h1{animation:up .7s .35s both cubic-bezier(.2,.7,.2,1)}
.lede{animation:up .7s .5s both cubic-bezier(.2,.7,.2,1)}
form{animation:up .7s .65s both cubic-bezier(.2,.7,.2,1)}
.track{position:absolute;top:12px;left:0;width:100%;height:24px;overflow:hidden;pointer-events:none;z-index:-1}
.track::after{content:"";position:absolute;left:0;right:0;bottom:0;height:2px;background:rgba(255,255,255,.16)}
.track svg{position:absolute;top:0;left:0;color:rgba(255,255,255,.22);animation:cross 28s linear infinite}
.avatar{position:relative;flex:none;width:40px;height:40px;border-radius:50%;display:grid;place-items:center;color:#fff;background:rgba(255,255,255,.14);border:2px solid rgba(255,255,255,.45);transition:border-color .3s,background .3s}
.avatar svg{width:22px;height:22px;fill:currentColor}
.avatar::after{content:"";position:absolute;right:-3px;bottom:-3px;width:13px;height:13px;border-radius:50%;border:2px solid var(--band2);background:#9AA9A1}
.avatar[data-state=live]{border-color:var(--gold);background:rgba(255,184,28,.2);animation:pop .5s cubic-bezier(.3,1.6,.5,1)}
.avatar[data-state=live]::after{background:#3ecf6e;animation:live 1.8s infinite}
.avatar[data-state=expired]{border-color:var(--sun)}
.avatar[data-state=expired]::after{background:var(--sun)}
.go:disabled{opacity:1;background:repeating-linear-gradient(135deg,var(--gold) 0 12px,#FFD268 12px 24px);background-size:34px 34px;animation:stripes .7s linear infinite}
.rail{animation:flowr 1.1s linear infinite}
.line{animation:flow 1s linear infinite}
.ticket{transition:transform .25s cubic-bezier(.2,.8,.2,1)}
.ticket:hover{transform:translateY(-4px) rotate(-.2deg)}
.ticket.animate-rise .bar i{transform-origin:left;animation:fill .9s .3s both cubic-bezier(.2,.8,.2,1)}
.badge{animation:pop .45s .4s both cubic-bezier(.3,1.6,.5,1)}
@media (prefers-reduced-motion:reduce){*,*::before,*::after{animation:none!important;transition:none!important}.track{display:none}.loco.go{left:calc(100% - var(--tw))}}
.loco svg{display:block;width:100%;height:auto;overflow:visible}
@keyframes bob{to{transform:translateY(-1.5px)}}
.loco.go svg{animation:bob .15s ease-in-out 10 alternate}
.wh,.smk{transform-box:fill-box;transform-origin:center}
@keyframes spin{to{transform:rotate(360deg)}}
@keyframes puff{from{opacity:.8;transform:translate(0,0) scale(.4)}to{opacity:0;transform:translate(-16px,-14px) scale(1.7)}}
@keyframes sweep{0%,55%{transform:translateX(-130%)}100%{transform:translateX(130%)}}
.loco.go .wh{animation:spin .3s linear 5}
.smk{opacity:0;animation:puff 1.8s ease-out infinite}
:root{--deep:#4F3BE8;--deep2:#271A8C;--deepglow:#B7A8FF}
@media (prefers-color-scheme:dark){:root{--deep:#8E7CFF;--deep2:#4634C9;--deepglow:#D3CAFF}}
.deep-search-btn{position:relative;overflow:hidden;background:linear-gradient(135deg,var(--deep),var(--deep2));color:#fff;box-shadow:0 8px 20px -10px var(--deep2);transition:transform .15s,box-shadow .15s}
.deep-search-btn::after{content:"";position:absolute;inset:0;background:linear-gradient(100deg,transparent 30%,rgba(255,255,255,.38) 50%,transparent 70%);animation:sweep 3s ease-in-out infinite;pointer-events:none}
.deep-search-btn:hover{background:linear-gradient(135deg,var(--deep),var(--deep2));color:#fff;transform:translateY(-2px);box-shadow:0 12px 24px -10px var(--deep2)}
.deep-search-btn svg{position:relative;z-index:1}
#deepOut{border-top:4px solid var(--deep);padding-top:4px}
.deepbar input:focus{border-color:var(--deep)}
.dtag.wide{background:var(--deep);color:#fff}
.dprog i{background:linear-gradient(90deg,var(--deep),var(--deepglow))}
.dtop h2{color:var(--deep)}
.acct{margin-top:32px}

/* ================= Premium railway redesign — creative intensity 80% ================= */
:root{--bg:#06110d;--surface:#0d1e17;--ink:#edf7f0;--mute:#9aafa4;--line:#233d30;--green:#1db582;--green2:#43d6a1;--on-green:#04130d;--red:#ff5367;--sun:#ff5367;--amber:#ffd166;--gold:#ffbd2e;--ok:#42d9a0;--band1:#0b3c2b;--band2:#04120d;--shadow:0 24px 70px -32px rgba(0,0,0,.9);color-scheme:dark}
body{background:radial-gradient(ellipse at 12% 0%,rgba(22,143,94,.19),transparent 34rem),radial-gradient(ellipse at 95% 16%,rgba(255,83,103,.10),transparent 28rem),var(--bg);letter-spacing:.005em}
body::before{height:420px;background:radial-gradient(ellipse at 50% 0%,rgba(19,115,79,.35),transparent 68%),linear-gradient(155deg,#0b3024 0%,#071b13 65%,#06110d 100%);opacity:.98}
main{max-width:1000px;padding:24px 20px 84px}
.eyebrow{font-size:12px;text-transform:uppercase;letter-spacing:.16em;font-weight:700;color:#b8d6c7}
h1{font-size:clamp(48px,8.5vw,84px);letter-spacing:-.035em;text-shadow:0 8px 35px rgba(0,0,0,.18)}
h1 .bn{color:#ffc857}
.lede{max-width:58ch;color:#bfd0c6}
form{margin-top:30px;padding:26px 24px 24px 30px;border-radius:22px;background:linear-gradient(135deg,rgba(17,39,29,.97),rgba(9,24,17,.97));border:1px solid rgba(130,185,154,.22);box-shadow:0 26px 70px -35px #000, inset 0 1px rgba(255,255,255,.035);backdrop-filter:blur(16px)}
form::before{width:4px;background:linear-gradient(180deg,#43d6a1 0 72%,#ff5367 72%);box-shadow:0 0 18px rgba(67,214,161,.18)}
form label{color:#9eb8aa;font-size:12px;text-transform:uppercase;letter-spacing:.08em}
form input,form select{background:#07130e;border:1px solid #294638;border-radius:12px;color:#eef8f2;min-height:52px;box-shadow:inset 0 1px 5px rgba(0,0,0,.2)}
form input:focus,form select:focus{border-color:#43d6a1;box-shadow:0 0 0 3px rgba(67,214,161,.12)}
.swap{background:#0b2118;border-color:#294638;color:#53dba8}
.go{border-radius:12px;background:linear-gradient(135deg,#ffd166,#ffb51f);box-shadow:0 8px 22px -12px rgba(255,181,31,.85);font-size:22px}
.go:hover{filter:brightness(1.07);box-shadow:0 12px 28px -12px rgba(255,181,31,.8)}
.chips button{background:#0d2018;border:1px solid #284436;color:#d5e5dc;border-radius:10px;padding:7px 13px}
.chips button:hover{background:#153528;border-color:#43d6a1;color:#fff}
.opts{color:#9ab0a4}
.route{gap:clamp(10px,2vw,20px);margin:38px 0 8px}
.route b{font-size:clamp(30px,5.4vw,50px);letter-spacing:-.025em;text-shadow:0 5px 25px rgba(0,0,0,.2)}
.rail{height:2px;background:repeating-linear-gradient(90deg,rgba(175,208,190,.6) 0 8px,transparent 8px 13px);box-shadow:0 0 12px rgba(67,214,161,.10)}
#meta{font-size:13px;color:#91a99b}
.filters{color:#9bb1a5}
.filters input[type=search],.deepbar input{background:#0c1e16;border:1px solid #294638;color:#eaf5ee;border-radius:12px;min-height:44px}
.book{background:linear-gradient(135deg,#42d2a0,#25b986);color:#03150e;border-radius:11px;box-shadow:0 10px 24px -16px #35d19c;font-weight:700}
.ticket{background:linear-gradient(145deg,#10231a,#0b1912);border:1px solid #294436;border-radius:18px;box-shadow:0 16px 38px -30px #000;overflow:hidden}
.ticket::before{width:4px;background:linear-gradient(180deg,#43d6a1 0 78%,#ff5367 78%)}
.main{padding:23px 22px 23px 30px}
.stub{background:linear-gradient(145deg,rgba(17,39,29,.72),rgba(8,22,15,.72));border-left:1px dashed #355443;padding:22px}
.name{font-size:clamp(24px,3vw,30px);letter-spacing:.015em}
.badge{border-radius:7px;background:#d5f5e5;color:#073421}
.badge.b2{background:#19392b;color:#c5ded0}
.times{font-size:clamp(32px,4.6vw,46px);gap:12px}
.line{background:repeating-linear-gradient(90deg,#6f9882 0 7px,transparent 7px 12px)}
.line::before,.line::after{background:#10241a;border-color:#43d6a1}
.bar{background:#1b3528;border-radius:3px;height:8px}
.seat .top{font-size:14px}
.deep-search-btn{border-radius:12px;background:linear-gradient(135deg,#8b7bff,#6048e8);box-shadow:0 10px 24px -14px #8b7bff}
.deep-search-btn:hover{transform:translateY(-2px) scale(1.015)}
#deepOut{border-top-color:#8b7bff}
.adv{background:rgba(11,29,20,.82);border:1px solid #294436;border-radius:16px;padding:16px}
@media(max-width:640px){main{padding:16px 12px 56px}form{padding:20px 14px 18px 20px;border-radius:17px;gap:10px}.route{gap:8px;margin-top:28px}.route b{font-size:clamp(25px,7vw,34px)}.ticket{grid-template-columns:1fr}.stub{border-left:0;border-top:1px dashed #355443}.times{font-size:34px}.main{padding:19px 18px 20px 24px}}
/* ================= End premium railway redesign ================= */
/* ================= Extra motion (non-train) ================= */
body::after{content:"";position:absolute;inset:0 0 auto 0;height:420px;z-index:-1;pointer-events:none;opacity:.5;background:radial-gradient(circle,rgba(255,255,255,.5) 0 1px,transparent 1.6px) 0 0/90px 90px,radial-gradient(circle,rgba(255,200,87,.45) 0 1.2px,transparent 2px) 40px 30px/140px 140px;-webkit-mask:linear-gradient(#000 40%,transparent);mask:linear-gradient(#000 40%,transparent);animation:drift 50s linear infinite}
@keyframes drift{to{background-position:0 -90px,40px -110px}}
.seat.low .n{animation:lowp 1.3s ease-in-out infinite}
@keyframes lowp{50%{opacity:.45}}
.badge:not(.b2){background:linear-gradient(100deg,#d5f5e5 40%,#fff 50%,#d5f5e5 60%) 0 0/250% 100%;animation:pop .45s .4s both cubic-bezier(.3,1.6,.5,1),badgeShine 3.2s 1s ease-in-out infinite}
@keyframes badgeShine{to{background-position:-150% 0}}
#trip:not([hidden]) .route b{animation:up .6s both cubic-bezier(.2,.7,.2,1)}
#trip:not([hidden]) #rTo{animation-delay:.25s}
button.go{position:relative;overflow:hidden}
button.go::after{content:"";position:absolute;inset:0;background:linear-gradient(100deg,transparent 35%,rgba(255,255,255,.55) 50%,transparent 65%);transform:translateX(-120%)}
button.go:hover::after{transition:transform .6s;transform:translateX(120%)}
.chips button:active{transform:scale(.94)}
/* ================= End extra motion ================= */
</style>
</head>
<body>
<div class="track" aria-hidden="true"><svg viewBox="0 0 310 22" width="310" height="22" fill="currentColor"><path d="M0 22V10l9-6h24v18z"/><rect x="38" y="5" width="64" height="17" rx="2"/><rect x="106" y="5" width="64" height="17" rx="2"/><rect x="174" y="5" width="64" height="17" rx="2"/><rect x="242" y="5" width="64" height="17" rx="2"/><g fill="var(--band2)"><rect x="44" y="9" width="9" height="6" rx="1"/><rect x="58" y="9" width="9" height="6" rx="1"/><rect x="72" y="9" width="9" height="6" rx="1"/><rect x="86" y="9" width="9" height="6" rx="1"/><rect x="112" y="9" width="9" height="6" rx="1"/><rect x="126" y="9" width="9" height="6" rx="1"/><rect x="140" y="9" width="9" height="6" rx="1"/><rect x="154" y="9" width="9" height="6" rx="1"/><rect x="180" y="9" width="9" height="6" rx="1"/><rect x="194" y="9" width="9" height="6" rx="1"/><rect x="208" y="9" width="9" height="6" rx="1"/><rect x="222" y="9" width="9" height="6" rx="1"/><rect x="248" y="9" width="9" height="6" rx="1"/><rect x="262" y="9" width="9" height="6" rx="1"/><rect x="276" y="9" width="9" height="6" rx="1"/><rect x="290" y="9" width="9" height="6" rx="1"/></g></svg></div>
<main>
  <div class="acct">
    <span class="avatar" id="avatar" data-state="out" aria-hidden="true"><svg viewBox="0 0 24 24"><circle cx="12" cy="8.5" r="4"/><path d="M4 21c0-4.4 3.6-7 8-7s8 2.6 8 7z"/></svg></span>
    <span id="who">Not logged in</span>
    <button class="small" id="topConnectBtn" type="button">Connect</button>
    <button class="small" id="openLogin" type="button">Log in</button>
    <button class="small" id="logoutBtn" type="button" hidden>Log out</button>
  </div>
  <dialog id="loginDlg">
    <h2>Log in to Bangladesh Railway</h2>
    <p class="fine" id="autoMsg"><b>Waiting for you to log in on the railway tab.</b> Log in there and this page will pick it up by itself. The railway tab closes when it is done.</p>
    <p class="fine" id="autoInstall" hidden>Nothing arrived yet. <b>One time only:</b> install the Tampermonkey extension, then <a href="/br-login.user.js" target="_blank">open this script</a> and press Install. After that, login is fully automatic. <button class="small" id="retryRail" type="button">Open the railway tab again</button></p>
    <details>
    <summary>Other ways to log in (bookmark, paste, password)</summary>
    <ol class="fine steps">
      <li><b>One time only:</b> show your bookmarks bar (Ctrl+Shift+B) and drag this button onto it: <a id="bm" class="bm" href="#">Send to my site</a> <button class="small" id="copyBm" type="button">or copy its code</button></li>
      <li><button class="small" id="openRail" type="button">Open the railway login in a new tab</button> and log in there.</li>
      <li>On the railway tab, click the <b>Send to my site</b> bookmark. This page logs in by itself.</li>
    </ol>
    </details>
    <details>
      <summary>No bookmark? Paste the values by hand</summary>
      <p class="fine">On the railway site after logging in, press F12, open the <b>Console</b> tab, paste this line and press Enter. It copies your 3 values: <code>copy(['token','uudid','ssdk'].map(k=>k+'='+localStorage.getItem(k)).join('\n'))</code> Then paste here (Ctrl+V).</p>
      <textarea id="all" placeholder="token=...&#10;uudid=...&#10;ssdk=..." spellcheck="false"></textarea>
      <button class="small" id="doToken" type="button">Use pasted values</button>
    </details>
    <details>
      <summary>Try mobile number + password instead (usually blocked now)</summary>
      <label>Mobile number <input id="mob" type="tel" autocomplete="username" placeholder="01XXXXXXXXX"></label>
      <label>Password <input id="pw" type="password" autocomplete="current-password"></label>
      <button class="small" id="doLogin" type="button">Log in</button>
    </details>
    <div id="loginNote" role="status"></div>
    <button class="small" id="cancelLogin" type="button">Close</button>
  </dialog>
  <div class="eyebrow"><i></i> Live seats and fares from Bangladesh Railway</div>
  <h1>Where are you going?<span class="bn" lang="bn">কোথায় যাচ্ছেন?</span></h1>
  <p class="sub lede">Pick your two stations and a date. You will see only the trains for that trip, with live seats and fares.</p>

  <form id="f">
    <label>From <input id="from" list="stations" value="Dhaka" required autocomplete="off"></label>
    <button type="button" class="swap" id="swap" aria-label="Swap stations">&#8644;</button>
    <label>To <input id="to" list="stations" value="Chattogram" required autocomplete="off"></label>
    <div class="row2">
      <label>Date <input id="date" type="date" required></label>
      <label>Class <select id="cls"></select></label>
      <button class="go" id="go" type="submit">Search trains</button>
    </div>
  </form>
  <datalist id="stations"></datalist>
  <div class="chips" id="chips" aria-label="Available dates"></div>

  <div class="opts">
    <label><input type="checkbox" id="live" checked> Refresh every 5 seconds</label>
    <label><input type="checkbox" id="demo"> Preview with demo data</label>
  </div>

  <div id="status" role="status" aria-live="polite"></div>
  <div id="demoBanner" hidden>DEMO: made-up trains for previewing the layout. Do not use them to plan a trip.</div>

  <section id="trip" hidden>
    <div class="route"><b id="rFrom"></b><div class="rail"><span class="loco" id="loco" aria-hidden="true"><svg viewBox="0 0 170 32" aria-hidden="true"><rect x="0" y="7" width="52" height="18" rx="3" fill="#00573F"/><rect x="5" y="10" width="6" height="6" rx="1" fill="#FFD268"/><rect x="14" y="10" width="6" height="6" rx="1" fill="#FFD268"/><rect x="23" y="10" width="6" height="6" rx="1" fill="#FFD268"/><rect x="32" y="10" width="6" height="6" rx="1" fill="#FFD268"/><rect x="41" y="10" width="6" height="6" rx="1" fill="#FFD268"/><rect x="0" y="20" width="52" height="2.5" fill="#F42A41"/><rect x="56" y="7" width="52" height="18" rx="3" fill="#00573F"/><rect x="61" y="10" width="6" height="6" rx="1" fill="#FFD268"/><rect x="70" y="10" width="6" height="6" rx="1" fill="#FFD268"/><rect x="79" y="10" width="6" height="6" rx="1" fill="#FFD268"/><rect x="88" y="10" width="6" height="6" rx="1" fill="#FFD268"/><rect x="97" y="10" width="6" height="6" rx="1" fill="#FFD268"/><rect x="56" y="20" width="52" height="2.5" fill="#F42A41"/><rect x="52" y="20" width="4" height="2" fill="#1b1200"/><rect x="108" y="20" width="4" height="2" fill="#1b1200"/><g transform="translate(112 0)"><rect x="12" y="2" width="6" height="4" rx="1" fill="#00573F"/><path d="M3 25V9Q3 6 6 6H38L54 14V25Z" fill="#0E9467"/><path d="M3 9Q3 6 6 6H38L42 8H3Z" fill="#00573F"/><rect x="3" y="19" width="51" height="3" fill="#F42A41"/><rect x="8" y="10" width="10" height="7" rx="1.5" fill="#FFD268"/><rect x="22" y="10" width="10" height="7" rx="1.5" fill="#FFD268"/><path d="M40 9l9 4.5H40z" fill="#FFD268"/><circle cx="52" cy="17" r="1.6" fill="#FFF3B0"/><g fill="#E7F0EA"><circle class="smk" cx="15" cy="1" r="2.6"/><circle class="smk" style="animation-delay:.5s" cx="15" cy="1" r="2.6"/><circle class="smk" style="animation-delay:1s" cx="15" cy="1" r="2.6"/></g></g><g transform="translate(12 27)"><g class="wh"><circle r="3.2" fill="#1b1200" stroke="#E7F0EA" stroke-width="1"/><path d="M-2.6 0H2.6M0-2.6V2.6" stroke="#E7F0EA" stroke-width=".8"/></g></g><g transform="translate(40 27)"><g class="wh"><circle r="3.2" fill="#1b1200" stroke="#E7F0EA" stroke-width="1"/><path d="M-2.6 0H2.6M0-2.6V2.6" stroke="#E7F0EA" stroke-width=".8"/></g></g><g transform="translate(68 27)"><g class="wh"><circle r="3.2" fill="#1b1200" stroke="#E7F0EA" stroke-width="1"/><path d="M-2.6 0H2.6M0-2.6V2.6" stroke="#E7F0EA" stroke-width=".8"/></g></g><g transform="translate(96 27)"><g class="wh"><circle r="3.2" fill="#1b1200" stroke="#E7F0EA" stroke-width="1"/><path d="M-2.6 0H2.6M0-2.6V2.6" stroke="#E7F0EA" stroke-width=".8"/></g></g><g transform="translate(123 27)"><g class="wh"><circle r="3.2" fill="#1b1200" stroke="#E7F0EA" stroke-width="1"/><path d="M-2.6 0H2.6M0-2.6V2.6" stroke="#E7F0EA" stroke-width=".8"/></g></g><g transform="translate(135 27)"><g class="wh"><circle r="3.2" fill="#1b1200" stroke="#E7F0EA" stroke-width="1"/><path d="M-2.6 0H2.6M0-2.6V2.6" stroke="#E7F0EA" stroke-width=".8"/></g></g><g transform="translate(147 27)"><g class="wh"><circle r="3.2" fill="#1b1200" stroke="#E7F0EA" stroke-width="1"/><path d="M-2.6 0H2.6M0-2.6V2.6" stroke="#E7F0EA" stroke-width=".8"/></g></g><g transform="translate(159 27)"><g class="wh"><circle r="3.2" fill="#1b1200" stroke="#E7F0EA" stroke-width="1"/><path d="M-2.6 0H2.6M0-2.6V2.6" stroke="#E7F0EA" stroke-width=".8"/></g></g></svg></span></div><b id="rTo"></b></div>
    <p id="meta"></p>
    <div class="filters">
      <label><input type="checkbox" id="onlyAvail"> Only trains with seats</label>
      <input type="search" id="q" placeholder="Filter by train name" aria-label="Filter by train name">
    </div>
  </section>

  <div id="bookbox" hidden>
    <a class="book" id="bookA" href="#" target="_blank" rel="noopener"></a>
    <div class="deepbar">
      <input type="search" id="trainNameInput" placeholder="e.g. Subarna Express" aria-label="Train name for DeepSearch" autocomplete="off">
      <button type="button" class="deep-search-btn" id="deepSearchBtn">
        <svg viewBox="0 0 24 24"><path d="M15.5 14h-.79l-.28-.27A6.471 6.471 0 0 0 16 9.5 6.5 6.5 0 1 0 9.5 16c1.61 0 3.09-.59 4.23-1.57l.27.28v.79l5 4.99L20.49 19l-4.99-5zm-6 0C7.01 14 5 11.99 5 9.5S7.01 5 9.5 5 14 7.01 14 9.5 11.99 14 9.5 14z"/></svg>
        DeepSearch
      </button>
    </div>
  </div>
  <section id="deepOut" hidden>
    <div class="dtop">
      <h2 id="deepTitle">DeepSearch</h2>
      <div class="dctl">
        <label><input type="checkbox" id="deepPartial" checked> Include shorter segments</label>
        <label><input type="checkbox" id="deepWide"> Include longer tickets (stations beyond your range)</label>
        <button class="small" id="deepRescan" type="button">Re-scan</button>
        <button class="small ghost" id="deepBack" type="button">Back to trains</button>
      </div>
    </div>
    <div class="dprog" id="deepProg" hidden><i id="deepBar"></i></div>
    <p id="deepStatus" role="status" aria-live="polite"></p>
    <div id="deepRes"></div>
  </section>
  <div id="out"></div>

  <section class="adv">
  <div class="advbar"><span><b>More</b> (connect to the real railway request, server details; open this if you see 401 or no live data)</span><button class="small" id="connectBtn" type="button" aria-expanded="false" aria-controls="connect">More</button></div>
  <div id="connect" hidden>
    <details id="rawBox" hidden>
      <summary>Show what the railway server sent back</summary>
      <pre id="raw"></pre>
    </details>
    <div>
      <div id="detailsBox" hidden>
      <p class="fine">Opens the railway site with those choices, catches the search request and saves it here. Be logged in on the railway site. Set up once, either way:</p>
      <ul class="fine">
        <li><b>Full auto:</b> install the Tampermonkey extension, then open <a href="/br-capture.user.js" target="_blank">this script</a> and press Install. After that the railway tab searches, sends the request and closes by itself.</li>
        <li><b>No install:</b> drag <a id="bm2" class="bm" href="#">Capture search</a> onto your bookmarks bar. On the railway tab click it, then click SEARCH TRAINS.</li>
      </ul>
    <p><b>Manual way</b> (if the automatic way fails):</p>
    <ol>
      <li>Open <b>eticket.railway.gov.bd</b> and log in.</li>
      <li>Press <b>F12</b>, open the <b>Network</b> tab, then search any trip on that site.</li>
      <li>Find the request that returns the train list (try the filter <b>Fetch/XHR</b>). Right-click it, then <b>Copy &gt; Copy as cURL</b> (bash or cmd both work).</li>
      <li>Paste it below and press <b>Save request</b>. Then search again here.</li>
    </ol>
    </div>
    </div>
    <h3 class="pastetitle">Drop Your cURL Here</h3>
    <p>It contains your private login: never share it. It expires, so repeat this when 401 comes back. It is kept in memory only.</p>
    <textarea id="curl" placeholder="curl 'https://...' -H 'authorization: Bearer ...'" spellcheck="false"></textarea>
    <div class="btnrow">
      <button class="small" id="saveReq" type="button">Save request</button>
      <button class="small" id="clearReq" type="button">Forget it</button>
      <span class="spacer"></span>
      <button class="small ghost" id="curlDetailsBtn" type="button" aria-expanded="false" aria-controls="curlSteps">Details</button>
    </div>
    <div id="curlSteps" hidden>
      <p class="fine"><b>How to get your cURL</b></p>
      <ol>
        <li>Open the railway ticket website and log in.</li>
        <li>Press <b>F12</b> and go to the <b>Network</b> tab.</li>
        <li>Search for a train (Dhaka to Khulna) so a request appears.</li>
        <li>Right-click the request (such as <code>search-trips</code> or <code>trains</code>), then choose <b>Copy &rarr; Copy as cURL</b>.</li>
        <li>Paste it in the box above and click <b>Save request</b>.</li>
      </ol>
    </div>
    <div id="reqNote" role="status"></div>
    <ol id="autoSteps" class="autosteps" aria-live="polite" hidden></ol>
  </div>
  </section>
</main>

<script>
const $ = (s) => document.querySelector(s);
const STATIONS = ["Dhaka","Chattogram","Sylhet","Rajshahi","Khulna","Rangpur","Dinajpur","Mymensingh","Cumilla","Cox's Bazar","Bhairab Bazar","Brahmanbaria","Akhaura","Jamalpur","Tangail","Sirajganj","Bogura","Ishwardi","Jashore","Kishoreganj","Parbatipur","Lalmonirhat","Noakhali","Feni","Alamdanga","Poradaha"];
const CLASSES = [["","All classes"],["SNIGDHA","Snigdha"],["S_CHAIR","Shovan chair"],["SHOVAN","Shovan"],["SHULOV","Shulov"],["AC_S","AC seat"],["AC_B","AC berth"],["F_SEAT","First seat"],["F_BERTH","First berth"],["F_CHAIR","First chair"]];
const LABEL = Object.fromEntries(CLASSES);
const MON = ["Jan","Feb","Mar","Apr","May","Jun","Jul","Aug","Sep","Oct","Nov","Dec"];

$("#stations").innerHTML = STATIONS.map((s) => `<option value="${s}">`).join("");
$("#cls").innerHTML = CLASSES.map(([v, l]) => `<option value="${v}">${l}</option>`).join("");
const now0 = new Date();
now0.setMinutes(now0.getMinutes() - now0.getTimezoneOffset());
$("#date").value = now0.toISOString().slice(0, 10);
$("#date").min = $("#date").value;
$("#swap").onclick = () => { const a = $("#from").value; $("#from").value = $("#to").value; $("#to").value = a; };

const availableDates = [];
for (let i = 0; i < 10; i++) {
  const d = new Date();
  d.setDate(d.getDate() + i);
  const iso = d.toISOString().slice(0, 10);
  const label = d.toLocaleDateString("en-US", { month: "short", day: "numeric" });
  availableDates.push({ iso, label });
}
$("#chips").innerHTML = availableDates.map((d) => `<button type="button" data-date="${d.iso}">${d.label}</button>`).join("");
$("#chips").onclick = (e) => {
  const b = e.target.closest("button"); if (!b) return;
  $("#date").value = b.dataset.date; search(false);
};

const esc = (s) => String(s ?? "").replace(/[&<>"]/g, (c) => ({"&":"&amp;","<":"&lt;",">":"&gt;",'"':"&quot;"}[c]));
const clock = (s) => String(s || "").split(",").pop().trim();

function normalize(json) {
  const list = json?.data?.trains || json?.trains || json?.data?.data?.trains || [];
  const num = (v) => { const n = parseFloat(v); return Number.isFinite(n) ? n : 0; };
  const seatCount = (s) => {
    const c = s.seat_counts;
    if (c && typeof c === "object") return num(c.online) + num(c.offline);
    return num(s.n ?? s.available ?? s.seats ?? 0);
  };
  const fareWithVat = (s) => {
    const base = s.fare ?? s.price;
    if (base == null || base === "") return null;
    const vatKey = Object.keys(s).find((k) => /vat/i.test(k) && !/percent|rate|%/i.test(k));
    const total = num(base) + (vatKey ? num(s[vatKey]) : 0);
    return Math.ceil(total - 0.001);
  };
  const trainName = (t) => {
    const direct = t.trip_number || t.train_name || t.name || t.train_model;
    if (direct) return String(direct);
    const k = Object.keys(t).find((k) => /name|trip|model/i.test(k) && typeof t[k] === "string" && t[k]);
    return k ? t[k] : "Train";
  };
  return list.map((t) => ({
    name: trainName(t),
    dep: t.departure_date_time || t.dep,
    arr: t.arrival_date_time || t.arr,
    dur: t.travel_time || t.dur || "",
    seats: (t.seat_types || t.seats || []).map((s) => ({
      cls: s.type || s.cls,
      fare: fareWithVat(s),
      n: seatCount(s)
    }))
  }));
}

function sample(from, to, date) {
  let h = 0;
  for (const c of from + to + date) h = (h * 31 + c.charCodeAt(0)) >>> 0;
  const rnd = () => (h = (h * 1664525 + 1013904223) >>> 0) / 4294967296;
  const trains = [["Demo Express A","06:40 AM","12:50 PM","6h 10m"],["Demo Intercity B","09:30 AM","04:15 PM","6h 45m"],["Demo Night Mail C","11:00 PM","06:10 AM","7h 10m"]];
  const classes = [["SNIGDHA",1350],["S_CHAIR",1050],["SHOVAN",760],["AC_S",1600]];
  return trains.map(([name, dep, arr, dur]) => ({ name, dep, arr, dur,
    seats: classes.map(([c, fare]) => ({cls: c, fare, n: Math.floor(rnd() * rnd() * 60)})) }));
}

function depTime(t, date) {
  const m = String(t.dep || "").match(/(\d{1,2}):(\d{2})\s*([AP]M)?/i);
  if (!m) return null;
  let h = +m[1]; const ap = (m[3] || "").toUpperCase();
  if (ap === "PM" && h < 12) h += 12;
  if (ap === "AM" && h === 12) h = 0;
  const [y, mo, d] = date.split("-").map(Number);
  return new Date(y, mo - 1, d, h, +m[2]);
}
function leavesIn(dt) {
  if (!dt) return "";
  const mins = Math.round((dt - new Date()) / 60000);
  if (mins < -30) return "Already left";
  if (mins < 0) return "Leaving now";
  if (mins > 48 * 60) return "";
  return "Leaves in " + (mins >= 60 ? Math.floor(mins / 60) + "h " : "") + (mins % 60) + "m";
}

let all = [], ctx = {};

function renderTicketHTML(t, cheapest, earliest, listLength, animate) {
  const open = t.seats.filter((s) => s.n > 0);
  const d = depTime(t, ctx.date);
  const badges = [];
  if (open.some((s) => s.fare != null && +s.fare === cheapest)) badges.push('<span class="badge">Cheapest seat</span>');
  if (d && +d === earliest && listLength > 1) badges.push('<span class="badge b2">Leaves first</span>');
  if (!open.length) badges.push('<span class="badge b2">Sold out</span>');
  const animClass = animate ? ' animate-rise' : '';
  return `
    <article class="ticket${animClass}" data-name="${esc(t.name)}">
      <div class="main">
        <h2 class="name">${esc(t.name)}</h2>
        <div class="badges">${badges.join("")}</div>
        <div class="times"><span>${esc(clock(t.dep))}</span><span class="line"><small>${esc(t.dur)}</small></span><span>${esc(clock(t.arr))}</span></div>
        <div class="leaves">${esc(leavesIn(d))}</div>
      </div>
      <div class="stub">${t.seats.map((s) => {
        const k = s.n === 0 ? "out" : s.n < 10 ? "low" : "ok";
        const pct = s.n === 0 ? 0 : Math.max(5, Math.min(100, s.n / 60 * 100));
        return `<div class="seat ${k}" data-cls="${esc(s.cls)}"><div class="top"><span><b>${esc(LABEL[s.cls] || s.cls)}</b> <span class="fare">${s.fare != null ? "&#2547;" + esc(s.fare) : ""}</span></span><span class="n">${s.n === 0 ? "Sold out" : s.n + " seats"}</span></div><span class="bar"><i style="width:${pct}%"></i></span></div>`;
      }).join("")}</div>
    </article>`;
}

function updateTicketDOM(card, t, cheapest, earliest, listLength) {
  const open = t.seats.filter((s) => s.n > 0);
  const d = depTime(t, ctx.date);

  const badges = [];
  if (open.some((s) => s.fare != null && +s.fare === cheapest)) badges.push('<span class="badge">Cheapest seat</span>');
  if (d && +d === earliest && listLength > 1) badges.push('<span class="badge b2">Leaves first</span>');
  if (!open.length) badges.push('<span class="badge b2">Sold out</span>');
  const badgesEl = card.querySelector(".badges");
  if (badgesEl && badgesEl.innerHTML !== badges.join("")) {
    badgesEl.innerHTML = badges.join("");
  }

  const leavesEl = card.querySelector(".leaves");
  if (leavesEl) {
    const text = leavesIn(d);
    if (leavesEl.textContent !== text) leavesEl.textContent = text;
  }

  const stubEl = card.querySelector(".stub");
  if (stubEl) {
    t.seats.forEach((s) => {
      let seatEl = stubEl.querySelector(`.seat[data-cls="${CSS.escape(s.cls)}"]`);
      const k = s.n === 0 ? "out" : s.n < 10 ? "low" : "ok";
      const pct = s.n === 0 ? 0 : Math.max(5, Math.min(100, s.n / 60 * 100));
      const fareText = s.fare != null ? "৳" + s.fare : "";
      const nText = s.n === 0 ? "Sold out" : s.n + " seats";

      if (seatEl) {
        seatEl.className = `seat ${k}`;
        const fareEl = seatEl.querySelector(".fare");
        if (fareEl && fareEl.textContent !== fareText) fareEl.textContent = fareText;
        const nEl = seatEl.querySelector(".n");
        if (nEl && nEl.textContent !== nText) nEl.textContent = nText;
        const barI = seatEl.querySelector(".bar i");
        if (barI) barI.style.width = pct + "%";
      } else {
        const temp = document.createElement("div");
        temp.innerHTML = `<div class="seat ${k}" data-cls="${esc(s.cls)}"><div class="top"><span><b>${esc(LABEL[s.cls] || s.cls)}</b> <span class="fare">${s.fare != null ? "&#2547;" + esc(s.fare) : ""}</span></span><span class="n">${s.n === 0 ? "Sold out" : s.n + " seats"}</span></div><span class="bar"><i style="width:${pct}%"></i></span></div>`;
        stubEl.appendChild(temp.firstElementChild);
      }
    });
  }
}

function view(isAutoRefresh) {
  const q = $("#q").value.trim().toLowerCase(), only = $("#onlyAvail").checked;
  const specificTrain = $("#trainNameInput").value.trim().toLowerCase();
  let list = all, note = "";

  if (specificTrain) {
    list = list.filter((t) => String(t.name || "").toLowerCase().includes(specificTrain));
  }

  if (ctx.cls) {
    const f = list.map((t) => ({...t, seats: t.seats.filter((s) => s.cls === ctx.cls)})).filter((t) => t.seats.length);
    if (f.length) list = f; else note = "No train in this answer lists " + (LABEL[ctx.cls] || ctx.cls) + ", so every class is shown. ";
  }
  if (only) list = list.map((t) => ({...t, seats: t.seats.filter((s) => s.n > 0)})).filter((t) => t.seats.length);
  if (q) list = list.filter((t) => String(t.name || "").toLowerCase().includes(q));

  let cheapest = Infinity, earliest = Infinity;
  list.forEach((t) => {
    const open = t.seats.filter((s) => s.n > 0);
    open.forEach((s) => { if (s.fare != null && +s.fare < cheapest) cheapest = +s.fare; });
    const d = depTime(t, ctx.date);
    if (open.length && d && +d < earliest) earliest = +d;
  });

  const out = $("#out");
  if (!list.length) {
    out.innerHTML = '<p class="sub">No trains match these filters. Untick "Only trains with seats", check the train name, or clear filters.</p>';
  } else {
    const currentCards = Array.from(out.querySelectorAll(".ticket"));
    const currentNames = currentCards.map(c => c.dataset.name);
    const newNames = list.map(t => t.name);

    if (!isAutoRefresh || currentNames.join("|") !== newNames.join("|")) {
      out.innerHTML = list.map((t) => renderTicketHTML(t, cheapest, earliest, list.length, !isAutoRefresh)).join("");
    } else {
      list.forEach((t, i) => {
        updateTicketDOM(currentCards[i], t, cheapest, earliest, list.length);
      });
    }
  }

  const when = ctx.demo ? "demo data" : "live seats as of " + ctx.at.toLocaleTimeString([], {hour: "2-digit", minute: "2-digit", second: "2-digit"});
  $("#meta").textContent = note + list.length + " of " + all.length + " trains shown, " + when + ".";
}
$("#onlyAvail").onchange = () => view(false);
$("#q").oninput = () => view(false);

function bookLink(from, to, date, cls) {
  const [y, m, d] = date.split("-");
  const u = "https://eticket.railway.gov.bd/booking/train/search?fromcity=" + encodeURIComponent(from) +
    "&tocity=" + encodeURIComponent(to) + "&doj=" + d + "-" + MON[+m - 1] + "-" + y + (cls ? "&class=" + cls : "");
  const a = $("#bookA");
  a.href = u; a.textContent = "Open the official site to book " + from + " to " + to; a.hidden = false;
  $("#bookbox").hidden = false;
}

/* ================= DeepSearch =================
   Scans nearby, longer and shorter station combinations for your trip and lists every
   train that still has seats. Edit CORRIDORS to teach it more lines (stations in order). */
const CORRIDORS = [
  ["Dhaka","Airport","Bhairab Bazar","Brahmanbaria","Akhaura","Cumilla","Feni","Chattogram","Cox's Bazar"],
  ["Dhaka","Airport","Bhairab Bazar","Brahmanbaria","Akhaura","Srimangal","Moulvibazar","Sylhet"],
  ["Dhaka","Airport","Bhanga","Jashore","Noapara","Khulna"],
  ["Dhaka","Airport","Rajbari","Poradaha","Alamdanga","Chuadanga","Darshana_Halt","Kotchandpur","Jashore","Noapara","Daulatpur","Khulna"],
  ["Dhaka","Airport","Joydebpur","Tangail","Ishwardi","Rajshahi"],
  ["Dhaka","Airport","Joydebpur","Tangail","Sirajganj","Bogura","Rangpur"],
  ["Dhaka","Airport","Joydebpur","Tangail","Sirajganj","Bogura","Parbatipur","Dinajpur"],
  ["Dhaka","Airport","Joydebpur","Mymensingh","Jamalpur"]
];
const ALIAS_GROUPS = [["Dhaka","Airport"]];
const KIND = { exact: ["Your exact route", 0], alias: ["Nearby station", 1], wide: ["Longer ticket, covers your trip", 2], partial: ["Only part of your trip", 3] };

const deep = { active: false, running: false, silent: false, id: 0, groups: [], from: "", to: "", date: "", total: 0, done: 0, errors: 0, firstErr: "", finishedAt: 0, at: null };

const keyOf = (s) => String(s || "").toLowerCase().replace(/[^a-z]/g, "");
const cap = (s) => s.charAt(0).toUpperCase() + s.slice(1);
const aliasesOf = (name) => {
  const k = keyOf(name);
  const g = ALIAS_GROUPS.find((g) => g.some((x) => keyOf(x) === k));
  return g ? g.filter((x) => keyOf(x) !== k) : [];
};

function deepPairs(from, to, withPartial, withWide) {
  const found = new Map();
  const add = (f, t, kind, notes) => {
    if (keyOf(f) === keyOf(t)) return;
    const id = keyOf(f) + ">" + keyOf(t);
    if (!found.has(id)) found.set(id, { from: f, to: t, kind, notes: notes.filter(Boolean) });
  };
  const sNote = (s) => s.k === "alias" ? "board at " + s.n + " instead of " + from : s.k === "wide" ? "ticket is for " + s.n + " \u2192 " + to + "; you board at " + from : "";
  const eNote = (e) => e.k === "alias" ? "get off at " + e.n + " instead of " + to : e.k === "wide" ? "ticket is for " + from + " \u2192 " + e.n + "; you get off at " + to + " (costs more than a direct ticket)" : "";
  const starts = [{ n: from, k: "exact" }, ...aliasesOf(from).map((n) => ({ n, k: "alias" }))];
  const ends = [{ n: to, k: "exact" }, ...aliasesOf(to).map((n) => ({ n, k: "alias" }))];
  const combine = (S, E) => S.forEach((s) => E.forEach((e) => {
    const kind = s.k === "wide" || e.k === "wide" ? "wide" : (s.k === "exact" && e.k === "exact" ? "exact" : "alias");
    add(s.n, e.n, kind, [sNote(s), eNote(e)]);
  }));
  combine(starts, ends);
  const skip = new Set([from, to, ...aliasesOf(from), ...aliasesOf(to)].map(keyOf));
  for (const corridor of CORRIDORS) {
    let L = corridor.slice();
    const at = (x) => L.findIndex((n) => keyOf(n) === keyOf(x));
    let iF = at(from), iT = at(to);
    if (iF < 0 || iT < 0) continue;
    if (iF > iT) { L.reverse(); iF = at(from); iT = at(to); }
    const S = [...starts], E = [...ends];
    // Stations outside the searched range are added only when "Include longer tickets" is ticked.
    if (withWide) {
      L.slice(0, iF).filter((n) => !skip.has(keyOf(n))).forEach((n) => S.push({ n, k: "wide" }));
      L.slice(iT + 1).filter((n) => !skip.has(keyOf(n))).forEach((n) => E.push({ n, k: "wide" }));
    }
    combine(S, E);
    if (withPartial) {
      L.slice(iF + 1, iT).filter((m) => !skip.has(keyOf(m))).slice(0, 14).forEach((m) => {
        add(from, m, "partial", ["only goes as far as " + m + ", so " + to + " is not covered"]);
        add(m, to, "partial", ["only starts from " + m + ", so " + from + " is not covered"]);
      });
    }
  }
  return [...found.values()].sort((a, b) => KIND[a.kind][1] - KIND[b.kind][1]).slice(0, 90);
}

async function scanPair(p, date) {
  try {
    let trains;
    if ($("#demo").checked) {
      trains = sample(p.from, p.to, date);
    } else {
      const res = await fetch("/api/search?" + new URLSearchParams({ from: p.from, to: p.to, date }));
      const text = await res.text();
      let body = {};
      try { body = JSON.parse(text); } catch (x) {}
      if (!res.ok) return { error: body.error || "status " + res.status, trains: [] };
      trains = normalize(body);
    }
    trains = trains.map((t) => ({ ...t, seats: t.seats.filter((s) => s.n > 0) })).filter((t) => t.seats.length);
    trains.sort((a, b) => (+depTime(a, date) || 0) - (+depTime(b, date) || 0));
    return { trains };
  } catch (e) {
    return { error: e.message || "network error", trains: [] };
  }
}

function deepTicket(t) {
  const d = depTime(t, deep.date);
  return `
    <article class="ticket">
      <div class="main">
        <h2 class="name">${esc(t.name)}</h2>
        <div class="times"><span>${esc(clock(t.dep))}</span><span class="line"><small>${esc(t.dur)}</small></span><span>${esc(clock(t.arr))}</span></div>
        <div class="leaves">${esc(leavesIn(d))}</div>
      </div>
      <div class="stub">${t.seats.map((s) => {
        const k = s.n < 10 ? "low" : "ok";
        const pct = Math.max(5, Math.min(100, s.n / 60 * 100));
        return `<div class="seat ${k}"><div class="top"><span><b>${esc(LABEL[s.cls] || s.cls)}</b> <span class="fare">${s.fare != null ? "&#2547;" + esc(s.fare) : ""}</span></span><span class="n">${s.n} seats</span></div><span class="bar"><i style="width:${pct}%"></i></span></div>`;
      }).join("")}</div>
    </article>`;
}

function renderDeep() {
  const name = $("#trainNameInput").value.trim().toLowerCase();
  const groups = deep.groups
    .map((g) => ({ ...g, trains: g.trains.filter((t) => !name || String(t.name || "").toLowerCase().includes(name)) }))
    .filter((g) => g.trains.length)
    .sort((a, b) => KIND[a.kind][1] - KIND[b.kind][1] || a.from.localeCompare(b.from) || a.to.localeCompare(b.to));

  $("#deepTitle").textContent = "DeepSearch: " + deep.from + " \u2192 " + deep.to;
  const pct = deep.total ? Math.round(deep.done / deep.total * 100) : 0;
  $("#deepProg").hidden = !deep.running;
  $("#deepBar").style.width = pct + "%";

  const st = $("#deepStatus");
  const nTrains = groups.reduce((n, g) => n + g.trains.length, 0);
  st.className = "";
  if (deep.running) {
    st.textContent = (deep.silent ? "Refreshing in the background" : "Scanning station combinations") + " (" + deep.done + "/" + deep.total + ")" +
      (nTrains ? " \u00b7 " + nTrains + " train" + (nTrains === 1 ? "" : "s") + " with seats so far" : "") + "...";
  } else if (deep.total && deep.errors === deep.total && deep.groups.length) {
    st.textContent = "Refresh failed (" + deep.firstErr + "), showing the last successful scan.";
  } else if (deep.total && deep.errors === deep.total) {
    st.className = "err";
    st.textContent = "Could not load any trains (" + deep.firstErr + "). Press \"More\" below to connect your railway login, then press Re-scan.";
  } else if (!nTrains) {
    st.textContent = name && deep.groups.length
      ? "No train with that name has seats in these results."
      : "No seats on any station combination right now. Seats come back when people cancel; with \"Refresh\" ticked this scan repeats every 30 seconds.";
  } else {
    st.textContent = nTrains + " train" + (nTrains === 1 ? "" : "s") + " with seats across " + groups.length + " of " + deep.total +
      " station combinations (all seat classes), live as of " + (deep.at || new Date()).toLocaleTimeString([], { hour: "2-digit", minute: "2-digit", second: "2-digit" }) + ".";
  }

  $("#deepRes").innerHTML = groups.map((g) => `
    <section class="dgroup">
      <div class="dhead"><b>${esc(g.from)} &rarr; ${esc(g.to)}</b><span class="dtag ${g.kind}">${KIND[g.kind][0]}</span></div>
      ${g.notes.length ? '<p class="dnote">' + esc(cap(g.notes.join("; "))) + ".</p>" : ""}
      ${g.trains.map(deepTicket).join("")}
    </section>`).join("");
}

async function runDeepSearch(from, to, date, silent) {
  const id = ++deep.id;
  const pairs = deepPairs(from, to, $("#deepPartial").checked, $("#deepWide").checked);
  Object.assign(deep, { active: true, running: true, silent: !!silent, from, to, date, total: pairs.length, done: 0, errors: 0, firstErr: "" });
  const fresh = [];
  if (!silent) deep.groups = fresh;
  $("#deepOut").hidden = false; $("#out").hidden = true; $("#trip").hidden = true;
  $("#demoBanner").hidden = !$("#demo").checked;
  $("#status").className = ""; $("#status").textContent = "";
  renderDeep();
  if (!silent) $("#deepOut").scrollIntoView({ behavior: "smooth", block: "start" });

  let next = 0;
  const worker = async () => {
    while (next < pairs.length) {
      const p = pairs[next++];
      const r = await scanPair(p, date);
      if (id !== deep.id) return;
      deep.done++;
      if (r.error) { deep.errors++; deep.firstErr = deep.firstErr || r.error; }
      else if (r.trains.length) fresh.push({ ...p, trains: r.trains });
      renderDeep();
    }
  };
  await Promise.all(Array.from({ length: Math.min(6, pairs.length) }, worker));
  if (id !== deep.id) return;
  if (!(silent && deep.errors === deep.total)) deep.groups = fresh;
  deep.running = false; deep.finishedAt = Date.now(); deep.at = new Date();
  renderDeep();
}

function exitDeep() {
  if (!deep.active) return;
  deep.active = false; deep.running = false; deep.id++;
  $("#deepOut").hidden = true; $("#out").hidden = false;
  $("#trip").hidden = !all.length; $("#demoBanner").hidden = !ctx.demo;
}
$("#deepRescan").onclick = () => runDeepSearch(deep.from, deep.to, deep.date);
$("#deepPartial").onchange = () => { if (deep.active) runDeepSearch(deep.from, deep.to, deep.date); };
$("#deepWide").onchange = () => { if (deep.active) runDeepSearch(deep.from, deep.to, deep.date); };
$("#deepBack").onclick = exitDeep;

// DeepSearch button + train name box: typing a name starts the scan by itself
function startDeep() {
  const from = $("#from").value.trim(), to = $("#to").value.trim(), date = $("#date").value;
  if (!from || !to || !date) return;
  if (keyOf(from) === keyOf(to)) { $("#status").className = "err"; $("#status").textContent = "Choose two different stations."; return; }
  runDeepSearch(from, to, date);
}
$("#deepSearchBtn").onclick = startDeep;
let nameTimer;
$("#trainNameInput").oninput = () => {
  clearTimeout(nameTimer);
  view(false);
  const name = $("#trainNameInput").value.trim();
  const same = deep.active && deep.from === $("#from").value.trim() && deep.to === $("#to").value.trim() && deep.date === $("#date").value;
  if (same) { renderDeep(); return; }
  if (!name) return;
  nameTimer = setTimeout(startDeep, 600);
};
$("#trainNameInput").onkeydown = (e) => { if (e.key === "Enter") { e.preventDefault(); clearTimeout(nameTimer); startDeep(); } };

let busy = false;
async function search(auto) {
  if (busy) return;
  if (!auto) exitDeep();
  const from = $("#from").value.trim(), to = $("#to").value.trim();
  const date = $("#date").value, cls = $("#cls").value;
  const status = $("#status"), btn = $("#go"), demo = $("#demo").checked;
  if (!from || !to || !date) return;
  if (from.toLowerCase() === to.toLowerCase()) {
    status.className = "err"; status.textContent = "Choose two different stations."; return;
  }
  busy = true;
  if (!auto) {
    btn.disabled = true;
    status.className = ""; status.textContent = "Searching " + from + " to " + to + "..."; $("#rawBox").hidden = true;
  }

  let trains = [], failed = "", rawText = "", lastDetail = "";
  if (demo) {
    trains = sample(from, to, date);
  } else {
    try {
      const q = new URLSearchParams({ from, to, date });
      if (cls) q.set("seat_class", cls);
      const res = await fetch("/api/search?" + q);
      rawText = await res.text();
      let body = {};
      try { body = JSON.parse(rawText); } catch (x) {}
      if (!res.ok) { lastDetail = body.detail || ""; throw new Error(body.error || "status " + res.status); }
      trains = normalize(body);
    } catch (err) { failed = err.message || "unknown error"; }
  }
  busy = false; if (!auto) btn.disabled = false;

  if (failed) {
    if (!auto) {
      all = []; $("#out").innerHTML = ""; $("#trip").hidden = true; $("#bookbox").hidden = true; $("#demoBanner").hidden = true;
      status.className = "err";
      status.textContent = "Could not load real trains for " + from + " to " + to + " (" + failed + "). Press \"More\" below to connect and see the server details.";
      $("#connectBtn").classList.add("pulse"); refreshWho();
      if (lastDetail) { $("#raw").textContent = "The railway server said:\n" + lastDetail; $("#rawBox").hidden = false; $("#rawBox").open = false; }
    }
    return;
  }
  if (!trains.length) {
    if (!auto) {
      all = []; $("#out").innerHTML = ""; $("#trip").hidden = true; $("#bookbox").hidden = true; $("#demoBanner").hidden = true;
      if (rawText && rawText.length > 2 && !demo) {
        status.className = "err";
        status.textContent = "The railway server answered, but no trains could be read from it. Its reply is shown below.";
        $("#raw").textContent = rawText.slice(0, 6000); $("#rawBox").hidden = false; $("#rawBox").open = false;
      } else {
        status.textContent = "No trains found from " + from + " to " + to + " on this date. Check the spelling or try another date.";
      }
    }
    return;
  }
  all = trains; ctx = { from, to, date, cls, demo, at: new Date() };
  if (rawText && !demo) { $("#raw").textContent = rawText.slice(0, 6000); $("#rawBox").hidden = false; $("#rawBox").open = false; }
  status.className = ""; status.textContent = "";
  $("#demoBanner").hidden = !demo;
  $("#rFrom").textContent = from; $("#rTo").textContent = to; $("#trip").hidden = false;
  if (!auto) { const l = $("#loco"); l.classList.remove("go"); void l.offsetWidth; l.classList.add("go"); }
  view(auto);
  if (demo) { $("#bookA").hidden = true; $("#bookbox").hidden = false; } else bookLink(from, to, date, cls);
}

const searchBase = search;
search = async function (auto) {
  await searchBase(auto);
  if (!auto && $("#trainNameInput").value.trim()) startDeep();
};
$("#f").addEventListener("submit", (e) => { e.preventDefault(); search(false); });
setInterval(() => {
  if (!$("#live").checked) return;
  if (deep.active) {
    if (!deep.running && deep.finishedAt && Date.now() - deep.finishedAt > 30000) runDeepSearch(deep.from, deep.to, deep.date, true);
  } else search(true);
}, 5000);

async function sendProfile(text) {
  const note = $("#reqNote");
  try {
    const r = await fetch("/api/profile", { method: "POST", headers: {"Content-Type": "application/json"}, body: JSON.stringify({ curl: text }) });
    const b = await r.json().catch(() => ({}));
    note.textContent = r.ok ? (b.host ? "Saved. Searches now use your copied request (" + b.host + "). Press Search trains." : "Forgotten. The built-in request is used again.")
                            : "Could not use it: " + (b.error || r.status);
  } catch (x) { note.textContent = "Could not reach server.py. Is it still running?"; }
}
function ago(sec) {
  const m = Math.round(Math.abs(sec) / 60);
  return m >= 120 ? Math.round(m / 60) + "h" : m + "m";
}
async function refreshWho() {
  try {
    const b = await (await fetch("/api/me")).json();
    let t = "Not logged in";
    if (b.loggedIn) {
      const tick = (ok) => ok ? "\u2713" : "\u2717 missing";
      const ex = b.expiresIn == null ? "" : b.expiresIn < 0 ? " \u00b7 token EXPIRED " + ago(b.expiresIn) + " ago" : " \u00b7 token valid " + ago(b.expiresIn);
      t = "Logged in" + (b.mobile ? " as " + b.mobile : "") + ex + " \u00b7 device ID " + tick(b.deviceId) + " \u00b7 device key " + tick(b.deviceKey);
    }
    $("#who").textContent = t;
    const av = $("#avatar");
    av.dataset.state = b.loggedIn ? (b.expiresIn != null && b.expiresIn < 0 ? "expired" : "live") : "out";
    av.title = b.loggedIn ? "Logged in" + (b.mobile ? " as " + b.mobile : "") : "Not logged in";
    $("#openLogin").hidden = !!b.loggedIn; $("#logoutBtn").hidden = !b.loggedIn;
  } catch (x) {}
}
async function doLogin(payload) {
  const note = $("#loginNote"); note.textContent = "Logging in...";
  try {
    const r = await fetch("/api/login", { method: "POST", headers: {"Content-Type": "application/json"}, body: JSON.stringify(payload) });
    const b = await r.json().catch(() => ({}));
    if (!r.ok) {
      note.textContent = b.error || ("status " + r.status);
      if (b.detail) { $("#raw").textContent = "The railway server said:\n" + b.detail; $("#rawBox").hidden = false; }
      return;
    }
    $("#pw").value = ""; $("#all").value = "";
    $("#loginDlg").close(); await refreshWho(); search(false);
  } catch (x) { note.textContent = "Could not reach server.py. Is it still running?"; }
}
let pollTimer = null, hintTimer = null;
const BM_CODE = "(function(){try{var o={},re=/token|uudid|ssdk|device|auth|user/i;for(var i=0;i<localStorage.length;i++){var k=localStorage.key(i);if(re.test(k))o[k]=localStorage.getItem(k)}if(!Object.keys(o).length){alert('Log in on the railway site first, then click this bookmark again.');return}location.href='" + location.origin + "/#grab='+encodeURIComponent(JSON.stringify(o))}catch(e){alert(e)}})()";
$("#bm").href = "javascript:" + BM_CODE;
$("#bm").onclick = (e) => { e.preventDefault(); alert("Drag this button onto your bookmarks bar. Do not click it here."); };
$("#copyBm").onclick = async () => {
  try { await navigator.clipboard.writeText("javascript:" + BM_CODE); $("#loginNote").textContent = "Copied. Make a new bookmark, name it Send to my site and paste this as its address."; }
  catch (x) { $("#loginNote").textContent = "Could not copy. Drag the yellow button instead."; }
};
async function startAutoLogin() {
  window.open("https://eticket.railway.gov.bd/login?brsend=1", "_blank");
  let startAt = 0;
  try { startAt = (await (await fetch("/api/me")).json()).at || 0; } catch (x) {}
  $("#loginNote").textContent = "";
  $("#autoInstall").hidden = true;
  clearInterval(pollTimer);
  clearTimeout(hintTimer);
  hintTimer = setTimeout(() => { $("#autoInstall").hidden = false; }, 45000);
  const end = Date.now() + 15 * 60 * 1000;
  pollTimer = setInterval(async () => {
    if (Date.now() > end) { clearInterval(pollTimer); return; }
    try {
      const b = await (await fetch("/api/me")).json();
      if (b.at > startAt) { clearInterval(pollTimer); clearTimeout(hintTimer); $("#loginDlg").close(); await refreshWho(); search(false); }
    } catch (x) {}
  }, 2000);
}
$("#openRail").onclick = startAutoLogin;
$("#retryRail").onclick = startAutoLogin;
$("#openLogin").onclick = () => { $("#loginNote").textContent = ""; $("#loginDlg").showModal(); startAutoLogin(); };
$("#cancelLogin").onclick = () => { clearInterval(pollTimer); clearTimeout(hintTimer); $("#loginDlg").close(); };
$("#doLogin").onclick = () => doLogin({ mobile: $("#mob").value.trim(), password: $("#pw").value });
$("#doToken").onclick = () => {
  const t = $("#all").value, pick = (k) => { const m = t.match(new RegExp(k + "\\s*[=:]\\s*\"?([^\\s\"]+)", "i")); return m ? m[1] : ""; };
  const token = pick("token");
  if (!token) { $("#loginNote").textContent = "I could not find token=... in what you pasted."; return; }
  doLogin({ token, device_id: pick("uudid"), device_key: pick("ssdk") });
};
$("#pw").addEventListener("keydown", (e) => { if (e.key === "Enter") $("#doLogin").click(); });
$("#logoutBtn").onclick = async () => { await fetch("/api/logout", { method: "POST" }); refreshWho(); };
refreshWho();

(async () => {
  const m = location.hash.match(/^#grab=(.+)$/);
  if (!m) return;
  history.replaceState(null, "", location.pathname + location.search);
  let st = null; try { st = JSON.parse(decodeURIComponent(m[1])); } catch (x) {}
  const status = $("#status");
  if (!st) { status.className = "err"; status.textContent = "The login data from the bookmark could not be read."; return; }
  status.className = ""; status.textContent = "Saving your railway login...";
  try {
    const r = await fetch("/api/login", { method: "POST", headers: {"Content-Type": "application/json"}, body: JSON.stringify({ storage: st }) });
    const b = await r.json().catch(() => ({}));
    if (!r.ok) { status.className = "err"; status.textContent = b.error || ("status " + r.status); return; }
    status.textContent = ""; await refreshWho(); search(false);
  } catch (x) { status.className = "err"; status.textContent = "Could not reach server.py. Is it still running?"; }
})();

$("#connectBtn").onclick = () => {
  const box = $("#connect"), show = box.hidden;
  box.hidden = !show;
  $("#connectBtn").textContent = show ? "Less" : "More";
  $("#connectBtn").setAttribute("aria-expanded", String(show));
  $("#connectBtn").classList.remove("pulse");
  if (show) box.scrollIntoView({ behavior: "smooth", block: "nearest" });
};
$("#curlDetailsBtn").onclick = () => {
  const box = $("#curlSteps"), show = box.hidden;
  box.hidden = !show;
  $("#curlDetailsBtn").textContent = show ? "Hide details" : "Details";
  $("#curlDetailsBtn").setAttribute("aria-expanded", String(show));
};
$("#saveReq").onclick = () => { if ($("#curl").value.trim()) sendProfile($("#curl").value); };
$("#clearReq").onclick = () => { $("#curl").value = ""; sendProfile(""); };

const brHook = /*__HOOK__*/;
const ADVANCE_DAYS = 10;
$("#bm2").href = "javascript:" + encodeURIComponent("(" + brHook.toString() + ")(" + JSON.stringify({mode: "page", server: location.origin, close: false, auto: false}) + ");");
$("#bm2").onclick = (e) => { e.preventDefault(); alert("Drag this button onto your bookmarks bar. Do not click it here."); };
const AUTO_LABELS = [
  "Open the railway site in a new tab",
  "Wait for the search request from the railway tab",
  "Request captured",
  "Fill the cURL box",
  "Search trains"
];
function showSteps(cur, opt) {
  opt = opt || {};
  const ol = $("#autoSteps"); ol.hidden = false;
  ol.innerHTML = AUTO_LABELS.map((t, i) => {
    const cls = i < cur ? "done" : i === cur ? (opt.fail ? "fail" : "now") : "todo";
    const extra = (i === cur && opt.msg) ? "<span>" + esc(opt.msg) + "</span>" : "";
    return '<li class="' + cls + '">' + esc(t) + extra + "</li>";
  }).join("");
}
const fmtLeft = (ms) => { const s = Math.max(0, Math.ceil(ms / 1000)); return Math.floor(s / 60) + ":" + String(s % 60).padStart(2, "0"); };

async function runAutoConnect() {
  const note = $("#reqNote"), d = new Date();
  d.setDate(d.getDate() + ADVANCE_DAYS);
  const y = d.getFullYear(), m = d.getMonth(), day = String(d.getDate()).padStart(2, "0");
  const when = day + "-" + MON[m] + "-" + y;
  const url = "https://eticket.railway.gov.bd/booking/train/search?fromcity=Dhaka&tocity=Khulna&doj=" + when + "&class=S_CHAIR&autocap=1";
  clearInterval(pollTimer);
  note.textContent = "";
  
  showSteps(0, {msg: "Dhaka to Khulna, " + when + ", S_CHAIR"});
  let start = 0;
  try { start = (await (await fetch("/api/me")).json()).profileAt || 0; }
  catch (x) { showSteps(0, {fail: true, msg: "Cannot reach this program. Is server running in terminal?"}); return; }
  const w = window.open(url, "_blank");
  if (!w) { showSteps(0, {fail: true, msg: "The browser blocked the new tab. Allow pop-ups for this page, then press Connect again."}); return; }
  const t0 = Date.now(), end = t0 + 5 * 60 * 1000;
  const waiting = () => {
    const spent = Date.now() - t0;
    let msg = "Log in on the railway tab if it asks. Time left " + fmtLeft(end - Date.now()) + ".";
    if (spent > 30000) msg += " Still nothing: is the Tampermonkey script on, or did you click the Capture search bookmark on the railway tab?";
    showSteps(1, {msg});
  };
  waiting();
  pollTimer = setInterval(async () => {
    if (Date.now() > end) {
      clearInterval(pollTimer);
      showSteps(1, {fail: true, msg: "Nothing was captured in 5 minutes."});
      return;
    }
    try {
      const b = await (await fetch("/api/me")).json();
      if (b.profileAt > start) {
        clearInterval(pollTimer);
        showSteps(2, {msg: "The railway tab sent the search request to this program."});
        showSteps(3);
        try { $("#curl").value = (await (await fetch("/api/profile-curl")).json()).curl || ""; } catch (x) {}
        showSteps(4, {msg: "Asking the railway server for trains..."});
        await search(false);
        if ($("#status").classList.contains("err")) {
          showSteps(4, {fail: true, msg: $("#status").textContent});
        } else {
          showSteps(AUTO_LABELS.length);
          note.textContent = all.length ? "Done: " + all.length + " trains found for Dhaka to Khulna on " + when + "." : "Done, but no trains were listed for " + when + ".";
        }
        return;
      }
      waiting();
    } catch (x) {
      clearInterval(pollTimer);
      showSteps(1, {fail: true, msg: "Lost contact with this program."});
    }
  }, 1500);
}

$("#topConnectBtn").onclick = runAutoConnect;
</script>
</body>
</html>
"""


class Handler(BaseHTTPRequestHandler):
    def reply(self, status, body, ctype="application/json; charset=utf-8"):
        if isinstance(body, str):
            body = body.encode("utf-8")
        self.send_response(status)
        self.send_header("Content-Type", ctype)
        self.send_header("Content-Length", str(len(body)))
        if self.headers.get("Origin") == ALLOWED_ORIGIN:
            self.send_header("Access-Control-Allow-Origin", ALLOWED_ORIGIN)
            self.send_header("Access-Control-Allow-Headers", "Content-Type")
            self.send_header("Access-Control-Allow-Methods", "POST, OPTIONS")
            self.send_header("Access-Control-Allow-Private-Network", "true")
            self.send_header("Vary", "Origin")
        self.end_headers()
        self.wfile.write(body)

    def do_OPTIONS(self):
        self.reply(204, b"")

    def do_GET(self):
        u = urllib.parse.urlparse(self.path)
        if u.path in ("/", "/index.html"):
            return self.reply(200, PAGE.replace("/*__HOOK__*/", HOOK_JS), "text/html; charset=utf-8")
        if u.path == "/api/search":
            return self.search(urllib.parse.parse_qs(u.query))
        if u.path == "/br-capture.user.js":
            return self.reply(200, userscript(self.server.server_address[1]), "text/javascript; charset=utf-8")
        if u.path == "/br-login.user.js":
            return self.reply(200, login_userscript(self.server.server_address[1]), "text/javascript; charset=utf-8")
        if u.path == "/api/capture":
            try:
                import base64
                set_captured(json.loads(base64.b64decode(urllib.parse.parse_qs(u.query).get("d", [""])[0])))
            except Exception as e:
                print("Capture refused:", e)
            return self.reply(204, b"")
        if u.path == "/api/profile-curl":
            return self.reply(200, json.dumps({"curl": profile_to_curl(PROFILE)}))
        if u.path == "/api/me":
            return self.reply(200, json.dumps({
                "loggedIn": bool(SESSION["token"]), "mobile": SESSION["mobile"],
                "deviceId": bool(SESSION["device_id"]), "deviceKey": bool(SESSION["device_key"]), "at": SESSION["at"],
                "expiresIn": token_expiry(SESSION["token"]) if SESSION["token"] else None,
                "profileAt": PROFILE_AT[0], "hasProfile": bool(PROFILE)}))
        if u.path == "/favicon.ico":
            return self.reply(204, b"")
        self.reply(404, json.dumps({"error": "not found"}))

    def do_POST(self):
        global PROFILE
        path = urllib.parse.urlparse(self.path).path
        if path not in ("/api/profile", "/api/login", "/api/logout", "/api/capture"):
            return self.reply(404, json.dumps({"error": "not found"}))
        try:
            n = int(self.headers.get("Content-Length") or 0)
            data = json.loads(self.rfile.read(n) or b"{}")
            cache.clear()
            if path == "/api/capture":
                set_captured(data)
                return self.reply(200, json.dumps({"ok": True}))
            if path == "/api/logout":
                for k in ("token", "mobile"):
                    SESSION[k] = ""
                CREDS.update(mobile="", password="")
                return self.reply(200, json.dumps({"ok": True}))
            if path == "/api/login":
                return self.login(data)
            text = data.get("curl", "")
            if not text.strip():
                PROFILE = None
                return self.reply(200, json.dumps({"ok": True, "host": ""}))
            PROFILE = parse_curl(text)
            host = urllib.parse.urlsplit(PROFILE["url"]).netloc
            print("Using copied request for", host, "with headers:", ", ".join(PROFILE["headers"]))
            self.reply(200, json.dumps({"ok": True, "host": host}))
        except ValueError as e:
            self.reply(400, json.dumps({"error": str(e)}))

    def login(self, data):
        global PROFILE
        mobile = str(data.get("mobile") or "").strip()
        token = str(data.get("token") or "").strip()
        dev_id = str(data.get("device_id") or "").strip().strip('"')
        dev_key = str(data.get("device_key") or "").strip().strip('"')
        if isinstance(data.get("storage"), dict):
            got = pick_from_storage(data["storage"])
            token, dev_id, dev_key = got["token"] or token, got["device_id"] or dev_id, got["device_key"] or dev_key
            if not token:
                return self.reply(400, json.dumps({"error": "no login token was found on the railway tab"}))
            mobile = mobile or token_phone(token)
        if dev_id:
            SESSION["device_id"] = dev_id
        if dev_key:
            SESSION["device_key"] = dev_key
        PROFILE = None
        if token:
            SESSION.update(token=re.sub(r"^bearer\s+", "", token, flags=re.I).strip('"'), mobile=mobile, at=time.time())
            CREDS.update(mobile="", password="")
            return self.reply(200, json.dumps({"ok": True}))
        password = str(data.get("password") or "")
        if not (mobile and password):
            return self.reply(400, json.dumps({"error": "enter your mobile number and password"}))
        ok, err, detail = full_login(mobile, password)
        if not ok:
            return self.reply(502, json.dumps({"error": err, "detail": detail}))
        SESSION["at"] = time.time()
        self.reply(200, json.dumps({"ok": True}))

    def search(self, p):
        get = lambda k: (p.get(k) or [""])[0].strip()
        frm, to, date, cls = get("from"), get("to"), get("date"), get("seat_class")
        if not (frm and to and re.fullmatch(r"\d{4}-\d{2}-\d{2}", date)):
            return self.reply(400, json.dumps({"error": "need from, to and date (YYYY-MM-DD)"}))
        url, headers, method, data = build_request(frm, to, date, cls)
        key = (method, url, data)

        hit = cache.get(key)
        if hit and time.time() - hit[0] < TTL:
            return self.reply(hit[1], hit[2])
        err = None
        for attempt in (0, 1):
            if attempt:
                url, headers, method, data = build_request(frm, to, date, cls)
            try:
                req = urllib.request.Request(url, data=data, headers=headers, method=method)
                with urllib.request.urlopen(req, timeout=20) as r:
                    status, body = r.status, r.read()
                break
            except urllib.error.HTTPError as e:
                if e.code in (401, 403) and attempt == 0 and CREDS["password"] and not PROFILE:
                    cache.clear()
                    if full_login(CREDS["mobile"], CREDS["password"], auto=True)[0]:
                        continue
                err = e
                break
            except Exception as e:
                return self.reply(502, json.dumps({"error": "cannot reach railway server (%s)" % e}))
        if err is not None:
            e = err
            detail = e.read()[:800].decode("utf-8", "replace")
            hint = ""
            return self.reply(502, json.dumps({"error": "railway server answered %s%s" % (e.code, hint), "detail": detail}))
        cache[key] = (time.time(), status, body)
        self.reply(status, body)

    def log_message(self, fmt, *args):
        print(self.address_string(), fmt % args)


def main():
    saved = {} if CLEAR_LOGIN_ON_START else load_device()
    tok = saved.pop("token", "")
    SESSION.update(saved)
    if tok and (token_expiry(tok) or 0) > 60:
        SESSION["token"] = tok
        print("Using the saved token (valid for another %d minutes)." % (token_expiry(tok) // 60))
    base = int(os.environ.get("PORT", 3000))
    for port in range(base, base + 10):
        try:
            server = ThreadingHTTPServer(("127.0.0.1", port), Handler)
            break
        except OSError:
            print("Port %d is busy, trying the next one..." % port)
    else:
        raise SystemExit("No free port found between %d and %d." % (base, base + 9))
    url = "http://localhost:%d" % port
    print("Running at", url, "  (press Ctrl+C to stop)")
    threading.Timer(1.0, lambda: webbrowser.open(url)).start()
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        print("\nStopped.")


if __name__ == "__main__":
    main()
