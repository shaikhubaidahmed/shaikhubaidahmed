#!/usr/bin/env python3
"""Render the profile README as a stack of animated SVG cards.

    python scripts/render_cards.py            # animated cards -> cards/
    STATIC=1 python scripts/render_cards.py   # final frame only -> cards-static/ (+ preview.html)

Reads data/github.json (written daily by .github/workflows/profile-cards.yml).
Cards are shown through <img>, so everything is self-contained: an inline
<style> with CSS keyframes, SMIL <animate> elements, and fonts subset with
pyftsubset and embedded as base64 WOFF data URIs. Every animation is switched
off under prefers-reduced-motion.

Look: warm paper cards, near-black ink, electric blue / red / yellow accents,
Instrument Serif headlines (second word in italic), Geist body, Geist Mono labels.

Requires: fonttools, brotli  (pip install fonttools brotli)
"""
from __future__ import annotations

import base64
import datetime as dt
import json
import math
import os
import re
import shutil
import subprocess
import sys
import tempfile
from collections import defaultdict
from html import escape
from pathlib import Path

from fontTools.ttLib import TTFont

ROOT = Path(__file__).resolve().parent.parent
STATIC = os.environ.get("STATIC") == "1"
DATA = Path(os.environ.get("DATA", ROOT / "data" / "github.json"))
OUT = Path(os.environ.get("OUT", ROOT / ("cards-static" if STATIC else "cards")))
README = ROOT / "README.md"

# ─── Profile config: edit these ──────────────────────────────────────────────
PROFILE = {
    "first": "Shaikh Ubaid",          # set in roman
    "last": "Ahmed",                  # set in italic, in blue
    "eyebrow": "ML Engineer · Researcher · MLOps",
    "lines": ["I build ML systems and turn models", "into reliable software."],
    "meta": ["IIT Indore", "MS (Research) CSE", "Indore, IN"],
    "portrait": True,  # uses assets/portrait.{png,jpg} when present
    "url": "https://github.com/shaikhubaidahmed",
}

# Exactly three tiles, rendered side by side at 32.5% width each, in blue,
# red and yellow. `title` is (roman, italic).
TILES = [
    {"key": "website", "label": "Website", "title": ("DSA", "Hub"),
     "desc": "My DSA practice workspace", "sub": "github.io/DSAHub",
     "url": "https://shaikhubaidahmed.github.io/DSAHub/"},
    {"key": "linkedin", "label": "LinkedIn", "title": ("Let's", "connect"),
     "desc": "Experience and research", "sub": "in/shaikhubaidahmed",
     "url": "https://www.linkedin.com/in/shaikhubaidahmed"},
    {"key": "leetcode", "label": "LeetCode", "title": ("Practice", "log"),
     "desc": "Problem-solving, daily", "sub": "leetcode.com/shaikhubaidahmed",
     "url": "https://leetcode.com/shaikhubaidahmed"},
]

# Repos to feature, in order. Empty = your pinned repos (forks skipped).
FEATURED: list[str] = []
MAX_FEATURED = 4
# Optional description overrides for repos whose GitHub description is empty.
DESCRIPTIONS: dict[str, str] = {}

# Palette. Cards are paper in both light and dark GitHub themes.
PAL = dict(
    paper="#F3F3F1", deep="#E2E2DE", ink="#111111", soft="#3A3A3C", muted="#6E6E73",
    rule="#DCDCD7", blue="#2F55FF", red="#EF3B2D", yellow="#FFC81A", partial="#C9C9C4",
)
TILE_COLOURS = [("blue", "#FFFFFF"), ("red", "#FFFFFF"), ("yellow", "#111111")]
HEAT = [PAL["deep"], "#C8D3FF", "#94A8FF", "#6280FF", PAL["blue"]]

FONT_DIR = ROOT / "fonts"
FONTS = {  # key: (file, family alias, weight, style)
    "serif": ("instrument-serif-latin-400-normal.woff2", "D", 400, "normal"),
    "serifI": ("instrument-serif-latin-400-italic.woff2", "D", 400, "italic"),
    "sans400": ("geist-sans-latin-400-normal.woff2", "S", 400, "normal"),
    "sans500": ("geist-sans-latin-500-normal.woff2", "S", 500, "normal"),
    "sans600": ("geist-sans-latin-600-normal.woff2", "S", 600, "normal"),
    "mono400": ("geist-mono-latin-400-normal.woff2", "M", 400, "normal"),
    "mono500": ("geist-mono-latin-500-normal.woff2", "M", 500, "normal"),
}
STACK = {
    "D": "'D','Instrument Serif',Georgia,serif",
    "S": "'S',Geist,system-ui,-apple-system,sans-serif",
    "M": "'M','Geist Mono',ui-monospace,SFMono-Regular,Menlo,monospace",
}

# Card start times (seconds) so the page plays top to bottom.
STEP = 0.3

W = 840  # full-width card width; GitHub's README column is ~830px


# ─── Fonts: measuring and subsetting ─────────────────────────────────────────
class Metrics:
    _cache: dict[str, tuple] = {}

    @classmethod
    def get(cls, key):
        if key not in cls._cache:
            f = TTFont(FONT_DIR / FONTS[key][0])
            cls._cache[key] = (f.getBestCmap(), f["hmtx"], f["head"].unitsPerEm)
        return cls._cache[key]


def text_width(s: str, font: str, size: float, ls: float = 0.0) -> float:
    cmap, hmtx, upm = Metrics.get(font)
    units = sum(hmtx[cmap.get(ord(ch), ".notdef")][0] for ch in s)
    return units / upm * size + ls * size * len(s)


def truncate(s, font, size, maxw, ls=0.0):
    if text_width(s, font, size, ls) <= maxw:
        return s
    while s and text_width(s + "…", font, size, ls) > maxw:
        s = s[:-1]
    return s.rstrip() + "…"


def wrap(s, font, size, maxw, lines=2):
    words, out, cur = s.split(), [], ""
    for i, w in enumerate(words):
        trial = f"{cur} {w}".strip()
        if text_width(trial, font, size) <= maxw:
            cur = trial
            continue
        if len(out) == lines - 1:
            return out + [truncate(" ".join([cur] + words[i:]), font, size, maxw)]
        out.append(cur)
        cur = w
    return out + [cur] if cur else out


_subset_cache: dict[tuple, str] = {}


def font_face(key: str, chars: set[str]) -> str:
    text = "".join(sorted(chars | {" "}))
    if (key, text) not in _subset_cache:
        tool = shutil.which("pyftsubset")
        cmd = [tool] if tool else [sys.executable, "-m", "fontTools.subset"]
        with tempfile.TemporaryDirectory() as tmp:
            tf, of = Path(tmp, "t.txt"), Path(tmp, "o.woff")
            tf.write_text(text, encoding="utf-8")
            subprocess.run(cmd + [str(FONT_DIR / FONTS[key][0]), f"--text-file={tf}",
                                  f"--output-file={of}", "--flavor=woff", "--no-hinting",
                                  "--layout-features+=tnum", "--name-IDs=", "--desubroutinize"],
                           check=True)
            _subset_cache[(key, text)] = base64.b64encode(of.read_bytes()).decode()
    _, fam, wt, style = FONTS[key]
    return (f'@font-face{{font-family:"{fam}";font-weight:{wt};font-style:{style};'
            f'src:url(data:font/woff;base64,{_subset_cache[(key, text)]}) format("woff")}}')


# ─── Card builder ────────────────────────────────────────────────────────────
BASE_CSS = """
@keyframes rise{from{opacity:0;transform:translateY(12px)}}
@keyframes fade{from{opacity:0}}
@keyframes grow{from{transform:scaleY(0)}}
@keyframes pop{from{opacity:0;transform:scale(.3)}}
@keyframes tile{from{opacity:0;transform:translateY(16px) scale(.97)}}
@keyframes draw{from{stroke-dashoffset:1}}
.a-rise{animation:rise .8s cubic-bezier(.2,.7,.2,1) both}
.a-fade{animation:fade .8s ease-out both}
.a-grow{transform-box:fill-box;transform-origin:50% 100%;animation:grow .9s cubic-bezier(.2,.7,.2,1) both}
.a-pop{transform-box:fill-box;transform-origin:center;animation:pop .45s cubic-bezier(.2,.7,.2,1) both}
.a-tile{transform-box:fill-box;transform-origin:50% 100%;animation:tile .8s cubic-bezier(.2,.7,.2,1) both}
.a-draw{stroke-dasharray:1;animation:draw 1.2s cubic-bezier(.6,0,.2,1) both}
.motion{display:none}
@media (prefers-reduced-motion:no-preference){.motion{display:inline}}
@media (prefers-reduced-motion:reduce){*{animation:none!important}}
"""


def col(c):
    """Palette name or raw hex -> hex."""
    return PAL.get(c, c)


class Card:
    def __init__(self, name, w, h, start, title, bg="paper"):
        self.name, self.w, self.h, self.start, self.title, self.bg = name, w, h, start, title, bg
        self.parts, self.motion, self.defs = [], [], []
        self.used = defaultdict(set)

    def add(self, s):
        self.parts.append(s)

    def anim(self, kind, at=0.0):
        """class/style attributes for a CSS animation starting `at` s into this card."""
        return f'class="a-{kind}" style="animation-delay:{self.start + at:.2f}s"'

    def t(self, s) -> float:
        """Absolute SMIL begin time for an offset into this card."""
        return round(self.start + s, 2)

    def text(self, x, y, s, font="sans400", size=14, fill="muted", anchor="start",
             ls=0.0, anim=None, at=0.0, opacity=None):
        self.used[font].update(s)
        _, fam, wt, style = FONTS[font]
        css = f"font-family:{STACK[fam]};font-weight:{wt};font-style:{style}"
        if ls:
            css += f";letter-spacing:{ls}em"
        cls = ""
        if anim:
            cls = f' class="a-{anim}"'
            css += f";animation-delay:{self.start + at:.2f}s"
        op = f' opacity="{opacity}"' if opacity is not None else ""
        self.add(f'<text x="{x:.1f}" y="{y:.1f}" font-size="{size}" text-anchor="{anchor}" '
                 f'fill="{col(fill)}"{op}{cls} style="{css}">{escape(s, quote=False)}</text>')

    def runs(self, x, y, parts, size, anchor="start", ls=0.0):
        """Draw consecutive (text, font, fill) runs on one baseline; returns end x."""
        total = sum(text_width(s, f, size, ls) for s, f, _ in parts)
        x = x - total if anchor == "end" else x
        for s, f, fill in parts:
            self.text(x, y, s, f, size, fill, ls=ls)
            x += text_width(s, f, size, ls)
        return x

    def frame(self):
        self.add(f'<rect width="{self.w}" height="{self.h}" rx="14" fill="{col(self.bg)}"/>')

    def render(self) -> str:
        faces = "".join(font_face(k, v) for k, v in sorted(self.used.items()))
        css = BASE_CSS + faces
        if STATIC:
            css += "*{animation:none!important}"
        motion = "" if STATIC or not self.motion else f'<g class="motion">{"".join(self.motion)}</g>'
        defs = f"<defs>{''.join(self.defs)}</defs>" if self.defs else ""
        return (f'<svg xmlns="http://www.w3.org/2000/svg" width="{self.w}" height="{self.h}" '
                f'viewBox="0 0 {self.w} {self.h}" role="img" aria-labelledby="t">'
                f'<title id="t">{escape(self.title)}</title><style><![CDATA[{css}]]></style>{defs}'
                f'{"".join(self.parts)}{motion}</svg>')

    def save(self):
        OUT.mkdir(parents=True, exist_ok=True)
        (OUT / f"{self.name}.svg").write_text(self.render(), encoding="utf-8")
        return self.name


# ─── Data ────────────────────────────────────────────────────────────────────
def load():
    raw = json.loads(DATA.read_text(encoding="utf-8"))
    user = (raw.get("data") or raw)["user"]
    cal = user["contributionsCollection"]["contributionCalendar"]
    days = [(dt.date.fromisoformat(d["date"]), d["contributionCount"])
            for w in cal["weeks"] for d in w["contributionDays"]]
    days.sort()
    repos = {}
    for src in ("repositories", "pinnedItems"):
        for r in (user.get(src) or {}).get("nodes") or []:
            if r and r.get("name"):
                repos[r["name"]] = r
    pinned = [r["name"] for r in (user.get("pinnedItems") or {}).get("nodes") or [] if r]
    return user, cal["totalContributions"], days, repos, pinned


def fmt_day(d):
    return f"{d:%b} {d.day}"


def stats(days):
    today = days[-1][0]
    # current streak: today counts if active, otherwise start from yesterday
    i = len(days) - 1
    if days[i][1] == 0:
        i -= 1
    cur, cur_start = 0, None
    while i >= 0 and days[i][1] > 0:
        cur, cur_start, i = cur + 1, days[i][0], i - 1
    best, best_rng, run, run_start = 0, None, 0, None
    for d, c in days:
        if c:
            run, run_start = run + 1, run_start or d
            if run > best:
                best, best_rng = run, (run_start, d)
        else:
            run, run_start = 0, None
    active = sum(1 for _, c in days if c)
    busiest = max(days, key=lambda x: (x[1], x[0]))
    months = []
    y, m = today.year, today.month
    for _ in range(12):
        months.append((y, m))
        y, m = (y, m - 1) if m > 1 else (y - 1, 12)
    months.reverse()
    per = defaultdict(int)
    for d, c in days:
        per[(d.year, d.month)] += c
    return dict(today=today, cur=cur, cur_start=cur_start, best=best, best_rng=best_rng,
                active=active, span=len(days), busiest=busiest,
                months=[(dt.date(y, m, 1), per[(y, m)]) for y, m in months])


# ─── Shared bits ─────────────────────────────────────────────────────────────
def eyebrow(c, x, y, s, fill="muted", anchor="start", at=0.0):
    c.text(x, y, s.upper(), "mono400", 12, fill, anchor, ls=0.09, anim="fade", at=at)


def bar_path(x, y, w, h, r=4):
    r = min(r, h, w / 2)
    return (f"M{x:.1f},{y + h:.1f}V{y + r:.1f}Q{x:.1f},{y:.1f} {x + r:.1f},{y:.1f}"
            f"H{x + w - r:.1f}Q{x + w:.1f},{y:.1f} {x + w:.1f},{y + r:.1f}V{y + h:.1f}Z")


def arrow(x, y, colour, size=9):
    """A → arrow (the fonts' latin subsets have no arrow glyph)."""
    return (f'<path d="M{x - size},{y}H{x}M{x - 4},{y - 4}L{x},{y}L{x - 4},{y + 4}" fill="none" '
            f'stroke="{col(colour)}" stroke-width="1.6" stroke-linecap="round" stroke-linejoin="round"/>')


def pulse(c, cx, cy, r0, r1, at, colour="blue", dur=2.4):
    """SMIL pulse ring; lives in the .motion group so reduced motion hides it."""
    c.motion.append(
        f'<circle cx="{cx:.1f}" cy="{cy:.1f}" r="{r0}" fill="none" stroke="{col(colour)}" stroke-width="1.5" opacity="0">'
        f'<animate attributeName="r" values="{r0};{r1}" dur="{dur}s" begin="{c.t(at)}s" repeatCount="indefinite"/>'
        f'<animate attributeName="opacity" values=".9;0" dur="{dur}s" begin="{c.t(at)}s" repeatCount="indefinite"/>'
        f"</circle>")


def portrait_data():
    if not PROFILE["portrait"]:
        return None
    env = os.environ.get("PORTRAIT")
    for p in ([Path(env)] if env else []) + [ROOT / "assets" / f"portrait.{e}" for e in ("png", "jpg", "jpeg", "webp")]:
        if p.is_file():
            b = p.read_bytes()
            mime = ("image/png" if b[:4] == b"\x89PNG" else "image/jpeg" if b[:2] == b"\xff\xd8"
                    else "image/webp" if b[8:12] == b"WEBP" else None)
            if mime:
                return f"data:{mime};base64,{base64.b64encode(b).decode()}"
    return None


# ─── Cards ───────────────────────────────────────────────────────────────────
def hero(start):
    p = PROFILE
    name = f"{p['first']} {p['last']}"
    c = Card("hero", W, 380, start, f"{name}. {p['eyebrow']}. {' '.join(p['lines'])}")
    c.frame()
    R = 96
    cx, cy = W - 44 - R, 176
    left, maxw = 44, cx - R - 44 - 36

    # eyebrow with a blinking "live" square (SMIL)
    c.add(f'<g {c.anim("fade")}><rect x="{left + 4}" y="47" width="8" height="8" rx="2" fill="{PAL["blue"]}">')
    if not STATIC:
        c.add(f'<animate attributeName="opacity" values="1;.25;1" dur="2s" begin="{c.t(1.2)}s" repeatCount="indefinite"/>')
    c.add("</rect></g>")
    c.text(left + 22, 55, p["eyebrow"].upper(), "mono400", 12, "muted", ls=0.09, anim="fade")

    size = 100
    while max(text_width(p["first"], "serif", size, -0.02),
              text_width(p["last"], "serifI", size, -0.02)) > maxw and size > 56:
        size -= 2
    c.add(f'<g {c.anim("rise", 0.15)}>')
    c.text(left, 150, p["first"], "serif", size, "ink", ls=-0.02)
    c.add("</g>")
    c.add(f'<g {c.anim("rise", 0.3)}>')
    c.text(left, 150 + size * 0.86, p["last"], "serifI", size, "blue", ls=-0.02)
    c.add("</g>")

    y = 150 + size * 0.86 + 48
    lines = p["lines"][:2]
    c.add(f'<g {c.anim("rise", 0.55)}>')
    for k, ln in enumerate(lines):
        c.text(left + 4, y + k * 26, truncate(ln, "sans400", 18, maxw), "sans400", 18, "soft")
    c.add("</g>")

    # meta row
    x, my = left + 4, y + 26 * (len(lines) - 1) + 40
    c.h = max(c.h, int(my + 36))
    c.parts[0] = f'<rect width="{c.w}" height="{c.h}" rx="14" fill="{PAL["paper"]}"/>'  # resize frame
    c.add(f'<g {c.anim("fade", 0.8)}>')
    for i, item in enumerate(p["meta"]):
        if i:
            c.text(x, my, "·", "mono400", 13, "muted")
            x += 20
        c.text(x, my, item, "mono400", 13, "ink")
        x += text_width(item, "mono400", 13) + 12
    c.add("</g>")

    # portrait (or monogram); a dashed blue ring rotates around it via SMIL
    c.add(f'<g {c.anim("pop", 0.25)}>')
    c.add(f'<circle cx="{cx}" cy="{cy}" r="{R + 12}" fill="none" stroke="{PAL["rule"]}" stroke-width="1"/>')
    img = portrait_data()
    if img:
        c.defs.append(f'<clipPath id="pc"><circle cx="{cx}" cy="{cy}" r="{R}"/></clipPath>')
        c.add(f'<circle cx="{cx}" cy="{cy}" r="{R}" fill="{PAL["deep"]}"/>')
        c.add(f'<image href="{img}" x="{cx - R}" y="{cy - R}" width="{2 * R}" height="{2 * R}" '
              f'preserveAspectRatio="xMidYMid slice" clip-path="url(#pc)"/>')
    else:
        c.add(f'<circle cx="{cx}" cy="{cy}" r="{R}" fill="{PAL["deep"]}"/>')
        c.runs(cx - (text_width(p["first"][0], "serif", 64) + text_width(p["last"][0], "serifI", 64)) / 2,
               cy + 22, [(p["first"][0], "serif", "ink"), (p["last"][0], "serifI", "blue")], 64)
    c.add("</g>")
    c.motion.append(
        f'<circle cx="{cx}" cy="{cy}" r="{R + 12}" fill="none" stroke="{PAL["blue"]}" stroke-width="2" '
        f'stroke-dasharray="3 10" stroke-linecap="round" opacity="0">'
        f'<animate attributeName="opacity" from="0" to="1" dur="0.8s" begin="{c.t(0.7)}s" fill="freeze"/>'
        f'<animateTransform attributeName="transform" type="rotate" from="0 {cx} {cy}" to="360 {cx} {cy}" '
        f'dur="40s" begin="{c.t(0.7)}s" repeatCount="indefinite"/></circle>')
    return c.save()


TILE_W, TILE_H = 300, 190


def tile(t, i, start):
    bg, on = TILE_COLOURS[i % 3]
    c = Card(f"tile-{t['key']}", TILE_W, TILE_H, start, f"{t['label']}: {t['url']}", bg=bg)
    pad = 24
    c.defs.append(f'<clipPath id="tc"><rect width="{TILE_W}" height="{TILE_H}" rx="14"/></clipPath>')
    c.add(f'<g {c.anim("tile")}><g clip-path="url(#tc)">')
    c.frame()
    # outline ring in the corner draws itself in
    c.add(f'<circle cx="{TILE_W - 14}" cy="{TILE_H - 6}" r="58" fill="none" stroke="{on}" '
          f'stroke-opacity=".45" stroke-width="2" pathLength="1" class="a-draw" '
          f'style="animation-delay:{c.start + .5:.2f}s"/>')
    c.add("</g>")
    c.text(pad, 40, t["label"].upper(), "mono400", 12, on, ls=0.1)
    c.add(f'<g {c.anim("fade", 0.5)}>{arrow(TILE_W - pad, 36, on)}</g>')
    roman, italic = t["title"]
    sep = "" if roman.isupper() and italic[:1].isupper() else " "  # "DSAHub" stays one word
    c.add(f'<g {c.anim("rise", 0.35)}>')
    c.runs(pad, 98, [(roman + sep, "serif", on), (italic, "serifI", on)], 44, ls=-0.01)
    c.add("</g>")
    c.add(f'<g {c.anim("rise", 0.55)}>')
    c.text(pad, 132, truncate(t["desc"], "sans400", 14, TILE_W - 2 * pad), "sans400", 14, on, opacity=.92)
    c.text(pad, 162, truncate(t["sub"], "mono400", 11.5, TILE_W - 2 * pad - 50), "mono400", 11.5, on, opacity=.8)
    c.add("</g></g>")
    return c.save()


def activity(st, total, start):
    c = Card("activity", W, 348, start, f"{total:,} contributions in the last 12 months")
    c.frame()
    L = 40
    eyebrow(c, L, 54, "The last 12 months on GitHub")
    c.add(f'<g {c.anim("rise", 0.1)}>')
    c.runs(L - 2, 112, [(f"{total:,} ", "serif", "ink"), ("contributions", "serifI", "ink")], 52, ls=-0.01)
    c.add("</g>")

    # monthly bars: peak in blue, current partial month in grey, the rest ink
    x0, x1, top, base = L, 572, 158, 296
    months = st["months"]
    last = len(months) - 1
    peak_i = max(range(last), key=lambda j: (months[j][1], j)) if last else 0
    peak = max(v for _, v in months) or 1
    slot = (x1 - x0) / len(months)
    bw = slot - 12
    c.add(f'<path d="M{x0},{base + .5}H{x1}" stroke="{PAL["rule"]}" stroke-width="1"/>')
    for j, (d, v) in enumerate(months):
        x = x0 + j * slot + (slot - bw) / 2
        h = max(2, v / peak * (base - top)) if v else 0
        fill = PAL["blue"] if j == peak_i else PAL["partial"] if j == last else PAL["ink"]
        at = 0.5 + 0.05 * j
        if h:
            c.add(f'<path d="{bar_path(x, base - h, bw, h)}" fill="{fill}" class="a-grow" '
                  f'style="animation-delay:{c.start + at:.2f}s"/>')
        c.text(x + bw / 2, base + 22, f"{d:%b}".lower(), "mono400", 11, "muted", "middle")
        if j == peak_i and v:
            c.text(x + bw / 2, base - h - 9, f"{v:,}", "mono400", 11, "blue", "middle", anim="fade", at=at + .6)

    # stats, 2 × 2 on the right
    b_d, b_c = st["busiest"]
    rng = st["best_rng"]
    blocks = [
        (f"{st['cur']}", "day streak", f"since {fmt_day(st['cur_start'])}" if st["cur"] else "start one today"),
        (f"{st['best']}", "longest streak", f"{fmt_day(rng[0])} – {fmt_day(rng[1])}" if rng else "—"),
        (f"{st['active']}", "active days", f"of the last {st['span']}"),
        (f"{b_c}", "busiest day", f"{b_d:%a}, {fmt_day(b_d)}"),
    ]
    sx = [616, 736]
    c.add(f'<path d="M{sx[0] - 20},{top - 6}V{base + 28}" stroke="{PAL["rule"]}" stroke-width="1"/>')
    for k, (big, small, sub) in enumerate(blocks):
        x, y = sx[k % 2], 196 + (k // 2) * 86
        c.add(f'<g {c.anim("rise", 0.9 + 0.1 * k)}>')
        c.text(x, y, big, "serif", 42, "ink", ls=-0.01)
        c.text(x, y + 22, small, "sans400", 13, "muted")
        c.text(x, y + 38, sub, "mono400", 10.5, "muted")
        c.add("</g>")
        if k == 0 and st["cur"]:
            dx = x + text_width(big, "serif", 42, -0.01) + 10
            c.add(f'<rect x="{dx:.1f}" y="{y - 26}" width="7" height="7" rx="2" fill="{PAL["blue"]}" '
                  f'{c.anim("pop", 1.2)}/>')
            pulse(c, dx + 3.5, y - 22.5, 4, 11, 1.6)
    return c.save()


def star_path(cx, cy, r):
    pts = []
    for k in range(10):
        a = -math.pi / 2 + k * math.pi / 5
        rr = r if k % 2 == 0 else r * 0.45
        pts.append(f"{cx + rr * math.cos(a):.2f},{cy + rr * math.sin(a):.2f}")
    return "M" + "L".join(pts) + "Z"


def repo_card(r, i, start):
    name = r["name"]
    desc = (r.get("description") or DESCRIPTIONS.get(name)
            or f"{(r.get('primaryLanguage') or {}).get('name', 'Code')} project")
    L, R = 40, W - 40
    lines = wrap(desc, "sans400", 15, R - L - 60, 2)
    rule = 104 + 23 * (len(lines) - 1) + 22
    foot = rule + 28
    c = Card(f"repo-{re.sub(r'[^A-Za-z0-9._-]', '-', name)}", W, foot + 30, start, f"{name}: {desc}")
    c.frame()
    c.add(f'<g {c.anim("rise")}>')
    c.add(f'<rect x="{L}" y="31" width="8" height="8" rx="2" fill="{PAL["blue"]}"/>')
    c.text(L + 18, 39, f"FEATURED · 0{i + 1}", "mono400", 12, "muted", ls=0.09)
    c.text(L, 76, truncate(name, "sans600", 22, R - L - 180), "sans600", 22, "ink", ls=-0.01)
    for k, ln in enumerate(lines):
        c.text(L, 104 + k * 23, ln, "sans400", 15, "soft")
    c.add("</g>")
    c.add(f'<g {c.anim("fade", 0.4)}>{arrow(R, 71, "ink", 12)}</g>')

    # footer: language · stars · forks · updated
    c.add(f'<g {c.anim("fade", 0.35)}>')
    c.add(f'<path d="M{L},{rule}H{R}" stroke="{PAL["rule"]}" stroke-width="1"/>')
    x = L
    lang = r.get("primaryLanguage") or {}
    if lang.get("name"):
        c.add(f'<circle cx="{x + 5}" cy="{foot - 4}" r="5" fill="{lang.get("color") or "#8B949E"}"/>')
        c.text(x + 16, foot, lang["name"], "mono400", 12, "ink")
        x += 16 + text_width(lang["name"], "mono400", 12) + 26
    c.add(f'<path d="{star_path(x + 6, foot - 4.5, 6.5)}" fill="{PAL["blue"]}"/>')
    s = f"{r.get('stargazerCount', 0):,}"
    c.text(x + 18, foot, s, "mono400", 12, "ink")
    x += 18 + text_width(s, "mono400", 12) + 24
    c.add(f'<g fill="none" stroke="{PAL["muted"]}" stroke-width="1.4" stroke-linecap="round">'
          f'<circle cx="{x + 2}" cy="{foot - 10}" r="2"/><circle cx="{x + 10}" cy="{foot - 10}" r="2"/>'
          f'<circle cx="{x + 6}" cy="{foot + 2}" r="2"/><path d="M{x + 2},{foot - 8}V{foot - 6}Q{x + 2},{foot - 4} {x + 4},{foot - 4}'
          f'H{x + 8}Q{x + 10},{foot - 4} {x + 10},{foot - 6}V{foot - 8}M{x + 6},{foot - 4}V{foot}"/></g>')
    c.text(x + 18, foot, f"{r.get('forkCount', 0):,}", "mono400", 12, "ink")
    when = r.get("pushedAt") or r.get("updatedAt")
    if when:
        d = dt.date.fromisoformat(when[:10])
        c.text(R, foot, f"updated {d:%b %Y}".lower(), "mono400", 12, "muted", "end")
    c.add("</g>")
    return c.save()


def graph(days, total, start):
    c = Card("graph", W, 236, start, f"Contribution graph: {total:,} contributions in the last year")
    c.frame()
    nonzero = sorted(v for _, v in days if v)

    def q(p):
        return nonzero[min(len(nonzero) - 1, int(p * len(nonzero)))] if nonzero else 1

    th = (q(.25), q(.5), q(.75))

    def level(v):
        return 0 if not v else 1 if v <= th[0] else 2 if v <= th[1] else 3 if v <= th[2] else 4

    c.add(f'<g {c.anim("rise")}>')
    c.runs(40, 58, [("Every day, ", "serif", "ink"), ("in blue", "serifI", "blue")], 34)
    c.add("</g>")
    eyebrow(c, W - 40, 56, "Refreshed daily", anchor="end")

    step, cell = 13.5, 11
    y0 = 96
    first = days[0][0]
    offset = (first.weekday() + 1) % 7  # GitHub columns start on Sunday
    weeks = (offset + len(days) + 6) // 7
    x0 = W - 40 - (weeks * step - (step - cell))  # right edge lines up with the legend
    for lbl, row in (("mon", 1), ("wed", 3), ("fri", 5)):
        c.text(x0 - 10, y0 + row * step + 9, lbl, "mono400", 10.5, "muted", "end", anim="fade", at=.1)
    last_month = None
    today_xy = None
    for k, (d, v) in enumerate(days):
        w, row = divmod(k + offset, 7)
        x, y = x0 + w * step, y0 + row * step
        if row == 0 and d.month != last_month:
            if w < weeks - 2 and (last_month is not None or d.day <= 7):
                c.text(x, y0 - 10, f"{d:%b}".lower(), "mono400", 10.5, "muted", anim="fade", at=.1)
            last_month = d.month
        at = 0.2 + w * 0.018 + row * 0.008
        c.add(f'<rect x="{x:.1f}" y="{y:.1f}" width="{cell}" height="{cell}" rx="2.5" fill="{HEAT[level(v)]}" '
              f'class="a-pop" style="animation-delay:{c.start + at:.2f}s"/>')
        today_xy = (x + cell / 2, y + cell / 2)
    pulse(c, *today_xy, 5, 13, 0.2 + weeks * 0.018 + 0.4)

    fy = y0 + 7 * step + 22
    c.text(40, fy, f"{total:,} contributions in the last year", "mono400", 11, "muted", anim="fade", at=.3)
    c.add(f'<g {c.anim("fade", .3)}>')
    lx = W - 40
    c.text(lx, fy, "more", "mono400", 11, "muted", "end")
    lx -= text_width("more", "mono400", 11) + 8 + cell
    for k in range(4, -1, -1):
        c.add(f'<rect x="{lx:.1f}" y="{fy - 10}" width="{cell}" height="{cell}" rx="2.5" fill="{HEAT[k]}"/>')
        lx -= cell + 3
    c.text(lx + cell - 5, fy, "less", "mono400", 11, "muted", "end")
    c.add("</g>")
    return c.save()


# ─── README block ────────────────────────────────────────────────────────────
def readme_block(tiles, repos, cards_dir="cards"):
    p = PROFILE
    name = f"{p['first']} {p['last']}"
    img = lambda n, alt, w="100%": f'<img src="{cards_dir}/{n}.svg" width="{w}" alt="{escape(alt)}">'
    out = ['<p align="center">',
           f'  <a href="{p["url"]}">{img("hero", name + " — " + p["eyebrow"])}</a>', "</p>",
           '<p align="center">']
    out += [f'  <a href="{t["url"]}">{img("tile-" + t["key"], t["label"], "32.5%")}</a>' for t in tiles]
    out += ["</p>", '<p align="center">',
            f'  <a href="{p["url"]}?tab=overview">{img("activity", "Contributions per month")}</a>', "</p>"]
    for r, n in repos:
        out += ['<p align="center">', f'  <a href="{r["url"]}">{img(n, r["name"])}</a>', "</p>"]
    out += ['<p align="center">',
            f'  <a href="{p["url"]}?tab=repositories">{img("graph", "Contribution graph")}</a>', "</p>"]
    return "\n".join(out)


def update_readme(block):
    start, end = "<!-- cards:start -->", "<!-- cards:end -->"
    text = README.read_text(encoding="utf-8") if README.exists() else ""
    if start in text and end in text:
        new = re.sub(re.escape(start) + r".*?" + re.escape(end),
                     lambda _: f"{start}\n{block}\n{end}", text, flags=re.S)
        if new != text:
            README.write_text(new, encoding="utf-8")
    else:
        print(f"README has no {start} … {end} markers; paste this block:\n{block}")


def preview_html(block):
    """A GitHub-like page for screenshots of the STATIC render."""
    return ("<!doctype html><meta charset=utf-8><title>Profile preview</title>"
            "<style>body{margin:0;background:#0d1117;font-family:system-ui}"
            "main{max-width:830px;margin:32px auto;padding:0 16px}"
            "img{max-width:100%;vertical-align:middle}p{margin:0 0 16px;text-align:center}"
            "@media (prefers-color-scheme:light){body{background:#fff}}</style>"
            f"<main>{block}</main>")


def main():
    user, total, days, repos, pinned = load()
    st = stats(days)
    names = FEATURED or [n for n in pinned if not repos.get(n, {}).get("isFork")]
    featured = [repos[n] for n in names if n in repos][:MAX_FEATURED]

    t = 0.0
    hero(t)
    for i, tl in enumerate(TILES[:3]):
        tile(tl, i, t + 0.5 + 0.12 * i)
    t += 0.5 + 0.36 + STEP
    activity(st, total, t)
    t += 1.1
    repo_cards = []
    for i, r in enumerate(featured):
        repo_cards.append((r, repo_card(r, i, t)))
        t += STEP
    graph(days, total, t)

    if STATIC:
        (OUT / "preview.html").write_text(preview_html(readme_block(TILES[:3], repo_cards, ".")), encoding="utf-8")
        print(f"Static final frames -> {OUT}/ (open preview.html to screenshot)")
    else:
        update_readme(readme_block(TILES[:3], repo_cards))
        keep = {n for _, n in repo_cards}  # drop cards for repos no longer featured
        for f in OUT.glob("repo-*.svg"):
            if f.stem not in keep:
                f.unlink()
        print(f"Rendered {len(list(OUT.glob('*.svg')))} cards -> {OUT}/")


if __name__ == "__main__":
    main()
