import base64
import ctypes
import json
import os
import queue
import random
import shutil
import sys
import threading
import time
import tkinter as tk
import tkinter.font as tkfont
from tkinter import filedialog, messagebox, simpledialog

try:
    ctypes.windll.shcore.SetProcessDpiAwareness(2)
except Exception:
    try:
        ctypes.windll.user32.SetProcessDPIAware()
    except Exception:
        pass

_MISSING_PKGS = []
try:
    import numpy as np
except ImportError:
    _MISSING_PKGS.append("numpy")
try:
    import cv2
except ImportError:
    _MISSING_PKGS.append("opencv-python")
try:
    import mss
except ImportError:
    _MISSING_PKGS.append("mss")
try:
    import keyboard
except ImportError:
    _MISSING_PKGS.append("keyboard")
try:
    import win32api
    import win32con
    import win32gui
except ImportError:
    _MISSING_PKGS.append("pywin32")

if _MISSING_PKGS:
    message = ("Missing packages: %s\n\nThis script is running on:\n    %s\n\n"
               "Install them into that interpreter:\n\n\"%s\" -m pip install %s"
               % (", ".join(_MISSING_PKGS), sys.executable, sys.executable,
                  " ".join(_MISSING_PKGS)))
    print(message)
    try:
        _r = tk.Tk()
        _r.withdraw()
        messagebox.showerror("Missing packages", message)
        _r.destroy()
    except Exception:
        pass
    sys.exit(1)

DEBUG = False
_last_log = {}


def log(message, key=None, every=5.0):
    key = key or message
    now = time.monotonic()
    if now - _last_log.get(key, -1e9) < every:
        return
    _last_log[key] = now
    try:
        print(message, file=sys.stderr)
    except Exception:
        pass


def debug(message):
    if DEBUG:
        try:
            print(message)
        except Exception:
            pass

_user32 = ctypes.windll.user32
_gdi32 = ctypes.windll.gdi32


class _POINT(ctypes.Structure):
    _fields_ = [("x", ctypes.c_long), ("y", ctypes.c_long)]


class _BITMAPINFOHEADER(ctypes.Structure):
    _fields_ = [("biSize", ctypes.c_uint32), ("biWidth", ctypes.c_int32),
                ("biHeight", ctypes.c_int32), ("biPlanes", ctypes.c_uint16),
                ("biBitCount", ctypes.c_uint16), ("biCompression", ctypes.c_uint32),
                ("biSizeImage", ctypes.c_uint32), ("biXPelsPerMeter", ctypes.c_int32),
                ("biYPelsPerMeter", ctypes.c_int32), ("biClrUsed", ctypes.c_uint32),
                ("biClrImportant", ctypes.c_uint32)]


class _BITMAPINFO(ctypes.Structure):
    _fields_ = [("bmiHeader", _BITMAPINFOHEADER), ("bmiColors", ctypes.c_uint32 * 3)]


_user32.RealChildWindowFromPoint.argtypes = [ctypes.c_void_p, _POINT]
_user32.RealChildWindowFromPoint.restype = ctypes.c_void_p
_user32.GetWindowDC.argtypes = [ctypes.c_void_p]
_user32.GetWindowDC.restype = ctypes.c_void_p
_user32.ReleaseDC.argtypes = [ctypes.c_void_p, ctypes.c_void_p]
_user32.PrintWindow.argtypes = [ctypes.c_void_p, ctypes.c_void_p, ctypes.c_uint]
_gdi32.CreateCompatibleDC.argtypes = [ctypes.c_void_p]
_gdi32.CreateCompatibleDC.restype = ctypes.c_void_p
_gdi32.CreateCompatibleBitmap.argtypes = [ctypes.c_void_p, ctypes.c_int, ctypes.c_int]
_gdi32.CreateCompatibleBitmap.restype = ctypes.c_void_p
_gdi32.SelectObject.argtypes = [ctypes.c_void_p, ctypes.c_void_p]
_gdi32.SelectObject.restype = ctypes.c_void_p
_gdi32.DeleteObject.argtypes = [ctypes.c_void_p]
_gdi32.DeleteDC.argtypes = [ctypes.c_void_p]
_gdi32.GetDIBits.argtypes = [ctypes.c_void_p, ctypes.c_void_p, ctypes.c_uint,
                             ctypes.c_uint, ctypes.c_void_p, ctypes.c_void_p,
                             ctypes.c_uint]
_gdi32.SetLayout.argtypes = [ctypes.c_void_p, ctypes.c_uint32]
_gdi32.SetLayout.restype = ctypes.c_uint32

try:
    _user32.GetDpiForWindow.argtypes = [ctypes.c_void_p]
    _user32.GetDpiForWindow.restype = ctypes.c_uint
    _HAS_WINDOW_DPI = True
except AttributeError:
    _HAS_WINDOW_DPI = False


def window_dpi(hwnd):
    if not hwnd or not _HAS_WINDOW_DPI:
        return None
    try:
        return int(_user32.GetDpiForWindow(hwnd)) or None
    except Exception:
        return None


DEAD_CLASSES = {"Intermediate D3D Window"}

_HERE = os.path.dirname(os.path.abspath(__file__))
CONFIG_PATH = os.path.join(_HERE, "item_config.json")
BACKUP_PATH = os.path.join(_HERE, "item_config.previous.json")
DEBUG_IMAGE = os.path.join(_HERE, "item_debug.png")
STATS_PATH = os.path.join(_HERE, "item_stats.json")
ITEM_DIR = os.path.join(_HERE, "items")
REMOVED_DIR = os.path.join(ITEM_DIR, "removed")

BUTTONS = {
    "left": (win32con.WM_LBUTTONDOWN, win32con.WM_LBUTTONUP, win32con.MK_LBUTTON),
    "middle": (win32con.WM_MBUTTONDOWN, win32con.WM_MBUTTONUP, win32con.MK_MBUTTON),
    "right": (win32con.WM_RBUTTONDOWN, win32con.WM_RBUTTONUP, win32con.MK_RBUTTON),
}

TARGET_TTL = 1.0
CONFIG_VERSION = 1

RENAMED = {"potion": "green potion"}

DEFAULT_ORDER = ["bolt", "ghost", "green potion", "purple potion",
                 "mushroom", "feather", "fang", "wax seal", "green drop"]

DISPLAY_NAMES = {"bolt": "Bolt", "ghost": "Ghost", "green potion": "Green Potion",
                 "purple potion": "Purple Potion", "mushroom": "Mushroom",
                 "feather": "Feather", "fang": "Fang",
                 "wax seal": "Wax Seal", "green drop": "Green Drop"}


CUSTOM_NAMES = {}


def nice(name):
    if not name:
        return "Unidentified"
    return CUSTOM_NAMES.get(name) or DISPLAY_NAMES.get(name, name.title())

SHELVE_FOR = 60.0

WATCH_AT_LEAST = 1.0

LIVE_FPS = 20
CAPTURE_STEPS = [5, 10, 15, 20, 30, 60]

LIVE_STALE = 0.25

FRAME_TOO_OLD = 1.0

PING_EVERY = 0.4

SLOW_PASS = 1.0
SHELVE_RADIUS = 46

LOST_LOOKS = 2

CONFIRM_EASE = 0.85

POPUP_GRACE = 6.0

POPUP_CLOSE = 0.6

LOSS_UNDO = 12.0

PRIZE_WINDOW = 25.0
PRIZE_TRIES = 6

PRIZE_RECHECK = 1.0
PRIZE_RECLICKS = 3

PRIZE_RELAX = 0.85
PRIZE_RELAX_LEAD = 0.015

CLAIM_WINDOW = 12.0

PRIZE_STEPS = [0.70, 0.75, 0.80, 0.85, 0.90]

PRIZE_WAIT_STEPS = [0.0, 0.3, 0.5, 0.75, 1.0, 1.5, 2.0, 3.0]

ORDINALS = {1: "st", 2: "nd", 3: "rd"}

SCORE_STEPS = [0.75, 0.80, 0.84, 0.88, 0.91, 0.94]
POLL_STEPS = [0.0, 0.15, 0.25, 0.35, 0.50, 0.75, 1.0]
CLICK_EVERY_STEPS = [0.3, 0.45, 0.6, 0.8, 1.0, 1.5, 2.0]
PATIENCE_STEPS = [4.0, 6.0, 8.0, 12.0, 16.0, 25.0, 40.0]
MARGIN_STEPS = [0, 10, 20, 30, 50, 80]
SPREAD_STEPS = [0, 1, 2, 3, 4, 6]
HOVER_STEPS = [0, 10, 15, 20, 25, 30, 35, 45, 60, 80, 120]
DWELL_STEPS = [5, 10, 15, 20, 25, 30, 40, 50, 65, 80]

LOAD_BOUNDS = {
    "min_score": (0.5, 0.99),
    "prize_score": (0.5, 0.99),
    "prize_wait": (0.0, 6.0),
    "poll": (0.0, WATCH_AT_LEAST),
    "click_every": (0.15, 5.0),
    "patience": (1.0, 120.0),
    "margin": (0, 400),
    "spread": (0, 20),
    "hover_ms": (0, 1000),
    "dwell_ms": (1, 1000),
}

def clamp(value, low, high):
    return max(low, min(high, value))


def ladder(values, current, direction):
    if direction > 0:
        above = [v for v in values if v > current + 1e-12]
        return min(above) if above else current
    below = [v for v in values if v < current - 1e-12]
    return max(below) if below else current


def commas(n):
    return "{:,}".format(int(n))


def clock(seconds):
    seconds = int(max(0, seconds))
    h, rem = divmod(seconds, 3600)
    m, sec = divmod(rem, 60)
    if h:
        return "%d:%02d:%02d" % (h, m, sec)
    return "%d:%02d" % (m, sec)


def backup_config():
    try:
        if os.path.exists(CONFIG_PATH):
            shutil.copyfile(CONFIG_PATH, BACKUP_PATH)
            return True
    except Exception as e:
        log("Config backup failed: %s" % e)
    return False


def _is_num(v):
    return isinstance(v, (int, float)) and not isinstance(v, bool)


def _fmt_rect(rect):
    if not rect:
        return "not set"
    return "%d, %d  %d\u00d7%d" % tuple(rect)


def _fmt_point(point):
    return "not set" if not point else "%d, %d" % tuple(point)


def raise_timer_resolution():
    try:
        ctypes.windll.winmm.timeBeginPeriod(1)
        return True
    except Exception:
        return False


def release_timer_resolution():
    try:
        ctypes.windll.winmm.timeEndPeriod(1)
    except Exception:
        pass


SPIN_MIN = 0.0004
SPIN_MAX = 0.0040
_spin_budget = [0.0018]


def precise_sleep(duration, stop=None):
    if duration <= 0:
        return
    end = time.perf_counter() + duration
    budget = _spin_budget[0]
    coarse = duration - budget
    if coarse > 0:
        before = time.perf_counter()
        if stop is not None:
            if stop.wait(coarse):
                return
        else:
            time.sleep(coarse)
        over = time.perf_counter() - before - coarse
        _spin_budget[0] = clamp(max(over * 1.3, _spin_budget[0] * 0.995),
                                SPIN_MIN, SPIN_MAX)
    if stop is None:
        while time.perf_counter() < end:
            pass
    else:
        while time.perf_counter() < end:
            if stop.is_set():
                return


def lparam(x, y):
    return ((y & 0xFFFF) << 16) | (x & 0xFFFF)


SCALE_MIN, SCALE_MAX = 0.55, 1.75

SWEEP_SCALES = (0.60, 0.75, 0.90, 1.05, 1.25, 1.50, 1.75)

SHAPE_WEIGHT = 5.0

HUE_MARGIN = 7
SAT_FLOOR = 70
VAL_FLOOR = 70
PATCH_SLACK = 5
PROBE_SLACK = 3

PROBE_FACTORS = (1.0, 0.88, 1.13, 0.78, 1.28)
HUNT_DIVISOR = 1
HUNT_SLACK = 0.94

SCALE_GRAIN = 40.0


def grain(scale):
    return round(scale * SCALE_GRAIN) / SCALE_GRAIN


LOOK_BUDGET = 0.12
_rotation = [0]

SPLIT_LIMIT = 4

SURE_SCORE = 0.90

HOPELESS = 0.45

CREAM_LO, CREAM_HI = (0, 0, 185), (179, 48, 255)


def _row_colour(bgr, y, x0, x1):
    return np.median(bgr[y, x0:x1].astype(np.float32), axis=0)


def group_rows(blobs):
    if not blobs:
        return []
    heights = sorted(b[3] for b in blobs)
    tol = max(8, heights[len(heights) // 2] * 0.7)
    ordered = sorted(blobs, key=lambda b: b[1] + b[3] / 2.0)
    rows, cur = [], [ordered[0]]
    for b in ordered[1:]:
        ref = sum(c[1] + c[3] / 2.0 for c in cur) / float(len(cur))
        if abs((b[1] + b[3] / 2.0) - ref) <= tol:
            cur.append(b)
        else:
            rows.append(cur)
            cur = [b]
    rows.append(cur)
    return [sorted(r, key=lambda b: b[0]) for r in rows]


def _bands(lo, hi):
    if lo < 0:
        return [((lo + 180) % 180, 179), (0, max(0, hi))]
    if hi > 179:
        return [(lo, 179), (0, hi - 180)]
    return [(lo, hi)]


class Template:
    def __init__(self, name, image, mask, click, enabled=True):
        self.name = name
        self.image = image
        self.mask = mask
        self.click = (float(click[0]), float(click[1]))
        self.enabled = enabled
        self.rank = 0
        self.prize = None
        self._cache = {}
        self._colours = None
        self._measure()

    def _measure(self):
        hsv = cv2.cvtColor(self.image, cv2.COLOR_BGR2HSV)
        body = self.mask > 128
        strong = body & (hsv[..., 1] > 90) & (hsv[..., 2] > 60)
        if np.count_nonzero(strong) < 20:
            strong = body
        hue = hsv[..., 0][strong]
        hist = np.bincount(hue, minlength=180).astype(np.float32)
        total = float(hist.sum())
        lo = hi = int(np.argmax(hist))
        acc = hist[lo]
        while acc < 0.85 * total and (hi - lo) < 50:
            down, up = hist[(lo - 1) % 180], hist[(hi + 1) % 180]
            if down >= up:
                lo -= 1
                acc += down
            else:
                hi += 1
                acc += up
        lo, hi = lo - HUE_MARGIN, hi + HUE_MARGIN
        if hi > 179 and lo > 179:
            lo, hi = lo - 180, hi - 180
        if lo < -180:
            lo, hi = lo + 180, hi + 180
        self.hue = (lo, hi)
        sat = hsv[..., 1][strong]
        val = hsv[..., 2][strong]
        self.sat_min = int(max(SAT_FLOOR, np.percentile(sat, 10) * 0.80))
        self.val_min = int(max(VAL_FLOOR, np.percentile(val, 10) * 0.75))

        gate = self.gate_mask(self.image)
        gate[~body] = 0
        ys, xs = np.nonzero(gate)
        if xs.size < 10:
            ys, xs = np.nonzero(body)
        self.gate_box = (int(xs.min()), int(ys.min()),
                         int(xs.max() - xs.min() + 1),
                         int(ys.max() - ys.min() + 1))
        self.gate_area = int(np.count_nonzero(gate)) or int(np.count_nonzero(body))

    def colours(self):
        if self._colours is None:
            self._colours = colour_print(self.image, self.mask)
        return self._colours

    def gate_mask(self, bgr, hsv=None):
        if hsv is None:
            hsv = cv2.cvtColor(bgr, cv2.COLOR_BGR2HSV)
        out = None
        for a, b in _bands(*self.hue):
            part = cv2.inRange(hsv, (max(0, a), self.sat_min, self.val_min),
                               (min(179, b), 255, 255))
            out = part if out is None else cv2.bitwise_or(out, part)
        return out

    @property
    def size(self):
        h, w = self.image.shape[:2]
        return w, h

    def at(self, scale):
        key = scale = grain(scale)
        hit = self._cache.get(key)
        if hit is not None:
            return hit
        w, h = self.size
        nw, nh = max(4, int(round(w * scale))), max(4, int(round(h * scale)))
        interp = cv2.INTER_AREA if scale < 1 else cv2.INTER_CUBIC
        img = cv2.resize(self.image, (nw, nh), interpolation=interp)
        mask = cv2.resize(self.mask, (nw, nh), interpolation=cv2.INTER_NEAREST)
        if len(self._cache) > 600:
            self._cache.clear()
        self._cache[key] = (img, cv2.merge([mask, mask, mask]), mask)
        return self._cache[key]

    def centred(self, scale):
        key = ("c", grain(scale))
        hit = self._cache.get(key)
        if hit is not None:
            return hit
        img, _mask3, mask = self.at(scale)
        idx = np.flatnonzero(np.repeat((mask > 128).ravel(), 3))
        values = img.ravel()[idx].astype(np.float32)
        values -= values.mean() if values.size else 0.0
        self._cache[key] = (idx, values)
        return self._cache[key]

    def click_at(self, x, y, scale):
        return (int(round(x + self.click[0] * scale)),
                int(round(y + self.click[1] * scale)))


def load_templates(blob):
    out = []
    for name, entry in sorted(json.loads(blob).items()):
        img = cv2.imdecode(np.frombuffer(base64.b64decode(entry["image"]),
                                         np.uint8), cv2.IMREAD_COLOR)
        mask = cv2.imdecode(np.frombuffer(base64.b64decode(entry["mask"]),
                                          np.uint8), cv2.IMREAD_GRAYSCALE)
        if img is None or mask is None:
            continue
        template = Template(name, img, mask, entry["click"])
        if entry.get("prize_image") and entry.get("prize_mask"):
            pimg = cv2.imdecode(np.frombuffer(
                base64.b64decode(entry["prize_image"]), np.uint8),
                cv2.IMREAD_COLOR)
            pmask = cv2.imdecode(np.frombuffer(
                base64.b64decode(entry["prize_mask"]), np.uint8),
                cv2.IMREAD_GRAYSCALE)
            if pimg is not None and pmask is not None:
                h, w = pmask.shape
                template.prize = Template(name, pimg, pmask, (w / 2.0, h / 2.0))
        out.append(template)
    return out


class Hit:
    def __init__(self, template, score, box, scale):
        self.template = template
        self.name = template.name
        self.score = score
        self.box = box
        self.scale = scale

    @property
    def centre(self):
        x, y, w, h = self.box
        return int(x + w / 2), int(y + h / 2)

    @property
    def click(self):
        return self.template.click_at(self.box[0], self.box[1], self.scale)

    def overlaps(self, other, cut=0.30):
        ax, ay, aw, ah = self.box
        bx, by, bw, bh = other.box
        ix = max(0, min(ax + aw, bx + bw) - max(ax, bx))
        iy = max(0, min(ay + ah, by + bh) - max(ay, by))
        if not ix or not iy:
            return False
        inter = ix * iy
        return inter / float(aw * ah + bw * bh - inter) > cut

    def near(self, point, slack=26):
        x, y, w, h = self.box
        cx, cy = x + w / 2.0, y + h / 2.0
        return abs(cx - point[0]) <= w / 2.0 + slack and \
            abs(cy - point[1]) <= h / 2.0 + slack


def _match(scene, template, mask3):
    if scene.shape[0] < template.shape[0] or scene.shape[1] < template.shape[1]:
        return None
    try:
        out = cv2.matchTemplate(scene, template, cv2.TM_CCOEFF_NORMED,
                                mask=mask3)
    except cv2.error:
        return None
    out[~np.isfinite(out)] = 0.0
    return out


def _offsets(template, scale):
    key = ("o", grain(scale))
    hit = template._cache.get(key)
    if hit is not None:
        return hit
    img, _mask3, _mask = template.at(scale)
    th, tw = img.shape[:2]
    idx, centred = template.centred(scale)
    ys, rest = np.divmod(idx, tw * 3)
    xs, cs = np.divmod(rest, 3)
    thin = slice(None, None, 4)
    tvals = centred[thin] - centred[thin].mean() if centred.size else centred
    template._cache[key] = ((ys, xs, cs, centred),
                            (ys[thin], xs[thin], cs[thin], tvals))
    return template._cache[key]


def _scores(region, offs, th, tw, step):
    ys, xs, cs, tvals = offs
    oy = np.arange(0, region.shape[0] - th + 1, step)
    ox = np.arange(0, region.shape[1] - tw + 1, step)
    vals = region[ys[:, None, None] + oy[None, :, None],
                  xs[:, None, None] + ox[None, None, :],
                  cs[:, None, None]].astype(np.float32)
    vals = vals.reshape(len(ys), -1)
    vals -= vals.mean(axis=0, keepdims=True)
    num = tvals.dot(vals)
    den = np.sqrt(np.einsum("ij,ij->j", vals, vals) * float(tvals.dot(tvals)))
    with np.errstate(invalid="ignore", divide="ignore"):
        scores = np.where(den > 1e-6, num / np.maximum(den, 1e-6), 0.0)
    return scores.reshape(len(oy), len(ox)), oy, ox


def _best_at(view, template, scale, x, y, slack=PATCH_SLACK):
    img, _mask3, _mask = template.at(scale)
    th, tw = img.shape[:2]
    if tw < 6 or th < 6:
        return None
    x, y = int(x), int(y)
    slack = max(slack, int(0.12 * max(tw, th)))
    x0, y0 = max(0, x - slack), max(0, y - slack)
    x1 = min(view.shape[1], x + tw + slack)
    y1 = min(view.shape[0], y + th + slack)
    if x1 <= x0 or y1 <= y0:
        return None
    region = view[y0:y1, x0:x1]
    if region.shape[0] < th or region.shape[1] < tw:
        return None
    full, thin = _offsets(template, scale)
    if full[0].size < 60:
        return None

    ny, nx = region.shape[0] - th + 1, region.shape[1] - tw + 1
    if ny * nx > 49 and thin[0].size >= 40:
        rough, oy, ox = _scores(region, thin, th, tw, 2)
        r, c = np.unravel_index(int(np.argmax(rough)), rough.shape)
        cy, cx = int(oy[r]), int(ox[c])
        ry0, rx0 = max(0, cy - 2), max(0, cx - 2)
        sub = region[ry0:min(region.shape[0], cy + 3 + th),
                     rx0:min(region.shape[1], cx + 3 + tw)]
        fine, oy, ox = _scores(sub, full, th, tw, 1)
        r, c = np.unravel_index(int(np.argmax(fine)), fine.shape)
        return (float(fine[r, c]),
                (x0 + rx0 + int(ox[c]), y0 + ry0 + int(oy[r])), (tw, th))
    fine, oy, ox = _scores(region, full, th, tw, 1)
    r, c = np.unravel_index(int(np.argmax(fine)), fine.shape)
    return float(fine[r, c]), (x0 + int(ox[c]), y0 + int(oy[r])), (tw, th)


def _split_oversized(view, template, box, bounds):
    x, y, w, h = box
    patch = view[y:y + h, x:x + w]
    if patch.size == 0:
        return []
    hsv = cv2.cvtColor(patch, cv2.COLOR_BGR2HSV)
    lo, hi = template.hue
    hue_ok = np.zeros(patch.shape[:2], bool)
    for a, b in _bands(lo, hi):
        hue_ok |= (hsv[..., 0] >= a) & (hsv[..., 0] <= b)
    sat, val = hsv[..., 1], hsv[..., 2]
    out = []
    for tighten in (1.35, 1.7):
        keep = (hue_ok & (sat >= min(250, template.sat_min * tighten))
                & (val >= min(250, template.val_min * tighten)))
        mask = cv2.morphologyEx(keep.astype(np.uint8) * 255, cv2.MORPH_OPEN,
                                np.ones((3, 3), np.uint8))
        n, _l, st, _c = cv2.connectedComponentsWithStats(mask, 8)
        for i in range(1, n):
            sx, sy, sw, sh, area = st[i]
            if _implied(area, template, bounds) is None:
                continue
            if not _solid(sw, sh, area):
                continue
            out.append((int(x + sx), int(y + sy), int(sw), int(sh), int(area)))
        if out:
            break
    out.sort(key=lambda b: -(b[4] / float(max(1, b[2] * b[3]))))
    return out[:SPLIT_LIMIT]


def _weigh(view, template, tx, ty, scale, threshold):
    best, best_scale = None, scale
    for factor in PROBE_FACTORS:
        size = clamp(scale * factor, SCALE_MIN * 0.75, SCALE_MAX * 1.3)
        got = _best_at(view, template, size, tx, ty, slack=PROBE_SLACK)
        if got is None:
            continue
        if best is None or got[0] > best[0]:
            best, best_scale = got, size
        if got[0] >= SURE_SCORE:
            break
    if best is None or best[0] < HOPELESS:
        return None

    x, y = best[1]
    for factor in (1.0, 0.94, 1.07):
        size = best_scale * factor
        got = _best_at(view, template, size, x, y, slack=PATCH_SLACK)
        if got is not None and got[0] > best[0]:
            best, best_scale = got, size
    score, (bx, by), (tw, th) = best
    if score < threshold:
        return None
    return Hit(template, score, (bx, by, tw, th), best_scale)


def _solid(w, h, area):
    if w < 4 or h < 4:
        return False
    if not (0.35 <= w / float(h) <= 3.0):
        return False
    return area / float(w * h) >= 0.34


def _implied(area, template, bounds):
    scale = (area / float(template.gate_area)) ** 0.5
    lo, hi = bounds
    return scale if lo * 0.75 <= scale <= hi * 1.3 else None


def _candidates(view, template, bounds, min_area=30, hsv=None):
    mask = template.gate_mask(view, hsv)
    mask = cv2.morphologyEx(mask, cv2.MORPH_OPEN, np.ones((3, 3), np.uint8))
    mask = cv2.morphologyEx(mask, cv2.MORPH_CLOSE, np.ones((5, 5), np.uint8))
    n, _lab, st, _ce = cv2.connectedComponentsWithStats(mask, 8)
    gx, gy, gw, gh = template.gate_box
    blobs = []
    for i in range(1, n):
        x, y, w, h, area = st[i]
        if area < min_area:
            continue
        implied = _implied(area, template, bounds)
        if implied is None:
            if area > template.gate_area * bounds[1] ** 2:
                blobs.extend(_split_oversized(view, template, (x, y, w, h),
                                              bounds))
            continue
        if not _solid(w, h, area):
            continue
        blobs.append((x, y, w, h, area))

    gate_aspect = gw / float(max(1, gh))
    out = []
    for x, y, w, h, area in blobs:
        by_box = 0.5 * (w / float(gw) + h / float(gh))
        by_area = (area / float(template.gate_area)) ** 0.5
        guesses = [by_box] if abs(by_box - by_area) < 0.08 * by_box \
            else [by_box, by_area]
        for guess in guesses:
            if not (bounds[0] * 0.75 <= guess <= bounds[1] * 1.3):
                continue
            scale = min(bounds[1], max(bounds[0], guess))
            cx, cy = x + w / 2.0, y + h / 2.0
            shape = abs(np.log((w / float(max(1, h))) / gate_aspect))
            size = abs(np.log(max(1, area) / float(template.gate_area)))
            out.append((int(round(cx - (gx + gw / 2.0) * scale)),
                        int(round(cy - (gy + gh / 2.0) * scale)), scale,
                        shape * SHAPE_WEIGHT + size))
    out.sort(key=lambda c: c[3])
    return [(x, y, s) for x, y, s, _rank in out]


def _crop_margin(frame, margin):
    if not margin:
        return frame, 0, 0
    h, w = frame.shape[:2]
    return (frame[margin:max(margin + 1, h - margin),
                  margin:max(margin + 1, w - margin)], margin, margin)


def find_items(frame, templates, threshold=0.88, margin=0, thorough=False,
               limit=10, everything=False, deadline=None, report=None):
    if report is not None:
        report["complete"] = True
    if frame is None or frame.size == 0:
        return []
    view, ox, oy = _crop_margin(frame, margin)
    if min(view.shape[:2]) < 16:
        return []

    hits = []
    if thorough:
        for template in templates:
            if template.enabled or everything:
                hits.extend(_sweep(view, template, threshold))
    else:
        if HUNT_DIVISOR > 1:
            small = cv2.resize(view, None, fx=1.0 / HUNT_DIVISOR,
                               fy=1.0 / HUNT_DIVISOR,
                               interpolation=cv2.INTER_AREA)
        else:
            small = view
        loose = threshold * HUNT_SLACK
        g = GATE_DIVISOR if HUNT_DIVISOR == 1 else 1
        gate_view = cv2.resize(small, None, fx=1.0 / g, fy=1.0 / g,
                               interpolation=cv2.INTER_AREA) if g > 1 else small
        gate_hsv = cv2.cvtColor(gate_view, cv2.COLOR_BGR2HSV)
        _weighed.tick()
        bounds = (SCALE_MIN / (HUNT_DIVISOR * g),
                  SCALE_MAX / (HUNT_DIVISOR * g))
        first, rest = [], []
        for template in templates:
            if not (template.enabled or everything):
                continue
            found = _candidates(gate_view, template, bounds,
                                min_area=max(4, 14 // (g * g)), hsv=gate_hsv)
            for i, (tx, ty, scale) in enumerate(found[:limit]):
                (first if i == 0 else rest).append(
                    (template, tx * g, ty * g, scale * g))
        if rest and deadline is not None:
            k = _rotation[0] % len(rest)
            rest = rest[k:] + rest[:k]
        done = 0
        for n, (template, tx, ty, scale) in enumerate(first + rest):
            if n >= len(first):
                if deadline is not None and time.perf_counter() > deadline:
                    if report is not None:
                        report["complete"] = False
                    break
                done += 1
            rough = _weighed(small, template, tx, ty, scale, loose)
            if rough is None:
                continue
            hit = rough if HUNT_DIVISOR == 1 else \
                _confirm(view, template, rough)
            if hit is not None and hit.score >= threshold:
                hits.append(hit)
        _rotation[0] += done

    for hit in hits:
        hit.box = (hit.box[0] + ox, hit.box[1] + oy, hit.box[2], hit.box[3])

    hits.sort(key=lambda hit: -hit.score)
    kept = []
    for hit in hits:
        if any(hit.overlaps(k) for k in kept):
            continue
        kept.append(hit)
    return kept


GATE_DIVISOR = 1


class _WeighMemory:
    KEEP = 3.0
    SAME = 3.5

    def __init__(self):
        self.memo = {}
        self.now = 0.0
        self.lock = threading.Lock()

    def tick(self):
        self.now = time.monotonic()
        if len(self.memo) > 4000:
            with self.lock:
                self.memo = {k: v for k, v in self.memo.items()
                             if self.now - v[2] < self.KEEP}

    @staticmethod
    def thumb(view, template, x, y, scale):
        w, h = template.size
        w, h = int(w * scale), int(h * scale)
        x0, y0 = max(0, int(x) - 4), max(0, int(y) - 4)
        patch = view[y0:y0 + h + 8, x0:x0 + w + 8]
        if patch.size == 0:
            return None
        return cv2.resize(patch, (10, 10),
                          interpolation=cv2.INTER_AREA).astype(np.int16)

    def __call__(self, view, template, x, y, scale, threshold):
        key = (id(template), view.shape[:2], int(x) // 3, int(y) // 3,
               round(scale * 25), round(threshold, 3))
        thumb = self.thumb(view, template, x, y, scale)
        with self.lock:
            kept = self.memo.get(key)
        if kept is not None and thumb is not None and \
                self.now - kept[2] < self.KEEP and \
                float(np.abs(kept[0] - thumb).mean()) < self.SAME:
            hit = kept[1]
            return None if hit is None else Hit(template, hit.score,
                                                hit.box, hit.scale)
        hit = _weigh(view, template, x, y, scale, threshold)
        if thumb is not None:
            with self.lock:
                self.memo[key] = (thumb, hit, self.now)
        return hit


_weighed = _WeighMemory()


class _FloorMemory:
    MOVED = 10

    def __init__(self):
        self.lock = threading.Lock()
        self.last = None
        self.reused = 0

    @staticmethod
    def grid(frame):
        return cv2.resize(frame, (64, 36),
                          interpolation=cv2.INTER_AREA).astype(np.int16)

    def find(self, frame, templates, threshold, margin, budget=LOOK_BUDGET):
        if frame is None or frame.size == 0:
            return []
        grid = self.grid(frame)
        settings = (frame.shape, round(threshold, 4), margin,
                    tuple((id(t), t.enabled) for t in templates))
        with self.lock:
            last = self.last
        if last is not None and last[3] and last[1] == settings and \
                int(np.abs(grid - last[0]).max()) <= self.MOVED:
            self.reused += 1
            return [Hit(h.template, h.score, h.box, h.scale) for h in last[2]]
        report = {}
        deadline = time.perf_counter() + budget if budget else None
        hits = find_items(frame, templates, threshold=threshold, margin=margin,
                          deadline=deadline, report=report)
        with self.lock:
            self.last = (grid, settings,
                         [Hit(h.template, h.score, h.box, h.scale)
                          for h in hits], report.get("complete", True))
        return hits

    def forget(self):
        with self.lock:
            self.last = None


floor_memory = _FloorMemory()


def track(frame, hit, threshold, drift=14):
    if frame is None or hit is None:
        return None
    best = None
    for factor in (1.0, 0.95, 1.05):
        size = min(SCALE_MAX, max(SCALE_MIN, hit.scale * factor))
        got = _best_at(frame, hit.template, size, hit.box[0], hit.box[1],
                       slack=drift)
        if got is not None and (best is None or got[0] > best[0][0]):
            best = (got, size)
        if best is not None and best[0][0] >= SURE_SCORE:
            break
    if best is None or best[0][0] < threshold:
        return None
    (score, (bx, by), (tw, th)), size = best
    return Hit(hit.template, score, (bx, by, tw, th), size)


def _confirm(view, template, rough):
    scale = rough.scale * HUNT_DIVISOR
    x = rough.box[0] * HUNT_DIVISOR
    y = rough.box[1] * HUNT_DIVISOR
    best, best_scale = None, scale
    for factor in (1.0, 0.93, 1.08):
        size = min(SCALE_MAX, max(SCALE_MIN, scale * factor))
        got = _best_at(view, template, size, x, y, slack=PATCH_SLACK)
        if got is not None and (best is None or got[0] > best[0]):
            best, best_scale = got, size
        if best is not None and best[0] >= SURE_SCORE:
            break
    if best is None:
        return None
    score, (bx, by), (tw, th) = best
    return Hit(template, score, (bx, by, tw, th), best_scale)


def _sweep(view, template, threshold):
    out = []
    for scale in SWEEP_SCALES:
        img, mask3, _m = template.at(scale)
        th, tw = img.shape[:2]
        response = _match(view, img, mask3)
        if response is None:
            continue
        work = response
        for _ in range(8):
            _mn, value, _ml, where = cv2.minMaxLoc(work)
            if value < threshold:
                break
            out.append(Hit(template, float(value),
                           (int(where[0]), int(where[1]), tw, th), scale))
            x0, y0 = max(0, where[0] - tw // 2), max(0, where[1] - th // 2)
            work[y0:where[1] + th // 2, x0:where[0] + tw // 2] = -1.0
    return out


CHOICE_ASPECT = (0.30, 3.0)
CHOICE_ROW = (2, 8)


def choice_row(frame, margin=0):
    if frame is None or frame.size == 0:
        return None
    view, ox, oy = _crop_margin(frame, margin)
    vh, vw = view.shape[:2]

    for _area, panel in _pale_panels(view):
        px, py, pw, ph = panel
        if pw > vw * 0.92 and ph > vh * 0.92:
            continue
        if _band_above(view, panel) is None:
            continue
        picks = _pictures_in(view, panel)
        if not (CHOICE_ROW[0] <= len(picks) <= CHOICE_ROW[1]):
            continue
        rows = group_rows(picks)
        if len(rows) != 1:
            continue
        return [(b[0] + ox, b[1] + oy, b[2], b[3]) for b in rows[0]]
    return None


def _pale_panels(view):
    hsv = cv2.cvtColor(view, cv2.COLOR_BGR2HSV)
    mask = cv2.inRange(hsv, CREAM_LO, CREAM_HI)
    mask = cv2.morphologyEx(mask, cv2.MORPH_OPEN, np.ones((3, 3), np.uint8))
    n, _l, st, _c = cv2.connectedComponentsWithStats(mask, 8)
    out = []
    for i in range(1, n):
        x, y, w, h, a = st[i]
        if w < 70 or h < 28 or a < w * h * 0.55:
            continue
        out.append((int(a), (int(x), int(y), int(w), int(h))))
    out.sort(reverse=True, key=lambda p: p[0])
    return out[:6]


def _band_above(view, panel, tol=70):
    bx, by, bw, bh = panel
    x0, x1 = bx + int(bw * 0.06), bx + int(bw * 0.94)
    k = int(clamp(bh * 0.12, 5, 20))
    if by < k + 3 or x1 - x0 < 20:
        return None
    ref = np.median([_row_colour(view, y, x0, x1) for y in range(by - k, by - 2)],
                    axis=0)
    cream = np.median(view[by + 2:by + min(bh, 12), x0:x1]
                      .reshape(-1, 3).astype(np.float32), axis=0)
    return None if np.linalg.norm(ref - cream) < tol else ref


def _pictures_in(view, panel, inset=4):
    x, y, w, h = panel
    x, y, w, h = x + inset, y + inset, w - 2 * inset, h - 2 * inset
    if w < 20 or h < 20:
        return []
    crop = view[y:y + h, x:x + w]
    bg = np.median(crop.reshape(-1, 3).astype(np.float32), axis=0)
    dist = np.linalg.norm(crop.astype(np.float32) - bg, axis=2)
    mask = (dist > 34).astype(np.uint8) * 255
    mask = cv2.morphologyEx(mask, cv2.MORPH_OPEN, np.ones((2, 2), np.uint8))
    mask = cv2.morphologyEx(mask, cv2.MORPH_CLOSE, np.ones((5, 5), np.uint8))
    n, _l, st, _c = cv2.connectedComponentsWithStats(mask, 8)
    out = []
    for i in range(1, n):
        bx, by, bw, bh, area = st[i]
        if area < max(60, w * h * 0.004) or bw < 9 or bh < 9:
            continue
        if bw > w * 0.6 or bh > h * 0.92:
            continue
        ratio = bw / float(bh)
        if not (CHOICE_ASPECT[0] <= ratio <= CHOICE_ASPECT[1]):
            continue
        out.append((int(x + bx), int(y + by), int(bw), int(bh), int(area)))
    return out


def identify(frame, template, boxes, threshold, lead=0.06):
    scored = []
    tw, th = template.size
    for box in boxes:
        bx, by, bw, bh = box[:4]
        guess = 0.5 * (bw / float(tw) + bh / float(th))
        guess = min(SCALE_MAX, max(SCALE_MIN, guess))
        best = None
        for factor in (1.0, 0.85, 1.18, 0.72, 1.4):
            size = min(SCALE_MAX, max(SCALE_MIN, guess * factor))
            sw, sh = tw * size, th * size
            got = _best_at(frame, template, size,
                           int(bx + bw / 2.0 - sw / 2.0),
                           int(by + bh / 2.0 - sh / 2.0),
                           slack=max(6, int(0.3 * max(bw, bh))))
            if got is not None and (best is None or got[0] > best):
                best = got[0]
        scored.append((best if best is not None else -1.0, box))
    scored.sort(key=lambda pair: -pair[0])
    if not scored or scored[0][0] < threshold:
        return None, scored
    if len(scored) > 1 and scored[0][0] - scored[1][0] < lead:
        return None, scored
    return scored[0][1], scored


BUILT_IN = "{\"bolt\": {\"click\": [20.5, 37.7], \"image\": \"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\", \"mask\": \"iVBORw0KGgoAAAANSUhEUgAAACoAAAAwCAAAAABC5bd6AAAAuklEQVRIDY3BCYKiUBBAsbz7H7oaXGYUEX+SZVmWZTk1dnmVM+Mp/+TTeJO7fBjvcpejcZSbHI0P2eVgfMouB+NENjkYZ0IOxqnIwTgVORinIgfjVOTdOBd5M76IvBhfRf4b30WexpXIw7gUuRnXQjbjlxDGL9mE8UN2YVzLTRiXchfGQW7GJk9hvMnD2OQpm/Eq57IbD/kuu3GXC9mNm1zJzdjkUpZlWZZlWZZlWZZlWZZlWZZlWZb9AT2HIjGBJqfuAAAAAElFTkSuQmCC\", \"prize_image\": \"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\", \"prize_mask\": \"iVBORw0KGgoAAAANSUhEUgAAACoAAAAwCAAAAABC5bd6AAAAxklEQVRIDY3BAZKDMBAEse7/P3rOkIuzsIayJNtkm2yTJiALchMO0slVOEknV+EknVyESW6kChdSSRGupJIi3EghRbiRQopwI4UUoZFJitDIJEVoZJIiNDJJERqZpAiNTFKERiYpQiOTFKGRSb7CikzyFVZkko+wJD9yCktSyCmsSCWHsCSVDGFFrmQIC3IjQ6hkTYZQyAMZwo88kSFM8kgO4UseySF8yAs5hYO8kVM4yBv5F5BXsk22yTbZJttkm2yTbbLtD6QfJDF2YtopAAAAAElFTkSuQmCC\"}, \"ghost\": {\"click\": [15.0, 47.5], \"image\": \"iVBORw0KGgoAAAANSUhEUgAAAB8AAAA2CAIAAACwb9WQAAAMsUlEQVRYCXXBa5CeZX0G8Ov63/fzvO+7x2x2ExIMkAYICoTEnDYZyWYDJCQQ0VoVxx5sq7XOaA86jmOn/abTTqvTdiwf2lq17ThaZ3BsFQVaUZRaThEJQSJhOAVy3Jyz+x6e577/V99NoOqH/n582/QaEACJ1xDzSLyOFEj0EcJF1OXXbzj+wjNVe44S5SAIECBEQgLhfNv0GhAECAoXkRBJEewDCNJIiCQIySUtX7Xx1X2PKScAlAwgQcgAUAQI8I7pN5MARECCcIEAMsTAYMGMwYykgfPQVzQai69Y+epTj8IdWUYRMIoCIRIQDOKuLavlAgSAAMwAGGkxhDLGRlm0imJooLFwqDkxGpuNXFXV+blYWTMOHn3mydyrVNfmgjshAgaRIEiIu7asxQWEQBgBoxUhlEVsloNLxi/ZdsPV11+5fOISBBByMTmOnDr5woGXTv7wyc6hGW+3VVVIbu6EDE6CICHu3LKWEAheZGaFxVbZHB1avGXV2pvWXzYxUcYi0iICQcEzlKBzvc6rJ049v/fZk/f9T31+Vp2u6hrZDW5CHwneNrWWxEWMIUYLzUZz4fCVv3bjhk1rFg4MEgiITRYBMTI4PCvXTB3V7Tqd7fZ+uu+ZQ994sDp2wrs91AnuBkAixN1b14HoMzNGi61i8NKJy3es37JtcrAsAy3AChQlihYbBWKF1FMvmypVPeXZuj7b6+zbs+/Vrz+YTp5St8fskghQzt3T60GBCCGwjM3hgWXvvHH65snhRqtAiAwRFmgFygYLOnuoM12A05Py+br34vHjjcGBJ77z4Mx/PeJzHVWJLigT4u5t60GZkTEWg62x9Ssnf3P78rEF0WIDMcIiooFGQIGQwUg65fIe6j379n/6s//4zvfdccWvXPbTr9w3t++A2l2kbMpGcfdNG0khMDYbrYnRlR+4dc2qlS2LpYUGYoEQGAwMEhifePpnf37X5webrc/82R8vmhgHbM/Tz3zyU389MjH6kU/83rFnn5/52n9qrsNeT57FzNu3rQNoRSgHmwvfct36d29dOragZGxYKBgLWIQZzQB6+NsvfuWeB76PEDauu+HTf/ghBR0+PvPBj38qm377Q3eOLRh98u++1j54JM+1VSdAfOeODRYZyqJcMLTsvbdcs3blSNkoLTQYChYRiAhGi7Sg8Bd//6UHHn6EtOGx0c/9yUcvvWQiJXzyLz+3d+/+t//WrnWb1v74y/eeeXw/Z+dYJyLz9qkbMoEYGssWb/7ouy95w8KRstGwWJoViBEWaFGMFgOKr91z/xfu/g822Bwe/Ks/+PDK5csyNNvp/uln7rr1jqkVK67c99CeI//+/frUeVU9pIpTG1aFsiiGByZWL9/0wdvjYDHRHGqaRYaCjAwRjGBgCAwnTpz9xN/cdersmZHRkc9+7MNLJhYmlxvO9To95LmU9u9//qUvfrM3c0bdHnLibdOTLGM50hrfsHL9b9zcRloyMNI0lhYiGGgFLcoizSwQPHjo6H0PPXzDtVdtXnUdaVlyqqvcyXUn5Z+98NKBz3+rOjyTe12kmrdt22iNshhpLXrLtevfM32y6i1pDQ0XZSQKMoLRLMoi+yzAjEYSkiAHXMjwSt7z3EnpuZcPHvine9qvHPVeV1Xirps2hmajGGpObH7jxl+/eabXLc0uG1oQgYKMgBERFmgGRJrRSEIUJMClDFXuNbyX8oGXDj73hXtmXzmaO12mxF03bbIyFkPNsbUrpt5/+5GqXXsaKVqXtoaLwAgEwMBIMyCAxnkGE9wBgS5VUpLX7vufe/m5L3177tARn+siZd62bZOVIQw2Rq65bOvv7z5TeDvXJBohjJbNhbHRsggwkAEKMCNBURTmCcxAdiWoznnfk8+++NXvtg/P5G4HKXHXtk0WLbTK1uLRDR+6Y+yKRUd7s6IbZWAAGxZaMS6Mg4OxIGQghXmEAIkis8uhqs57vvf4y998qHv8VOp25Zm7picRrGgWcaS1dHrNjb+6dQbdudQlBQiSUQCMHI3NxeVQacFgkAABlAAiCyI77d4P/vU7p/b8rHfmvHqV5Nw5vZHGUITQajQnRq9/703XrrvmYPdsUiYcEiAQBljggJVLy5FIQqAAUKQIdwk8cfT0j/7hG+2XD6e5jlc13Llz20YC1lfEYqQ1tHzJm97+ljddf+VMas+mDuCEBBgdFIDxYmhBaJlIECIoARmokl546vl9/3JvdeJM6naVMgDunN6APiJEC41GMTo4eOnCy6dWXzd5bdGKJ+rZWhlwwkmRaFlzURwwGIQLaAynem3U3POth45978n67HnvVXIHwJ3T6yGQAGkxhGZZDDVbi8cmVq9YsW7lsuWXdC2fTbMGkCJQMC4qhk2ACICkw460Z6ujc0989b7ZA6+kdjf3akiEuHN6PSEIIEDSGMoiDjYbwwOtxQsu3bpq7YZr42A4meYIJxhh43HIwD5ISf7iuTNvaEw8+u2HDt7/SH12znuVcoYcAHdOr4PQRwEECRqtKGOrKIdazUWjE+tXTm7fMDBUnPMu4A2LQywNBK32fGj2fDM0Tx88/ZN/vqd7aCa1q1TV9KwLuHP6zRAI4hfQaDGEZlm0ms3xkaVbV03vmsxFrpQGWQQGo3Vyfbw7ZwqjufW9L3/n+CNPp9kOqgSQSmYWgnHX9FrMIwUBggChj2Swotlojg0PLxtf9zu3XnX5olqpZABZuR/rnAdsHEOP3f/wiYd+Up087b06V7XnnN1Tlifnzq3rBEEiIAFwAmbsK2IoWmVzbLgxPrziXds2rrmSlIG1dKyadfcRH3zqR0/v//oDczOnuufaVadOKeUsl1ykwN3b3hwDSRBG0OUQ3D25JKAM5chga/HYqvdtv3H1G0XveHWsd760YjQNPPrfe39894PnXjmR2l3ViZKZBQshIIY+8q3TNyRXnVTXua5zyqlOnqXsCBaKgXJwfHTpDcvf8dE7l44Pd7w6m7pNK8Ocvn/voy8/uLc6O+udWlUFJQIgCAMyAAJc9carIYEgQMKMIVgIDCGUzbI5NDC8ZGzd+3dMrbu+jV7l2WscPHDs6e8+fub5w73T51O3p6rOOVMu9AkA+2AguGXjOhqNZkbCaSRBgsaiWTSGB666df3uO29OzO46/OrJZx7+6ZF9L7SPna5nO6lbecqQIEEOQJhHAiAJ7pjaBAmkRMBBEaDBgsVGOfSG8dv/6F1XrVjSyf7EY/uf/cHe9pGTvdl2dX7Oe3VOWS4jAEqOeQREACQB3nLjRoAASAIgBUD0WMTYKMdXr/jdj90ZW+XeA688/tUH2jOnU6eq57qp2/WUBVAgAEEQAGIeQRB9vOXGSVxEEABJCITFEJvF+HXL3//x95St5v0PPvH8vY9Wc11Gzh47nTpdZYcAEnICggjiAoKEAHD7lkn8AhIgBFhgLIuBRQt2f+Qdq1ddvee5l566/7FWq9EcaT1x9w9Su6ecISco9BEQIIIEBBAgwO1TGwVCIAGIIAhJNIQilgPNpatW3PGBty8dHzt+5szRozM//LcHju5/MXcqCFAGCREQBJBEn0ASgMRbpiYhEQDxGpKAKLMQCouNcmTZosuuuSKndGj/y2cPn8hVT8klJ4h5BIR5BEAAxAXk9i2T6CMuIiFhHgWRwRhgRTSL6st1Thk5S4JkJEEABAQBhAQSr+P2qU0AJJFEH5Wzd1Pq1LUAEgBhJCAJcHcQDgEwUkYWFgbLGM0AEvNIiiDA7VObAAECea7T7eXsEvEaASR+TgIEECAgQIAIAoxm0WyoURTBIIAEwO1TmwB0U32u2wUggPg54QKBBCBAeI0BIoQLBBIgQGKk0WyVEUIft2/dXKV0utMmIICYJ4CYJwggAQGEAOE1BEAAFEBCAAUQ8yYGBmIAQG6fmmzX1Wyvwi8T5hG/SIDwOuIC4gIjJMwjMD4wUAQC4I6tmyQ/1e4kd/wy4SKHDHTi/0EQBgggoVZRjDQbxDzumN5MSMJs1WtXtSAI/0focwgXkXgd8XMkBMBoQ42iFY0EAfbtmN5MCIAkh+qUO3Wqc84SXic58RpSAiGAgGBEGUIRrVWEwbIgBBKQBAm8dXqzAIKQQEiAXBCEyr1X1y6XKymDMtKIRgg0K2OIRKsMcGQp5ZxSTinXOeXkOStl5y1TmwEJIECIEM0IkQhmRgFyyXNyrwm44NlT9jp7qnNyz8lTltwdJAjAjGbRjHzrLZuMCEZSgNw9ZXn27Dl5TimnWimlnJWTu7I7BLmMJEAzmAWzGKOFEMxoFswQzIzkrqk1yT2nnLPX2evcJ89yCC6HIBiDXRAMZggWzKIFmhE0MpBGvI64QBD+F/vBmNdnNRNiAAAAAElFTkSuQmCC\", \"mask\": \"iVBORw0KGgoAAAANSUhEUgAAAB8AAAA2CAAAAAAaZh0bAAAAwUlEQVQ4EX3BAUKDQBAAscz/H71SWgEtXJK1rGUtF+MthxzGKR/5GH9ll7fxT3bZjS95yW58yyYv40Y22YxbIZtxK4RxL4TxIMJ4EGE8iDAeRBgPIowHEcaDCONBhHEvZDNuhbyMOyEv407IbnzLJh/jv2zya/yVl5zGRXY5jYvschqnvOU0DvnIafzKr5zGRw45jbecchhvuchh7HKVw9jlKoexy1VO4yVXuRjkj6xlLWtZy1rWspa1rGUta1nL2g/z1CY3yxcePgAAAABJRU5ErkJggg==\"}, \"green potion\": {\"click\": [15.5, 50.0], \"image\": \"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\", \"mask\": \"iVBORw0KGgoAAAANSUhEUgAAAB8AAAA2CAAAAAAaZh0bAAAAxElEQVQ4EX3BAWKbQBAAMc3/H721udiQBk7KXvayl6ux5CNX4yNLrsZXDrkaXznkNC5yyNe4yJKvcZElX+MiS07jlCWnccqS0zhlyWmcsuQ0TllyGqcs+RoXWfIxrrLkx/gthyzjPznkMP7IW97GjbzkZdwKeRm3QhgPIowHEcaDCONBhPEgwngQYTyIMB5EXsa9yMu4FfI27oS8jRt5yWH8kbf8GL/lkI9xkR+5GEu+spe97GUve9nLXvayl73sZS97/wDfxyY3yfN8xAAAAABJRU5ErkJggg==\", \"prize_image\": \"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\", \"prize_mask\": \"iVBORw0KGgoAAAANSUhEUgAAABwAAAAtCAAAAACYQFRAAAAAp0lEQVQ4EX3BwQHDMAzEMHL/oa+u5TRyHwJkIAMZyECOsMhFSijSSAlFGimhSCNbOKSRLRzSyBYOaaSEIo2UUKSREoo0UkKRRkoo0sgWHvKSLfzIj2zhJQ/Zwkse8hU6OWQJFzlkCTcpsoSbFIHwR4pA+CNFIPyRIku4yCFLuMghS7jIIV+hk0O28JKHbOElD9nCIY0cYZNGBjKQgQxkIAMZyEAGMvgA/j4fLqIAl1sAAAAASUVORK5CYII=\"}, \"mushroom\": {\"click\": [33.5, 66.8], \"image\": \"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\", \"mask\": \"iVBORw0KGgoAAAANSUhEUgAAAEQAAABLCAAAAADsLn67AAABY0lEQVRYCa3BAZKDIAAEwZn/P3ovGpMDBIMW3bKALCALyKWA/CR9oSJXpCecyJh0hC4ZkJMwIn3SCFekRyrhB+mQUvhJzuRfmCIt+QqTpCEfYZrU5BBukIocwg1Skbdwi5RkF26SgmzCbfJPNuE++ZKX8IB8yUt4Qj4EwjNyEAjPyEEgPCRvAuEp2QmEF/kKs2QnEEAqYYrsBALSChNkJxCQVpggOxkIE2QnA2GGbGQgzJCN9IUpspGuME2QnjBPkI5wgyBn4Q5BTsItgpyEWwRphQ7ZhA5BGqFHduFMkFrokkNoCVILXfIRGoJUQp/8CyUBKYUBKYSCgJTCgFTCh7xIIYxILbzJRgphRBphIzsphBFpBJCDFMKIXJFCGJErUggjckUKYUSuSCGMyBUphT65JKXQJ5ekFjrkmjTCifwgJ6Emv0hP+JAJsoAsIAvIArKALCALyAKygCwgC8gCsoAs8AcaikhMKbHsqgAAAABJRU5ErkJggg==\"}, \"feather\": {\"click\": [21.0, 64.6], \"image\": \"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\", \"mask\": \"iVBORw0KGgoAAAANSUhEUgAAACsAAABHCAAAAAC7AOE9AAABGUlEQVRIDY3BiYGbABAAMU3/RW8Af+QMBin35b7cl1OD7OTMWGQnJ8YqOzk2NtnJsbHJTo6NTXZybDzkI8fGU95ybLzkJcfGW55ybHzkIcfGTjY5NvayyrHxnyxyZPyRRb6MAyF/jSMh/xsnInvjVGRnnIt8jB8iL+OnyMv4KfI0fos8jAuRh3EhshlXIqtxKbIaV0IW41LIYlwKWYwrWYRxKYswLmURxpWswriSVRgXsgnjQjZh/JaHMH7KUxg/5SmL8UNeshjn8pbFOJWPrMa3/JXV+JIvWY0v+ZLN+Ctfshl/5Usexn9yIA9jJ4fyNN5yLE/jJSfyMjY5lbexyql8DHIu9+W+3Jf7cl/uy325L/flvtyX+/4B+Io4SI+XxBYAAAAASUVORK5CYII=\"}, \"fang\": {\"click\": [19.5, 41.1], \"image\": \"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\", \"mask\": \"iVBORw0KGgoAAAANSUhEUgAAACgAAAAuCAAAAAB/zASsAAAAvElEQVQ4EY3BAYICIQwEwe7/P3rcxVMJwZMqOSSH5JD8EC6C/Cc8CfJdeBHkm/AmIHthIiBbYSIX2QiFXKQLhdykCYUMsgqFPMkiFPJHqlDIixShkDeZhUI+ZBIq+ZBb2JGJ3MKGzOQWNmQmQ2ikkCE0UsgQGilkCI0UMoRGCrmFDZnJJezITC5hR2ZyCTsyEwgbUgmEThYCoZGVQFhJIxBW0kgRBmmkCjdpZBVAGjkkh+SQHJJDckgOyaEHrMQjLzM/xGIAAAAASUVORK5CYII=\"}, \"purple potion\": {\"click\": [19.0, 45.6], \"image\": \"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\", \"mask\": \"iVBORw0KGgoAAAANSUhEUgAAACcAAAAzCAAAAAAxj05kAAAAzklEQVQ4EY3BgYGiABAAsUz/Re+DAno+IknuyT25J5/GKn/l3XiTN3kz/sohL+NDDnkZH3LIy/iQQ17GhxxyGP/JLrtxIpvsxolsshsnsslunMgmu3Eim+zGiWyyGyeyyWacylOexhd5yMP4Kqs8jK+yympcyCKLcSlkMS6FLMalkMW4FML4IcL4IcL4IcL4IbIYl0IW41LIYlwKWY0LWWQ1LmSRh/FVVnkaX+Qhm3EqT9mNE9nkZXzIIX+MQ97lntyTe3JP7sk9uSf35J5/RtUnNPcUlloAAAAASUVORK5CYII=\"}, \"wax seal\": {\"click\": [18.5, 44.4], \"image\": \"iVBORw0KGgoAAAANSUhEUgAAACYAAAAyCAIAAAC/GD50AAAOF0lEQVRYCU3B2ZOcV30G4Pc953xLL7Nr3yxL8iIUDLaJekYjCUwI5iJFJRdJKqlc5CqVu9zkr0oqlVSKSioBAlUYU1CUCVDEWMgLlqWRRpqe6e7p7m855/emZyAmz8Pbg01AOCZAwgKJ3xEEkBAIicQRQRAEkiABASTxG5II4v+QEI4Q4sL2YIAFYUEgIIKgiCMCKRgEEgKJI4IgLEgkQRAQSOJTkhwgkAAo4QhJCLy1uQmJIogjxAIlHBNIQBAAiSQkEBBEACQkkgBICARBABKOERAAgiIISQS4PdjEMRIQBICARFAAAWFBJiFap4m9JDOLDlUIde6d945YcKRwjCAoCQABEADxWwLE25tbAigBEEAAghwAUQCdzGCpU7WfNX+1UTGr2jbOHaZ5tleGBxmf5A55IB0XQFAQSXxKEEkcESTe3twSAAkEJOK3HKljdVW/3Ohree+6QjudzdsmJUsCSXk/ytz7gT9nfJJh7p0F74P39CA+JYBYEI6Id7Y2AUgQRAACCUnOwLY9WafrlW354mKWV00zqutZSlW0BFH0RCf4LMumxC7SUBpDexk+yf20XyJ44ogA4lPknc1NEZSwIAiAzOrmcos3b3z2C1euxm+/VT8bjlPcmVdP6maSYpUsQiY5oON8P4TVEFazsOSDo4sOj5y+4ZrhqRUfPARBFEiAzLzjna0tCIBISCLgx7Ov9tf+8s//dOPSxeG//9fj737v/mR27/BwP7ZzU6ScY04GOoLRUjQzoEt/MssuluVGlge6e9b8w7ILJ1e8d96RAkgiSca7W1sAJBGA5Gb1m8n9xd0vdi+eH/3wnd3/ufeTZ8MPZ7PKLPNuNcs2QtYP3pOSkilKDjpMaRjjKEYC58victGpzf7Ntx+fX5YPMbWxTU3btDHJEu9ubUL4jdi2F4bTvztx7vm1tf1Hj2az6iejyTsHB97500X+fFGuhBC9m5VFBNrZdLeqx237e2WvTzbSMMYHTTWO6XSeXynLD2L9H1nc7YYES2YJSDE6kHdvbUEAhIXp/PNPJn975VqXnD4bfjidfefp3kz2crd3pShA7HfK/OyplbzIhqN7jx+/Mxk/F4rLee5hgCPQmB60zeOmOpcXVzrlO231TdcMS7iF4IosFCHwzq1NGgRRKR9Nbz+Zfv3MmSUfpvP6+8PhL6bTFzvdF4pOAw373dMvv3BqVumTnfcn4++O9q9k5fN5EXXEAJMcERN3YvOorV7p98/lxTvW/GjF1WtdnwV6R4BfvT3wHt47WsqfTLYfTV9dXzvTKfen1X8+22ukW/3lJNsN/tTnbjwnjt69lwnfHu/vNc2tzlJtSSAJASYCoBihj+q5OWytrATgh2x/sF5U/Y5ESfyjN7aa2DZNrOum/3T25VF8fnnpM8v9p4fzb+0NLxTl5aIYxTjeWB3cuF69e78aT4rgv7G/lxk+U3ZqM4CGIxQpEDBgZvZBM/vcyvLZEPZS+sc8PT6zCu9k4vUXrsWYTEqmlWn7hxXvntp4Pi8eTmZvHRxcK7sbWXgS23Jt5fX1k7s7j2VWevfW5GC3al/rLNVKEgR4EiAEAQSS9EEzO9ctXyw7+037z3H28ZVTCoEAL5w7L4EOBLq1/QnKv7pwLk3rp9P526PR5bK7FvzD2G70e5/r9HcOD0k46mkTf3A4PolwOitMIkRwAaLBBJi0E9ulwt/o9nbm83+dTx69fAFZIMUbL7wY8pB5T8+1w/pv2NteW/l4d9hE+9FotBTyM1m208aiV24tLT8ZH4IwGcGx9OPpoWvThgtdOg9GqAKGimZ2KRSPY7taZi93uz8fjb7VTPc+85zLAyR+aXMggoLVzbVn07///Otnr1z86Dtvz4aj96v5kzpeKzsji/vE7RMnfdOOm1pQggjCuQNpElMiEXzwviP8bH84q9obeW8n1VeXeye8f/tg//tMs5fOMwsAeOfmTeFIXrevPxr99UsvP/+VOwe/eO/jH//0oGrem057Pus5txOb9aXeq0srVVVN2yaaBEDKnA/euzwz7+qmfX88fudw/LwvV1w2dem11ZU6tj84OHirZHvlLLwDwNuDAQBJ3brd/OTga93+iYvn0e1OHjycTucHbfz1fL7sM0GPY7vaLZ/rdFwbZeZAAGaWpNY0SfGjurrXVJdC52IoDpGurPTPhnBQ1/89nX6z55qLJ+gcAW7fvIkj7M2bWw8P/qDTH8b4/nxeBL/i/HqWgXxaNQCi6XHTzJ0VZEHmpCMl1aZJivsptoYzvjjhMnqe7JdXO5292ZzQvbr6lz7nZ9adcwC4PbgJkEBeN5//ZPTHeffXdf3TyWQ/xcLxbFHeWlvvhPB4Oq+iNcnGKU5SPLRUwaIWQCIH+y4sM5TOF7k/3S1OZtlhU89j6mfhvWr+T33OTq3ROQDcHmwCAmCtbQwnfzZTjOlnh4dPY9tKgXi11//y2TOl84/Gh9MYq2jzpNZSY4qSGQjLwBB8mYeVIl/JPM3aZJJIVpZ+ND383ules7YMAgS3B5uAAEhg017aO7yyP5/WzTOLjYnA2RBeW155/dTJ0rkHo3GdUmuqzaJJAJwr+t1Ov+eaJh3OLKVWlkxRaqWDFId1827QxxfX1SsJiuL2YBP/j6qmu3uwcTBTG2EWwBM+nArh5eWlayur+7NZNOt0ymQW22hmrsiz0yemyR49evx0NJmk2MpgEmlSI7XE/RO9w1MrLnjSOZDbg00AJARCRjCm1E7nHE6Kw3q1bi/DrwZ/KSsK58YxFcGdO7lxdmWlXxTVbPruwye/PBx/VM32pAqIQA/s0WVk17mS7nGZPbm0wX5pPAZwe7AJiIQEEBABEhJUz9pqB2vT6XUbv1rkJf1u25ppoQihKPPHbfNwOttNsaJMIARglS4Hc7Lj3CTzj86uxhMrPjhJEBa4PdjEEYGkABKATAazUXc5Xh1OnxTj+7fc/BIxj0ZCgkl7KX4Y6waYS4IMMMIDfXpHZo5tke2t9+3kss8zSQAEEOD2YAAQEEgKAAEBmk/SKXtlNbuwO3l4MN/J5juX0uh0atagkvDgw9g+aOtDqAYan7W+0wZ0ln3wHg70zjoF+p0QPABJAIQj3B4MAJA0CTIAFNrGutXlC+Urh9Vk3s6iNZN6WFXPijRa4cGJyF5Ke231VDb33Zj1UujRd+VVnpqXPeccF8AjglEQKIkkAN4eDIQF4RjpIMNk+Wp5t21tOH2KIzKleTs5iJ+wN/SHS3nstKlqFI05XU4QQGCI+bh/JualIwgCImACCIISCIjbN2+SBEECAuHaihfw+6vh0s7oQRsbEgAX6nY2xK+K9Vm1F5bsudXumVkzaVMTrW5TE60W4OCs/2zplPPeAZKwQIKABEAAeWdzAAgCAZEWrajPXSlvjWfj0WwfEOkAATis9yfhg95Jm41TZ3bp8upnx9VBTBGEZElJEmDj9rEtP+2vOjoHCQThAEFHQPLu5gCAJBISrc7O4QtL/vzu+EGTWgAEceyg3qk7D3obLjZRz05e3/jipD44nI+dcxJAECRYp+l+/HXYOOj0ggNBQAAImAQu3BkMQAEEaW3K69NXirvj+ehgulunuYjV8mQTaykNm4+1vNtfy0xo9jo3lt9MMe2MPiGIYyQBEJjF8TDeXzrT5EUAQBImg0hC4N3Nm1ggIGc1e9XVvj+7M74/j6NWU1h4af1WMk2q/f32I7e2313KALQz/5y7u15c/HDvPUstwaQY1XgGTy9y2gxn4UHvRAoZQUICBJAAv7g1gARQUlshTbtwbXRz7xFyNpWK6aWr668+GT98XP2y2DjsdHMSllhOL15f+crD0YeT+T6ASb03Tbve0yl3yADWacbeePmEC5knBMBkEPjGrU0ck2TJkilkniBAwVLSZNheyrYCur8avR3Wp3kZAGhhWl4v3/Qsdg4+ruJ0WH+kpYO8cBBhhDlLMqbuEjvdzAGO8h4E+cbWAIRASAABCYqmZEoySSmZm66u29Xd+n679CzLAxwgKLq15tpLK7cfHny4M7k/zT4p1xWcJ+UcPOGJwvsiC3lGTweoNYsx8o1bNyWQBBDN2mStmWSCJBxjaqHDsqmbsNZkuQMIwAysihvdr03n43uj72cbs6LjA51zdBRAAFKiiWYxWt2ktk3RIr+0NSAgqLVUtTFJ+B0BBGBSai3WKZTeO8I5R5DOCcuzKz51nvifdpYIIabUpiNta9FSTCbJSRk9AcFk4pduDSABmrZNTAIgABKOCCAh5wmAAEEQIFJSiikmpTlTg5RNjZBkgo5A0BHICc654NkLoch8ERzf2B5Aktm0bVNKID1BwhN0BCApmbXJ2jbFFFOSSclgCxJEyQQTYCYQBD3pvQ/e5cFlWciCL3xY7WTBeYH8yvbNQDoiKdUxtpIdS2ZtTG2ytk1tbGOSCZJBACgcEUCQlCe9ZxZ88D4PwQfv6RboQIBkGTJPrwWQX//yVoyxaWLTtvOqrWKsYopmJplJkEytFowiKEcXHJ1z3vss+Dz44J33joAjBZeEYyTh4bx3mQvOE6QDQPCVF6/VbZtiEgEQksioaCaCIEj6jN5nmXfB++C89w6kQJBKEAyAgQTofOACgjsGeudwRCCxQPC58+clEwTJOxeCz4Mvs1AWIQ/BBw+RTqBLhmgSYEYThQUJcM4BcCQAko5wJEEQJAURBIQjFMBL58+XWeiUoch9EbIiCyF4OBKUKRkgJUCCQAkCBAIkAQkLxAJJARRAEMQC8RsEAQECKJDbX3itzAMpEjBEk1EyQXR0ggiIgCAIJASAOCKAOCKAgACQxDGSkgCSgEAAFEAA/wsFmeZSFh49ygAAAABJRU5ErkJggg==\", \"mask\": \"iVBORw0KGgoAAAANSUhEUgAAACYAAAAyCAAAAAAVEfb/AAAAzklEQVQ4EY3BAXKjQBAAMfX/Hz23xuCQC6aQ8kgeySP5bSz5X87GLr/lZJzkJB/jj+yyG1fyls34Ipu8jG+yycv4Ki95Gd9lyTJuRZZxK7KMW5Fl3Ios41ZkGbciy7gVYdyLMO5FGPciy7gVWcatyMu4E9mM70J245uQw7iWJYdxKS85jCvZZDf+yiFv40IOeRsXcshmXMkhm3Epu2zGLsaPvGUzdmF85C0/BmF85C0nQxgvOcnJkGUsOcmFQc7ySB7JI3kkj+SRPJJH8sg/G5crMwVMIOcAAAAASUVORK5CYII=\"}, \"green drop\": {\"click\": [27.5, 72.4], \"image\": \"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\", \"mask\": \"iVBORw0KGgoAAAANSUhEUgAAADgAAABQCAAAAABpSun1AAABLklEQVRYCZ3BAUICMRAAscz/H70eiAoK0l6Sk3JSTsquIWTTIGTPuIhsGVeRHeNTZMO4iawbNyHLxpeQVeNbyJpxJ2TJuBeyYDwKeWv8FvLG+CvkX+OZkNfGCyGvjJdCnhv/CHli/C/kj/FOyG/jrZBHY0HIg7Ei5M5YE/JjLAr5NlaFfBnLQm7GupCbsS7k09gQcjV2hFyNLZGLsSdyMfZEDmNT5DD2hBzGnhDGphDGphDGphDGphDGphDGphDGphDGnhzC2JNDGHtyCGNLLnIYO3KRw9iQqxzGhlzlYizLp1yMVbnJ1ViUm3waS/IlN2NBvuXLeCs/8m28kTv5Mf6Ve7kzXsujPBjP5bf8Nv7IX3li/MhzOSkn5aSclJNyUk7KSTkpJ+WknJSTctIH8qM/UZ92FB0AAAAASUVORK5CYII=\"}}"


def builtin_names():
    return set(json.loads(BUILT_IN))


def gather_templates():
    out = {t.name: t for t in load_templates(BUILT_IN)}
    try:
        names = sorted(os.listdir(ITEM_DIR))
    except OSError:
        names = []
    for entry in names:
        stem, ext = os.path.splitext(entry)
        if ext.lower() not in (".png", ".bmp", ".jpg", ".jpeg"):
            continue
        if stem.endswith("_mask") or stem.endswith("_template") or \
                stem.endswith("_prize"):
            continue
        try:
            image = cv2.imread(os.path.join(ITEM_DIR, entry), cv2.IMREAD_COLOR)
            if image is None:
                continue
            mask = None
            for suffix in ("_mask.png", "_mask.bmp"):
                path = os.path.join(ITEM_DIR, stem + suffix)
                if not os.path.exists(path):
                    continue
                mask = cv2.imread(path, cv2.IMREAD_GRAYSCALE)
                if mask is not None and mask.shape[:2] != image.shape[:2]:
                    log("items/%s%s is %dx%d but the picture is %dx%d; "
                        "ignoring it" % (stem, suffix, mask.shape[1],
                                         mask.shape[0], image.shape[1],
                                         image.shape[0]))
                    mask = None
                break
            if mask is None:
                mask = item_mask(image)
            if mask is None or np.count_nonzero(mask) < 60:
                log("items/%s: could not tell the item from its background"
                    % entry)
                continue
            ys, xs = np.nonzero(mask)
            click = ((xs.min() + xs.max()) / 2.0,
                     ys.max() + (ys.max() - ys.min()) * 0.12)
            out[stem] = Template(stem, image, mask, click)
            log("items/%s loaded" % entry, key="load:" + stem, every=0)
        except Exception as e:
            log("items/%s could not be read: %s" % (entry, e))
    for entry in names:
        stem, ext = os.path.splitext(entry)
        if not stem.endswith("_prize") or ext.lower() not in (".png", ".bmp",
                                                              ".jpg", ".jpeg"):
            continue
        item = out.get(stem[:-len("_prize")])
        if item is None:
            continue
        try:
            image = cv2.imread(os.path.join(ITEM_DIR, entry), cv2.IMREAD_COLOR)
            mask = item_mask(image) if image is not None else None
            if mask is None or np.count_nonzero(mask) < 40:
                log("items/%s: could not tell the picture from its "
                    "background" % entry)
                continue
            h, w = mask.shape
            item.prize = Template(item.name, image, mask, (w / 2.0, h / 2.0))
            log("items/%s loaded" % entry, key="load:" + stem, every=0)
        except Exception as e:
            log("items/%s could not be read: %s" % (entry, e))
    return [out[k] for k in sorted(out)]


def colour_print(bgr, mask):
    hsv = cv2.cvtColor(bgr, cv2.COLOR_BGR2HSV)
    hist = cv2.calcHist([hsv], [0, 1], mask, [18, 6], [0, 180, 0, 256])
    cv2.normalize(hist, hist, 1, 0, cv2.NORM_L1)
    return hist


def identify_by_colour(frame, template, boxes, least=0.5, lead=0.25):
    mine = template.colours()
    scored = []
    for box in boxes:
        x, y, w, h = box[:4]
        crop = frame[max(0, y - 3):y + h + 3, max(0, x - 3):x + w + 3]
        mask = body_mask(crop)
        if mask is None:
            scored.append((0.0, box))
            continue
        theirs = colour_print(crop, mask)
        same = 1.0 - cv2.compareHist(mine, theirs, cv2.HISTCMP_BHATTACHARYYA)
        scored.append((float(same), box))
    scored.sort(key=lambda pair: -pair[0])
    if not scored or scored[0][0] < least:
        return None, scored
    if len(scored) > 1 and scored[0][0] - scored[1][0] < lead:
        return None, scored
    return scored[0][1], scored


def body_mask(bgr, tol=34):
    if bgr is None or bgr.size == 0 or min(bgr.shape[:2]) < 8:
        return None
    border = np.concatenate([bgr[0], bgr[-1], bgr[:, 0], bgr[:, -1]])
    bg = np.median(border.astype(np.float32), axis=0)
    px = bgr.astype(np.float32)
    dist = np.linalg.norm(px - bg, axis=2)
    unit = bg / (np.linalg.norm(bg) + 1e-6)
    along = (px * unit).sum(axis=2)
    across = np.linalg.norm(px - along[..., None] * unit, axis=2)
    ratio = along / (np.linalg.norm(bg) + 1e-6)
    shadow = (across < 26) & (ratio > 0.60) & (ratio < 0.97)
    body = ((dist > tol) & ~shadow).astype(np.uint8) * 255
    k = np.ones((3, 3), np.uint8)
    body = cv2.morphologyEx(body, cv2.MORPH_OPEN, k)
    body = cv2.morphologyEx(body, cv2.MORPH_CLOSE, k, iterations=2)
    n, lab, st, _c = cv2.connectedComponentsWithStats(body, 8)
    if n < 2:
        return None
    keep = max(range(1, n), key=lambda i: st[i][4])
    mask = np.where(lab == keep, 255, 0).astype(np.uint8)
    h, w = mask.shape
    flood = mask.copy()
    cv2.floodFill(flood, np.zeros((h + 2, w + 2), np.uint8), (0, 0), 255)
    return mask | cv2.bitwise_not(flood)


def item_mask(bgr):
    if bgr is None or bgr.size == 0 or min(bgr.shape[:2]) < 8:
        return None
    hsv = cv2.cvtColor(bgr, cv2.COLOR_BGR2HSV)
    edge = np.concatenate([hsv[0], hsv[-1], hsv[:, 0], hsv[:, -1]])
    floor_sat = float(np.percentile(edge[:, 1], 75))
    base = ((hsv[..., 1] > max(70.0, floor_sat + 30)) &
            (hsv[..., 2] > 45)).astype(np.uint8) * 255
    body = body_mask(bgr)
    if body is not None:
        base = base | body
    base = cv2.morphologyEx(base, cv2.MORPH_CLOSE, np.ones((3, 3), np.uint8))
    side = max(3, int(round(min(bgr.shape[:2]) * 0.13)) | 1)
    brush = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (side, side))
    core = cv2.morphologyEx(base, cv2.MORPH_OPEN, brush)
    n, lab, st, _c = cv2.connectedComponentsWithStats(core, 8)
    if n < 2:
        return body
    keep = max(range(1, n), key=lambda i: st[i][4])
    core = np.where(lab == keep, 255, 0).astype(np.uint8)
    small = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (3, 3))
    mask = cv2.bitwise_and(base, cv2.dilate(core, small, iterations=max(1, side // 4))) | core
    h, w = mask.shape
    flood = mask.copy()
    cv2.floodFill(flood, np.zeros((h + 2, w + 2), np.uint8), (0, 0), 255)
    mask = mask | cv2.bitwise_not(flood)
    return mask if np.count_nonzero(mask) >= 60 else body


def prepare_item(bgr, pad=3):
    mask = item_mask(bgr)
    if mask is None or np.count_nonzero(mask) < 60:
        return None
    ys, xs = np.nonzero(mask)
    x0, x1, y0, y1 = xs.min(), xs.max(), ys.min(), ys.max()
    h, w = mask.shape
    cx0, cy0 = max(0, x0 - pad), max(0, y0 - pad)
    cx1 = min(w, x1 + pad + 1)
    cy1 = min(h, y1 + int((y1 - y0) * 0.18) + pad + 1)
    image = np.ascontiguousarray(bgr[cy0:cy1, cx0:cx1])
    mask = np.ascontiguousarray(mask[cy0:cy1, cx0:cx1])
    ys, xs = np.nonzero(mask)
    click = ((xs.min() + xs.max()) / 2.0,
             ys.max() + (ys.max() - ys.min()) * 0.12)
    return image, mask, click


class Chase:
    def __init__(self, hit, started=None):
        self.hit = hit
        self.name = hit.name
        self.started = time.monotonic() if started is None else started
        self.last_click = 0.0
        self.clicks = 0
        self.lost_for = 0

    @property
    def age(self):
        return time.monotonic() - self.started

    def due(self, every):
        return time.monotonic() - self.last_click >= every

    def clicked(self):
        self.last_click = time.monotonic()
        self.clicks += 1


class Attempts:
    def __init__(self):
        self.rows = []

    def _row(self, point, radius):
        for row in self.rows:
            if abs(row[0][0] - point[0]) <= radius and \
                    abs(row[0][1] - point[1]) <= radius:
                return row
        return None

    def begin(self, point, radius=SHELVE_RADIUS):
        now = time.monotonic()
        self.rows = [r for r in self.rows if now - r[1] < SHELVE_FOR]
        row = self._row(point, radius)
        if row is None:
            self.rows.append([point, now])
            return now
        row[0] = point
        return row[1]

    def forget(self, point, radius=SHELVE_RADIUS):
        self.rows = [r for r in self.rows
                     if not (abs(r[0][0] - point[0]) <= radius
                             and abs(r[0][1] - point[1]) <= radius)]

    def clear(self):
        self.rows = []


class Shelf:
    def __init__(self):
        self.spots = []

    def add(self, point):
        self.spots.append((point, time.monotonic()))

    def holds(self, point, radius=SHELVE_RADIUS):
        now = time.monotonic()
        self.spots = [(p, t) for p, t in self.spots if now - t < SHELVE_FOR]
        for (px, py), _t in self.spots:
            if abs(px - point[0]) <= radius and abs(py - point[1]) <= radius:
                return True
        return False

    def drop(self, point, radius=SHELVE_RADIUS):
        self.spots = [(p, t) for p, t in self.spots
                      if not (abs(p[0] - point[0]) <= radius
                              and abs(p[1] - point[1]) <= radius)]

    def clear(self):
        self.spots = []

    def __len__(self):
        self.holds((-9999, -9999))
        return len(self.spots)


SPAWN_PATH = os.path.join(_HERE, "spawn_spots.json")
SPAWN_KEEP = 400
SPAWN_RADIUS = 0.055
SPAWN_MIN = 3
SPAWN_QUIET = 20.0
HOME_CLICKS = (0.6, 2.0)


class SpawnMap:
    def __init__(self, path=SPAWN_PATH):
        self.path = path
        self.spots = {}
        self.present = []
        self.scans = 0
        self.dirty = False
        self.version = 0
        self._homes = {}
        self.load()

    def load(self):
        try:
            with open(self.path, "r", encoding="utf-8") as f:
                data = json.load(f)
            if isinstance(data, dict):
                for name, pts in data.items():
                    good = [[float(p[0]), float(p[1]), float(p[2])]
                            for p in pts if isinstance(p, list) and len(p) >= 3]
                    name = name.split("/")[-1]
                    self.spots.setdefault(name, []).extend(good)
                for name, pts in self.spots.items():
                    pts.sort(key=lambda p: p[2])
                    self.spots[name] = pts[-SPAWN_KEEP:]
        except FileNotFoundError:
            pass
        except Exception as e:
            log("Spawn spots unreadable, starting fresh: %s" % e)

    def save(self):
        if not self.dirty:
            return
        self.dirty = False
        try:
            tmp = self.path + ".tmp"
            with open(tmp, "w", encoding="utf-8") as f:
                json.dump(self.spots, f)
            os.replace(tmp, self.path)
        except Exception as e:
            log("Could not save spawn spots: %s" % e, key="spawnsave", every=60)

    def _key(self, name):
        return name

    def clear(self):
        self.spots = {}
        self.present = []
        self.version += 1
        self.dirty = True
        self.save()

    def caught(self):
        self.present = [p for p in self.present if p[4] == self.scans]

    @staticmethod
    def _close(ax, ay, bx, by):
        return (ax - bx) ** 2 + (ay - by) ** 2 <= SPAWN_RADIUS ** 2

    def observe(self, hits, size):
        if not size or size[0] <= 0 or size[1] <= 0:
            return 0
        w, h = float(size[0]), float(size[1])
        now = time.time()
        self.present = [p for p in self.present if now - p[3] < SPAWN_QUIET]
        self.scans += 1
        new = 0
        for hit in hits:
            fx, fy = hit.click[0] / w, hit.click[1] / h
            known = None
            for p in self.present:
                if p[0] == hit.name and self._close(p[1], p[2], fx, fy):
                    known = p
                    break
            if known is not None:
                known[1:5] = [fx, fy, now, self.scans]
                continue
            self.present.append([hit.name, fx, fy, now, self.scans])
            pts = self.spots.setdefault(self._key(hit.name), [])
            pts.append([round(fx, 4), round(fy, 4), round(now, 1)])
            del pts[:-SPAWN_KEEP]
            self.version += 1
            self.dirty = True
            new += 1
        return new

    def count(self, name):
        return len(self.spots.get(self._key(name), []))

    def home(self, name):
        key = self._key(name)
        cached = self._homes.get(key)
        if cached is not None and cached[0] == self.version:
            return cached[1]
        answer = self._find_home(key)
        self._homes[key] = (self.version, answer)
        return answer

    def _find_home(self, name):
        pts = self.spots.get(name) or []
        best, best_n = None, 0
        for ax, ay, _t in pts:
            near = [(bx, by) for bx, by, _u in pts if self._close(ax, ay, bx, by)]
            if len(near) > best_n:
                best_n = len(near)
                best = (sum(p[0] for p in near) / len(near),
                        sum(p[1] for p in near) / len(near))
        if best is None:
            return None
        return best[0], best[1], best_n


PING_GOOD, PING_SLOW, PING_BAD = "good", "slow", "bad"
PING_WORDS = {PING_GOOD: "Good", PING_SLOW: "Slow", PING_BAD: "Lagging"}


class Ping:
    def __init__(self, level, digits, hue, box):
        self.level = level
        self.digits = digits
        self.hue = hue
        self.box = box

    @property
    def range_text(self):
        return {1: "under 10 ms", 2: "under 100 ms", 3: "100\u2013999 ms"}.get(
            self.digits, "1 s or more" if self.digits >= 4 else "?")

    def __str__(self):
        return "%s (%s)" % (PING_WORDS[self.level], self.range_text)


def _ink_runs(cols, gap):
    runs, x, n = [], 0, len(cols)
    while x < n:
        if not cols[x]:
            x += 1
            continue
        start = x
        while x < n and cols[x]:
            x += 1
        if runs and start - runs[-1][1] < gap:
            runs[-1] = (runs[-1][0], x)
        else:
            runs.append((start, x))
    return runs


def _hue_mean(hues):
    angle = hues.astype(np.float64) * (np.pi / 90.0)
    mean = np.arctan2(np.sin(angle).mean(), np.cos(angle).mean())
    return float((mean * 90.0 / np.pi) % 180.0)


def _split_colour(a, b, tint, least):
    t = tint[a:b]
    if b - a < least * 2 or not (t > 0).any() or not (t < 0).any():
        return [(a, b)]
    best, cut = 0.0, None
    for k in range(int(least), b - a - int(least) + 1):
        left, right = t[:k], t[k:]
        score = max(-left.mean() + right.mean(), left.mean() - right.mean())
        if score > best:
            best, cut = score, k
    if cut is None or best < 1.2:
        return [(a, b)]
    return [(a, a + cut), (a + cut, b)]


def _label_line(hsv):
    white = (hsv[..., 2] > 170) & (hsv[..., 1] < 80)
    rows = np.nonzero(white.sum(axis=1) >= 6)[0]
    if not len(rows):
        return None
    band = [rows[0]]
    for r in rows[1:]:
        if r - band[-1] > 2:
            break
        band.append(r)
    y0, y1 = int(band[0]), int(band[-1]) + 1
    tall = y1 - y0
    if not (3 <= tall <= 30):
        return None
    cols = np.nonzero(white[y0:y1].any(axis=0))[0]
    x0 = max(0, int(cols[0]) - tall)
    x1 = min(hsv.shape[1], int(cols[-1]) + tall * 5)
    ya, yb = max(0, y0 - 2), min(hsv.shape[0], y1 + 2)
    return x0, ya, x1 - x0, yb - ya


def read_ping(frame, depth=0.12):
    if frame is None or frame.size == 0:
        return None
    h, w = frame.shape[:2]
    top = frame[:max(12, int(h * depth))]
    hsv = cv2.cvtColor(top, cv2.COLOR_BGR2HSV)
    navy = cv2.inRange(hsv, (95, 150, 55), (130, 255, 160))
    shut = cv2.morphologyEx(navy, cv2.MORPH_CLOSE, np.ones((5, 9), np.uint8))
    n, _l, st, _c = cv2.connectedComponentsWithStats(shut, 8)
    best = None
    for i in range(1, n):
        x, y, bw, bh, area = st[i]
        if bw < 60 or bh < 5:
            continue
        cover = (shut[y:y + bh, x:x + bw] > 0).sum(axis=1)
        full = np.nonzero(cover >= bw * 0.6)[0]
        if len(full) < 5:
            continue
        y0b, y1b = y + int(full[0]), y + int(full[-1]) + 1
        if y1b - y0b > 44:
            continue
        if best is None or bw > best[2]:
            best = (int(x), y0b, int(bw), y1b - y0b)
    if best is None:
        best = _label_line(hsv)
        if best is None:
            return None
        bx, by, bw, bh = best
        inside = np.ones((bh, bw), bool)
    else:
        bx, by, bw, bh = best
        inside = navy[by:by + bh, bx:bx + bw] > 0
    box = top[by:by + bh, bx:bx + bw]
    boxhsv = hsv[by:by + bh, bx:bx + bw]
    if not inside.any():
        return None
    ground = np.median(box[inside].reshape(-1, 3).astype(np.float32), axis=0)
    inked = np.linalg.norm(box.astype(np.float32) - ground, axis=2) > 60
    inked[inked.mean(axis=1) > 0.7, :] = False
    inked[:, inked.mean(axis=0) > 0.8] = False
    rows = np.nonzero(inked.sum(axis=1) >= 3)[0]
    if not len(rows):
        return None
    runs, start = [], rows[0]
    for a0, a1 in zip(rows, rows[1:]):
        if a1 - a0 > 2:
            runs.append((start, a0 + 1))
            start = a1
    runs.append((start, rows[-1] + 1))
    y0, y1 = max(runs, key=lambda r: r[1] - r[0])
    if not (3 <= y1 - y0 <= 30):
        return None
    top, hsv = box, boxhsv
    ink = inked[y0:y1]
    sat = hsv[y0:y1, :, 1]
    hue = hsv[y0:y1, :, 0]
    tall = y1 - y0
    tokens = []
    tinted = (ink & (sat > 90)).sum(axis=0)
    plain = (ink & (sat <= 90)).sum(axis=0)
    tint = np.where(tinted > plain, 1, np.where(plain > 0, -1, 0))
    pieces = []
    for a, b in _ink_runs(ink.any(axis=0), gap=max(3, int(round(tall * 0.5)))):
        pieces += _split_colour(a, b, tint, least=max(3, tall * 0.8))
    for a, b in pieces:
        m = ink[:, a:b]
        if b - a < max(2, tall * 0.55) and m.sum() < tall * 1.5:
            continue
        s = sat[:, a:b][m]
        coloured = float(np.mean(s > 90)) > 0.5
        hs = hue[:, a:b][m & (sat[:, a:b] > 90)]
        tokens.append({"x": a, "w": b - a, "value": coloured,
                       "hue": _hue_mean(hs) if len(hs) else None})
    pairs = []
    i = 0
    while i + 1 < len(tokens):
        if not tokens[i]["value"] and tokens[i + 1]["value"]:
            pairs.append((tokens[i], tokens[i + 1]))
            i += 2
        else:
            i += 1
    if len(pairs) < 3:
        return None
    label, value = pairs[2]
    advance = (label["w"] + 1) / 3.85
    if advance < 2:
        return None
    chars = int(round((value["w"] + 1) / advance))
    digits = chars - 2
    if digits < 1:
        return None
    hue_v = value["hue"]
    if hue_v is None:
        level = PING_GOOD if digits <= 2 else PING_SLOW
    elif 38 <= hue_v <= 95:
        level = PING_GOOD
    elif 18 <= hue_v < 38:
        level = PING_SLOW
    elif hue_v < 18 or hue_v > 160:
        level = PING_BAD
    else:
        level = PING_GOOD if digits <= 2 else PING_SLOW
    if digits >= 4:
        level = PING_BAD
    elif digits == 3 and level == PING_GOOD:
        level = PING_SLOW
    return Ping(level, digits, hue_v,
                (bx + value["x"], by + int(y0), value["w"], int(y1 - y0)))


PROFILE_URL = "https://www.boombang.tv/en/profile/%s"
PROFILE_PATH = os.path.join(_HERE, "profile.json")
AVATAR_PATH = os.path.join(_HERE, "profile_avatar.png")
PROFILE_REFRESH = 24 * 3600


def page_lines(page):
    import html as _html
    import re as _re
    page = _re.sub(r"(?is)<(script|style|noscript|svg)\b.*?</\1>", " ", page)
    page = _re.sub(r"(?s)<!--.*?-->", " ", page)
    text = _html.unescape(_re.sub(r"<[^>]+>", "\n", page))
    lines = [_re.sub(r"\s+", " ", l).strip() for l in text.split("\n")]
    return [l for l in lines if l]


def _count(value):
    import re as _re
    digits = _re.sub(r"[^\d]", "", value or "")
    return int(digits) if digits else None


def profile_stats(page, name):
    import re as _re
    lines = page_lines(page)
    text = "\n".join(lines)
    stats = {}
    labels = ("vip", "achievement", "on the island", "ring season", "minigames",
              "back to", "gestures", "badges")
    for i, line in enumerate(lines[:-1]):
        if line.lower() == name.lower():
            nxt = lines[i + 1]
            if not any(nxt.lower().startswith(l) or l in nxt.lower()
                       for l in labels):
                stats["motto"] = nxt.strip(" t") if _re.fullmatch(
                    r"t .+ t", nxt) else nxt
            break
    m = _re.search(r"^[^\n]*\bVIP\b[^\n]*$", text, _re.M)
    if m:
        stats["vip"] = m.group(0)
    m = _re.search(r"Achievement points\s*\n?\s*([\d.,]+)", text, _re.I)
    if m:
        stats["achievements"] = _count(m.group(1))
    m = _re.search(r"On the island for\s*\n\s*([^\n]+)", text, _re.I)
    if m:
        stats["island"] = m.group(1)
    m = _re.search(r"Ring season\s*\n\s*([^\n\d][^\n]*)\s*\n\s*([\d.,]+)\s*\n?"
                   r"\s*points?\s*\n?\s*([\d.,]+)\s*\n?\s*of\s*\n?\s*([\d.,]+)"
                   r"\s*\n?\s*fights?", text, _re.I)
    if m:
        stats["ring"] = {"rank": m.group(1).strip(),
                         "points": _count(m.group(2)),
                         "fights": _count(m.group(3)),
                         "of": _count(m.group(4))}
    heads = [m.end() for m in _re.finditer(r"(?m)^Minigames$", text)]
    if heads:
        block = text[heads[-1]:]
        stop = _re.search(r"(?mi)^On the forum$", block)
        if stop:
            block = block[:stop.start()]
        games = []
        for m in _re.finditer(r"(?m)^([^\n\d][^\n]*)\n\s*([\d.,]+)\s*\n?\s*"
                              r"wins?\b[^\n]*(?:\n\s*Best place:?\s*\n?\s*"
                              r"([\d.,]+))?", block, _re.I):
            games.append({"name": m.group(1).strip(),
                          "wins": _count(m.group(2)),
                          "best": _count(m.group(3)) if m.group(3) else None})
        if games:
            stats["games"] = games
    m = _re.search(r"([\d.,]+)\s*\n?\s*posts?\b", text, _re.I)
    if m:
        stats["posts"] = _count(m.group(1))
    m = _re.search(r"([\d.,]+)\s*\n?\s*topics?\b", text, _re.I)
    if m:
        stats["topics"] = _count(m.group(1))
    return stats


def parse_profile(page, slug):
    import html as _html
    import re as _re
    text = _html.unescape(page)
    links = _re.findall(r"https://api\.boombang\.tv/api/avatar\?[^\"'\s<>)]+",
                        text)
    face = next((u for u in links if "variant=caragrande" in u), None) \
        or (links[0] if links else None)
    name = None
    for pattern in (r'<meta[^>]+property="og:title"[^>]+content="([^"]+)"',
                    r'<meta[^>]+content="([^"]+)"[^>]+property="og:title"',
                    r"<title>([^<]+)</title>"):
        m = _re.search(pattern, text, _re.I)
        if m:
            name = _re.split(r"\s+[-|\u2013\u2014\u00b7:]\s+|'s\b", m.group(1))[0]
            break
    if not name or slug.lower() not in name.lower():
        name = slug[:1].upper() + slug[1:]
    colours = []
    if face:
        m = _re.search(r"colors=([0-9A-Fa-f]+)", face)
        if m:
            code = m.group(1)
            colours = ["#" + code[i:i + 6].upper()
                       for i in range(0, len(code) - 5, 6)]
    name = name.strip()
    try:
        stats = profile_stats(page, name)
    except Exception as e:
        log("Profile stats not read: %s" % e, key="stats", every=0)
        stats = {}
    return {"slug": slug, "name": name, "avatar": face,
            "colours": colours, "stats": stats, "fetched": time.time()}


def fetch_profile(slug, timeout=10, force=False):
    import urllib.request
    kept = load_profile()
    if not force and kept and kept.get("slug") == slug and \
            time.time() - kept.get("fetched", 0) < PROFILE_REFRESH and \
            os.path.exists(AVATAR_PATH):
        return kept
    headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"}
    try:
        req = urllib.request.Request(PROFILE_URL % slug, headers=headers)
        with urllib.request.urlopen(req, timeout=timeout) as r:
            page = r.read().decode("utf-8", "replace")
        profile = parse_profile(page, slug)
        if profile["avatar"]:
            url = profile["avatar"]
            url = url.replace("size=320", "size=128").replace("size=256",
                                                                "size=128")
            req = urllib.request.Request(url, headers=headers)
            with urllib.request.urlopen(req, timeout=timeout) as r:
                data = r.read()
            with open(AVATAR_PATH, "wb") as f:
                f.write(data)
        with open(PROFILE_PATH, "w", encoding="utf-8") as f:
            json.dump(profile, f, indent=1)
        return profile
    except Exception as e:
        log("Profile not fetched (%s); using what was kept." % e,
            key="profile", every=0)
        return kept if kept and kept.get("slug") == slug else None


def load_profile():
    try:
        with open(PROFILE_PATH, "r", encoding="utf-8") as f:
            data = json.load(f)
        return data if isinstance(data, dict) else None
    except Exception:
        return None


def avatar_disc(size, background):
    try:
        raw = np.fromfile(AVATAR_PATH, dtype=np.uint8)
        img = cv2.imdecode(raw, cv2.IMREAD_UNCHANGED)
    except Exception:
        return None
    if img is None:
        return None
    if img.ndim == 2:
        img = cv2.cvtColor(img, cv2.COLOR_GRAY2BGRA)
    if img.shape[2] == 3:
        img = cv2.cvtColor(img, cv2.COLOR_BGR2BGRA)
    h, w = img.shape[:2]
    side = min(h, w)
    x0 = (w - side) // 2
    img = img[:side, x0:x0 + side]
    big = size * 4
    img = cv2.resize(img, (big, big), interpolation=cv2.INTER_AREA)
    alpha = img[..., 3:4].astype(np.float32) / 255.0
    back = np.zeros((big, big, 3), np.float32)
    back[:] = _bgr(background)
    ring = np.zeros((big, big, 3), np.float32)
    ring[:] = _bgr(C.SURFACE_2)
    face = img[..., :3].astype(np.float32) * alpha + ring * (1 - alpha)
    yy, xx = np.mgrid[:big, :big]
    inside = (((xx - big / 2.0 + 0.5) ** 2 + (yy - big / 2.0 + 0.5) ** 2)
              <= (big / 2.0 - 1) ** 2)[..., None]
    out = np.where(inside, face, back)
    return cv2.resize(out.astype(np.uint8), (size, size),
                      interpolation=cv2.INTER_AREA)


ME_EVERY = 0.8
ME_TRUST = 6.0


def telling_colours(colours):
    out = []
    for c in colours:
        try:
            r, g, b = _rgb(c)
        except Exception:
            continue
        hi, lo = max(r, g, b), min(r, g, b)
        if hi < 60:
            continue
        if hi - lo < 40:
            continue
        if c not in out:
            out.append(c)
    return out[:4]


def find_me(frame, colours, size=(56, 96), margin=0, near=None, step=2,
            least=0.006, avoid=()):
    colours = telling_colours(colours)
    if frame is None or not colours:
        return None
    view, ox, oy = _crop_margin(frame, margin)
    small = view[::step, ::step]
    lab = cv2.cvtColor(small, cv2.COLOR_BGR2LAB).astype(np.float32)
    kw = max(3, int(size[0] / step)) | 1
    kh = max(3, int(size[1] / step)) | 1
    blank = np.zeros(lab.shape[:2], bool)
    for bx, by, bw, bh in avoid:
        x0 = max(0, int((bx - ox) / step) - 1)
        y0 = max(0, int((by - oy) / step) - 1)
        blank[y0:int((by - oy + bh) / step) + 2,
              x0:int((bx - ox + bw) / step) + 2] = True
    usable = []
    for c in colours:
        swatch = cv2.cvtColor(np.uint8([[_bgr(c)]]), cv2.COLOR_BGR2LAB)
        d = np.linalg.norm(lab - swatch.astype(np.float32)[0, 0], axis=2)
        hit = (d < 16) & ~blank
        common = float(hit.mean())
        if common > 0.04 or not hit.any():
            continue
        usable.append((hit, common))
    if not usable:
        return None
    score = np.zeros(lab.shape[:2], np.float32)
    need = np.zeros(lab.shape[:2], np.float32)
    matched = np.zeros(lab.shape[:2], bool)
    for hit, _common in usable:
        matched |= hit
        share = cv2.boxFilter(hit.astype(np.float32), -1, (kw, kh),
                              normalize=True)
        present = (share >= least).astype(np.float32)
        score += present
        need += present * share
    if len(usable) >= 2:
        score = np.where(score >= 2, score + need * 4.0, 0)
    elif usable[0][1] < 0.005:
        score = np.where(score >= 1, score + need * 4.0, 0)
    else:
        return None
    if near is not None:
        nx, ny = (near[0] - ox) / step, (near[1] - oy) / step
        yy, xx = np.mgrid[:score.shape[0], :score.shape[1]]
        far = np.sqrt((xx - nx) ** 2 + (yy - ny) ** 2) / max(score.shape)
        score = np.where(score > 0, score - far * 0.5, 0)
    best = float(score.max())
    if best <= 0:
        return None
    y, x = np.unravel_index(int(np.argmax(score)), score.shape)
    y0, y1 = max(0, y - kh // 2), min(score.shape[0], y + kh // 2 + 1)
    x0, x1 = max(0, x - kw // 2), min(score.shape[1], x + kw // 2 + 1)
    ys, xs = np.nonzero(matched[y0:y1, x0:x1])
    if len(ys):
        fx = x0 + (int(xs.min()) + int(xs.max())) / 2.0
        fy = y0 + max(int(ys.max()), int(ys.min()) + kh * 0.45)
    else:
        fx, fy = x, y + kh * 0.35
    return (int(fx * step + ox), int(min(fy, score.shape[0] - 1) * step + oy),
            best)


class C:
    BG = "#0F1012"
    SURFACE = "#16171A"
    SURFACE_2 = "#1D1E22"
    SURFACE_3 = "#26282D"
    SUNK = "#0A0A0C"
    LINE = "#24262A"
    LINE_2 = "#303238"

    INK = "#ECEDEF"
    INK_2 = "#A4A7AE"
    INK_3 = "#71747C"
    INK_OFF = "#46484E"

    GOOD = "#4CC38A"
    DATA = "#8A9BB8"
    WARN = "#D9A441"
    RED = "#E5534B"
    RED_DIM = "#3A1D1C"

    AMBER = GOOD
    AMBER_HOT = GOOD
    AMBER_DIM = "#1E3A2C"
    AMBER_WELL = SURFACE_2
    GREEN = GOOD
    BLUE = INK
    BLUE_DIM = SURFACE_2
    BLUE_WELL = SURFACE_2
    CHASSIS = BG
    CHASSIS_2 = SURFACE
    WELL = SURFACE
    WELL_2 = SUNK
    RAISED = SURFACE_2
    RAISED_HI = SURFACE_3
    RAISED_LO = "#141518"
    RIDGE = LINE_2
    GROOVE = LINE
    EDGE = LINE_2


UI_FONTS = ("Inter", "Segoe UI Variable Text", "Segoe UI", "Helvetica Neue", "Arial")
NUM_FONTS = ("JetBrains Mono", "Cascadia Mono", "IBM Plex Mono", "Consolas",
             "Courier New")

W, H = 1040, 788
SCALE = 1.0
MIN_SCALE = 0.65
MAX_SCALE = 2.0
SCALE_STEP = 0.05


def _rgb(colour):
    colour = colour.lstrip("#")
    return (int(colour[0:2], 16), int(colour[2:4], 16), int(colour[4:6], 16))


def _bgr(colour):
    r, g, b = _rgb(colour)
    return (b, g, r)


def mix(base, top, t):
    t = clamp(t, 0.0, 1.0)
    (r1, g1, b1), (r2, g2, b2) = _rgb(base), _rgb(top)
    return "#%02X%02X%02X" % (int(round(r1 + (r2 - r1) * t)),
                              int(round(g1 + (g2 - g1) * t)),
                              int(round(b1 + (b2 - b1) * t)))


def dpi_scale(root=None):
    try:
        dpi = ctypes.windll.user32.GetDpiForSystem()
    except Exception:
        dpi = 96
    scale = clamp(round((dpi / 96.0) * 4) / 4, 1.0, 2.0)
    if root is not None:
        try:
            room_w = root.winfo_screenwidth() - 40
            room_h = root.winfo_screenheight() - 90
            while scale > 1.0 and (W * scale > room_w or H * scale > room_h):
                scale -= 0.25
            while scale > 0.7 and (W * scale > room_w or H * scale > room_h):
                scale = round(scale - 0.05, 2)
        except Exception:
            pass
    return scale


def s(v):
    return v * SCALE


def rounded(cv, x, y, w, h, r, **kw):
    x, y, w, h, r = s(x), s(y), s(w), s(h), s(r)
    x2, y2 = x + w, y + h
    pts = [x + r, y, x2 - r, y, x2, y, x2, y + r,
           x2, y2 - r, x2, y2, x2 - r, y2,
           x + r, y2, x, y2, x, y2 - r,
           x, y + r, x, y]
    return cv.create_polygon(pts, smooth=True, splinesteps=24, **kw)


def rect(cv, x, y, w, h, **kw):
    return cv.create_rectangle(s(x), s(y), s(x + w), s(y + h), **kw)


def hline(cv, x1, x2, y, colour, width=1):
    return cv.create_line(s(x1), s(y), s(x2), s(y), fill=colour,
                          width=max(1, s(width)))


def vline(cv, x, y1, y2, colour, width=1):
    return cv.create_line(s(x), s(y1), s(x), s(y2), fill=colour,
                          width=max(1, s(width)))


def engrave(cv, x1, x2, y):
    return hline(cv, x1, x2, y, C.LINE)


def well(cv, x, y, w, h, r=6, fill=C.SURFACE, outline=C.LINE):
    return rounded(cv, x, y, w, h, r, fill=fill, outline=outline,
                   width=max(1, s(1)))


def glow(cv, x, y, w, h, r, colour, base, layers=3, spread=3):
    items = []
    for i in range(layers, 0, -1):
        pad = spread * i
        items.append(rounded(cv, x - pad, y - pad, w + 2 * pad, h + 2 * pad,
                             r + pad, fill=mix(base, colour, 0.22 / (i * 1.5)),
                             outline=""))
    return items


def brackets(cv, x, y, w, h, arm, colour, width=1.5, tags=()):
    items = []
    for cx, cy, dx, dy in ((x, y, 1, 1), (x + w, y, -1, 1),
                           (x, y + h, 1, -1), (x + w, y + h, -1, -1)):
        items.append(cv.create_line(s(cx), s(cy + dy * arm), s(cx), s(cy),
                                    s(cx + dx * arm), s(cy), fill=colour,
                                    width=max(1, s(width)), tags=tags))
    return items


def dot(cv, x, y, r, **kw):
    return cv.create_oval(s(x - r), s(y - r), s(x + r), s(y + r), **kw)


def text(cv, x, y, **kw):
    return cv.create_text(s(x), s(y), **kw)


def measure(font, string):
    return font.measure(string) / SCALE


class Btn:
    _seq = 0

    def __init__(self, cv, x, y, w, h, label, command, font,
                 fill=C.SURFACE_2, hover=None, fg=C.INK_2, border=C.LINE_2,
                 radius=5, pressed=None, sheen=False, dot=False):
        self.cv = cv
        self.command = command
        self.fill = fill
        self.hover = hover or C.SURFACE_3
        self.pressed = pressed or C.RAISED_LO
        self.fg = fg
        self.border = border
        self.enabled = True
        self.dot_colour = C.INK_OFF
        self.shape = rounded(cv, x, y, w, h, radius, fill=fill,
                             outline=border, width=max(1, s(1)))
        self.dot = None
        if dot:
            self.dot = globals()["dot"](cv, x + 13, y + h / 2, 3.5,
                                        fill=C.INK_OFF, outline="")
            self.label = text(cv, x + 23, y + h / 2, text=label, fill=fg,
                              font=font, anchor="w")
        else:
            self.label = text(cv, x + w / 2, y + h / 2, text=label, fill=fg,
                              font=font, anchor="center")
        Btn._seq += 1
        self.tag = "btn%d" % Btn._seq
        for item in (self.shape, self.dot, self.label):
            if item is not None:
                cv.addtag_withtag(self.tag, item)
        cv.tag_bind(self.tag, "<Enter>", self._enter)
        cv.tag_bind(self.tag, "<Leave>", self._leave)
        cv.tag_bind(self.tag, "<ButtonPress-1>", self._press)
        cv.tag_bind(self.tag, "<ButtonRelease-1>", self._release)

    def adopt(self, *items):
        for item in items:
            self.cv.addtag_withtag(self.tag, item)

    def _enter(self, _=None):
        if self.enabled:
            self.cv.itemconfigure(self.shape, fill=self.hover)
            self.cv.configure(cursor="hand2")

    def _leave(self, _=None):
        if self.enabled:
            self.cv.itemconfigure(self.shape, fill=self.fill)
        self.cv.configure(cursor="")

    def _press(self, _=None):
        if self.enabled:
            self.cv.itemconfigure(self.shape, fill=self.pressed)

    def _contains(self, event):
        if event is None:
            return True
        try:
            x1, y1, x2, y2 = self.cv.bbox(self.shape)
        except Exception:
            return True
        x, y = self.cv.canvasx(event.x), self.cv.canvasy(event.y)
        return x1 <= x <= x2 and y1 <= y <= y2

    def _release(self, event=None):
        if not self.enabled:
            return
        inside = self._contains(event)
        self.cv.itemconfigure(self.shape, fill=self.hover if inside else self.fill)
        if inside:
            self.cv.after(1, self.command)

    def restyle(self, fill=None, hover=None, fg=None, border=None, pressed=None,
                dot=None):
        if fill is not None:
            self.fill = fill
            if self.enabled:
                self.cv.itemconfigure(self.shape, fill=fill)
        if border is not None:
            self.border = border
            if self.enabled:
                self.cv.itemconfigure(self.shape, outline=border)
        if hover is not None:
            self.hover = hover
        if pressed is not None:
            self.pressed = pressed
        if fg is not None:
            self.fg = fg
            if self.enabled:
                self.cv.itemconfigure(self.label, fill=fg)
        if dot is not None:
            self.dot_colour = dot
            if self.dot is not None and self.enabled:
                self.cv.itemconfigure(self.dot, fill=dot)

    def set_label(self, value):
        self.cv.itemconfigure(self.label, text=value)

    def set_enabled(self, value):
        value = bool(value)
        if value == self.enabled:
            return
        self.enabled = value
        if self.enabled:
            self.cv.itemconfigure(self.shape, fill=self.fill, outline=self.border)
            self.cv.itemconfigure(self.label, fill=self.fg)
            if self.dot is not None:
                self.cv.itemconfigure(self.dot, fill=self.dot_colour)
        else:
            self.cv.itemconfigure(self.shape, fill=C.SURFACE, outline=C.LINE)
            self.cv.itemconfigure(self.label, fill=C.INK_OFF)
            if self.dot is not None:
                self.cv.itemconfigure(self.dot, fill=C.INK_OFF)

class ItemGrabber:
    VERSION = "1.8"

    def __init__(self, root):
        global SCALE

        self.root = root
        SCALE = dpi_scale(root)

        self.root.title("Cruel Auto Event")
        self.root.configure(bg=C.BG)
        self.root.resizable(True, True)
        self.root.minsize(int(W * 0.55), int(H * 0.55))

        self.ui_queue = queue.Queue()
        self._pump_job = None
        self._save_job = None
        self._cfg_lock = threading.Lock()
        self._fonts = {}
        self._drag = (0, 0)
        self._photo = None
        self._preview_item = None
        self._swatches = []
        self.last_frame = None
        self.last_verdict = None

        self.game_hwnd = None
        self.game_active = False
        self.client_now = None
        self.shutting_down = False
        self.closing = threading.Event()
        self.hotkeys_blocked = False
        self.own_hwnds = set()
        self.frameless = False
        self.parent_hwnd = None

        self._target_cache = {}
        self._target_key = None
        self._sct_local = threading.local()
        self._cap_lock = threading.Lock()
        self._pw = None
        self._mirror = {}
        self.live_at = 0.0
        self.new_frame = threading.Event()
        self.looks_counted = 0
        self.looks_since = time.perf_counter()
        self.last_look = 0.0
        self.crop_mode = False
        self.crop_frame = None
        self.crop_start = None
        self.preview_geom = None
        self._alerted = {}
        self._dialog_photos = []
        self.capture_used = None

        self.armed = threading.Event()
        self.seen = 0
        self.taken = 0
        self.dropped = 0
        self.clicks = 0
        self.session_start = time.perf_counter()
        self.last_line = "nothing seen yet"

        self.templates = gather_templates()
        self.by_name = {t.name: t for t in self.templates}
        self.hits = []
        self.chase = None
        self.shelf = Shelf()
        self.attempts = Attempts()
        self.awaiting = None
        self.walked_to = []
        self.popup_since = None
        self.idle_shown = 0.0
        self.claimed = 0
        self.unclaimed = 0
        self.stolen = 0
        self.pending = None
        self.last_prize = None
        self.prize_state = None
        self.floor_last = []
        self.floor_last_at = 0.0
        self.floor_mid = None
        self.vanished = []
        self.spawns = SpawnMap()
        self.popup_open = False
        self.popup_missing = None
        self.popup_item = None
        self.recent_loss = None
        self.session_counts = {}
        self.catch_times = []
        self.events = []
        self.stats = self.load_stats()
        self.home_plan = []
        self.verdict_state = ([], None)
        self.live_frame = None
        self.live_seq = 0
        self.live_shown = -1
        self.visible = True
        self.fps_frames = 0
        self.fps_since = time.perf_counter()
        self._fit_job = None
        self._fit_size = None
        self.maximized = False
        self.ox = self.oy = 0
        self.ping = None
        self.ping_seen = 0.0
        self.ping_recent = []
        self.ping_read = 0.0
        self._stage_list = []
        self._stage_at = 0.0
        self.lag_hold = False
        self.lag_clear_since = None
        self.profile = load_profile()
        self._avatar_photo = None
        self.me = None
        self.me_walk = None
        self.me_looked = 0.0
        self.me_announced = False
        self.card_open = False
        self.card_fetching = False
        self._card_photo = None

        cfg = self.load_config()
        if _is_num(cfg.get("ui_scale")):
            SCALE = clamp(round(float(cfg["ui_scale"]), 3),
                          MIN_SCALE, self.max_fitting_scale())
        self.ui_scale = SCALE
        self.apply_config_values(cfg)

        self.centre_window()
        self.apply_window_chrome()
        self.hires_timer = raise_timer_resolution()

        self.ui_family = self.resolve_font(UI_FONTS, "Arial")
        self.num_family = self.resolve_font(NUM_FONTS, "Courier New")

        self.build_ui()
        self.bind_hotkeys()

        if self.config_fixes:
            n = len(self.config_fixes)
            self.cv.itemconfigure(self.status_line, fill=C.WARN,
                                  text="%d config note%s on load \u2014 see console"
                                       % (n, "" if n == 1 else "s"))

        self.root.protocol("WM_DELETE_WINDOW", self.shutdown)
        self.root.bind("<Configure>", self.on_configure)
        self.root.bind("<Escape>", self.on_escape)
        if self.start_maximized:
            try:
                self.root.state("zoomed")
            except Exception:
                pass
        threading.Thread(target=self.watchdog_loop, daemon=True).start()
        threading.Thread(target=self.chase_loop, daemon=True).start()
        threading.Thread(target=self.tick_loop, daemon=True).start()
        threading.Thread(target=self.live_loop, daemon=True).start()
        threading.Thread(target=self.profile_loop, daemon=True).start()
        self.pump_ui()
        self.live_tick()

    def default_config(self):
        return {
            "version": CONFIG_VERSION,
            "min_score": 0.88,
            "prize_score": 0.80,
            "prize_wait": 1.0,
            "poll": 0.0,
            "capture_fps": LIVE_FPS,
            "watch_rev": 2,
            "click_every": 0.6,
            "patience": 12.0,
            "margin": 0,
            "spread": 2,
            "hover_ms": 35,
            "dwell_ms": 30,
            "button": "left",
            "auto_click": True,
            "nearest_first": True,
            "answer_prize": True,
            "return_home": True,
            "maximized": False,
            "profile": "cruel",
            "lag_guard": True,
            "find_me": True,
            "order": list(DEFAULT_ORDER),
            "off_items": [],
            "removed_items": [],
            "names": {},
            "alerts": True,
            "capture_method": "print",
            "topmost": False,
            "ui_scale": None,
        }

    def load_config(self):
        cfg = self.default_config()
        self.config_fixes = []
        backup_config()
        try:
            with open(CONFIG_PATH, "r", encoding="utf-8") as f:
                saved = json.load(f)
            if isinstance(saved, dict):
                if saved.get("version") != CONFIG_VERSION:
                    self.config_fixes.append(
                        "settings were written by an older version; defaults "
                        "used, the old file is kept in %s"
                        % os.path.basename(BACKUP_PATH))
                    saved = {}
                for k in cfg:
                    if k in saved:
                        cfg[k] = saved[k]
            if isinstance(saved, dict) and saved and \
                    saved.get("watch_rev", 1) < 2:
                cfg["poll"] = 0.0
                cfg["watch_rev"] = 2
                log("Settings moved over to watching every frame; 'Look "
                    "every' can still be turned up.", every=0)
        except FileNotFoundError:
            pass
        except Exception as e:
            log("Config load failed, using defaults: %s" % e)
            self.config_fixes.append("config unreadable (%s); defaults used, "
                                     "original kept in %s"
                                     % (e, os.path.basename(BACKUP_PATH)))
        self.config_fixes += self.normalise_config(cfg)
        for fix in self.config_fixes:
            log("Config: %s" % fix, key="cfg:" + fix, every=0)
        return cfg

    def normalise_config(self, cfg):
        fallback = self.default_config()
        fixes = []

        def reset(key, why):
            fixes.append("%s %r %s; using %r" % (key, cfg.get(key), why,
                                                 fallback[key]))
            cfg[key] = fallback[key]

        def as_int(key, low, high):
            try:
                raw = int(cfg[key])
            except (KeyError, TypeError, ValueError, OverflowError):
                reset(key, "is not a whole number")
                return
            value = int(clamp(raw, low, high))
            if value != raw:
                fixes.append("%s %d is outside %d to %d; using %d"
                             % (key, raw, low, high, value))
            cfg[key] = value

        def as_float(key, low, high):
            try:
                raw = float(cfg[key])
                if raw != raw:
                    raise ValueError("NaN")
            except (KeyError, TypeError, ValueError):
                reset(key, "is not a number")
                return
            value = float(clamp(raw, low, high))
            if abs(value - raw) > 1e-12:
                fixes.append("%s %g is outside %g to %g; using %g"
                             % (key, raw, low, high, value))
            cfg[key] = value

        as_int("margin", *LOAD_BOUNDS["margin"])
        as_int("spread", *LOAD_BOUNDS["spread"])
        as_int("hover_ms", *LOAD_BOUNDS["hover_ms"])
        as_int("dwell_ms", *LOAD_BOUNDS["dwell_ms"])
        as_float("min_score", *LOAD_BOUNDS["min_score"])
        as_float("prize_score", *LOAD_BOUNDS["prize_score"])
        as_float("prize_wait", *LOAD_BOUNDS["prize_wait"])
        as_float("poll", *LOAD_BOUNDS["poll"])
        as_float("click_every", *LOAD_BOUNDS["click_every"])
        as_float("patience", *LOAD_BOUNDS["patience"])
        as_int("capture_fps", CAPTURE_STEPS[0], CAPTURE_STEPS[-1])

        if cfg.get("button") not in BUTTONS:
            reset("button", "is not left, middle or right")
        if cfg.get("capture_method") not in ("print", "screen"):
            reset("capture_method", "is not print or screen")

        off = cfg.get("off_items")
        if not isinstance(off, list) or not all(isinstance(v, str) for v in off):
            reset("off_items", "is not a list of item names")
        order = cfg.get("order")
        if not isinstance(order, list) or not all(isinstance(v, str)
                                                  for v in order):
            reset("order", "is not a list of item names")
        gone = cfg.get("removed_items")
        if not isinstance(gone, list) or not all(isinstance(v, str)
                                                 for v in gone):
            reset("removed_items", "is not a list of item names")
        names = cfg.get("names")
        if not isinstance(names, dict) or not all(
                isinstance(k, str) and isinstance(v, str)
                for k, v in names.items()):
            reset("names", "is not a list of item names")

        for key in ("order", "off_items"):
            renamed, seen = [], set()
            for name in cfg[key]:
                name = RENAMED.get(name, name)
                if name not in seen:
                    seen.add(name)
                    renamed.append(name)
            if renamed != cfg[key]:
                fixes.append("%s used an old item name; updated" % key)
            cfg[key] = renamed

        scale = cfg.get("ui_scale")
        if scale is not None:
            if not _is_num(scale):
                reset("ui_scale", "is not a number")
            else:
                cfg["ui_scale"] = float(clamp(float(scale), MIN_SCALE, MAX_SCALE))

        cfg["auto_click"] = bool(cfg["auto_click"])
        cfg["nearest_first"] = bool(cfg["nearest_first"])
        cfg["answer_prize"] = bool(cfg["answer_prize"])
        cfg["return_home"] = bool(cfg["return_home"])
        cfg["maximized"] = bool(cfg["maximized"])
        cfg["lag_guard"] = bool(cfg["lag_guard"])
        cfg["find_me"] = bool(cfg["find_me"])
        cfg["alerts"] = bool(cfg["alerts"])
        cfg["profile"] = str(cfg.get("profile") or "").strip().lower()
        cfg["topmost"] = bool(cfg["topmost"])
        return fixes

    def apply_config_values(self, cfg):
        self.min_score = cfg["min_score"]
        self.prize_score = cfg["prize_score"]
        self.prize_wait = cfg["prize_wait"]
        self.answer_prize = cfg["answer_prize"]
        self.return_home = cfg["return_home"]
        self.start_maximized = cfg["maximized"]
        self.profile_slug = cfg["profile"]
        self.lag_guard = cfg["lag_guard"]
        self.find_me_on = cfg["find_me"]
        self.poll = 0.0
        self.capture_fps = cfg["capture_fps"]
        self.click_every = cfg["click_every"]
        self.patience = cfg["patience"]
        self.margin = cfg["margin"]
        self.spread = cfg["spread"]
        self.hover_ms = cfg["hover_ms"]
        self.dwell_ms = cfg["dwell_ms"]
        self.button = cfg["button"]
        self.auto_click = cfg["auto_click"]
        self.nearest_first = cfg["nearest_first"]
        self.capture_method = cfg["capture_method"]
        self.topmost = False
        self.alerts = cfg["alerts"]
        self.removed = set(cfg["removed_items"])
        CUSTOM_NAMES.clear()
        CUSTOM_NAMES.update({k: v for k, v in cfg["names"].items() if v.strip()})
        self.templates = [t for t in self.templates
                          if t.name not in self.removed]
        self.by_name = {t.name: t for t in self.templates}
        off = set(cfg["off_items"])
        for template in self.templates:
            template.enabled = template.name not in off
        self.set_order(cfg["order"])

    def set_order(self, names):
        rank = {name: i for i, name in enumerate(names)}
        self.templates.sort(key=lambda t: (rank.get(t.name, len(rank)), t.name))
        for i, template in enumerate(self.templates):
            template.rank = i

    def promote(self, template):
        order = [t.name for t in self.templates]
        i = order.index(template.name)
        if i > 0:
            order[i - 1], order[i] = order[i], order[i - 1]
            self.set_order(order)
            self.render_items()
            self.save_config_soon()

    def current_config(self):
        return {
            "min_score": self.min_score,
            "prize_score": self.prize_score,
            "prize_wait": self.prize_wait,
            "poll": self.poll,
            "capture_fps": self.capture_fps,
            "click_every": self.click_every,
            "patience": self.patience,
            "margin": self.margin,
            "spread": self.spread,
            "hover_ms": self.hover_ms,
            "dwell_ms": self.dwell_ms,
            "button": self.button,
            "auto_click": self.auto_click,
            "nearest_first": self.nearest_first,
            "answer_prize": self.answer_prize,
            "return_home": self.return_home,
            "maximized": self.maximized,
            "profile": self.profile_slug,
            "lag_guard": self.lag_guard,
            "find_me": self.find_me_on,
            "order": [t.name for t in self.templates],
            "off_items": sorted(t.name for t in self.templates
                                if not t.enabled),
            "removed_items": sorted(self.removed),
            "names": dict(sorted(CUSTOM_NAMES.items())),
            "alerts": self.alerts,
            "capture_method": self.capture_method,
            "topmost": self.topmost,
            "ui_scale": self.ui_scale,
            "version": CONFIG_VERSION,
            "watch_rev": 2,
        }

    def save_config_soon(self, delay=1200):
        if self.shutting_down:
            return
        self._cancel_save_job()
        self._save_job = self.root.after(delay, self.save_config)

    def _cancel_save_job(self):
        if self._save_job is not None:
            try:
                self.root.after_cancel(self._save_job)
            except Exception:
                pass
            self._save_job = None

    def save_config(self):
        try:
            self._cancel_save_job()
        except Exception:
            self._save_job = None
        data = self.current_config()
        try:
            with self._cfg_lock:
                tmp = CONFIG_PATH + ".tmp"
                with open(tmp, "w", encoding="utf-8") as f:
                    json.dump(data, f, indent=2)
                os.replace(tmp, CONFIG_PATH)
        except Exception as e:
            log("Config save failed: %s" % e)


    def ui(self, fn, *args, **kwargs):
        self.ui_queue.put((fn, args, kwargs))

    def set_text(self, name, text=None, fill=None):
        item = getattr(self, name, None)
        if item is None:
            return
        options = {}
        if text is not None:
            options["text"] = text
        if fill is not None:
            options["fill"] = fill
        if options:
            self.cv.itemconfigure(item, **options)

    def pump_ui(self):
        try:
            while True:
                fn, args, kwargs = self.ui_queue.get_nowait()
                try:
                    fn(*args, **kwargs)
                except Exception as e:
                    log("UI update failed: %s" % e)
        except queue.Empty:
            pass
        if not self.shutting_down:
            self._pump_job = self.root.after(15, self.pump_ui)

    def status(self, message, colour=C.INK_2):
        self.ui(self.show_status, message, colour)

    def event(self, message, colour=C.INK_2):
        self.events.append((time.strftime("%H:%M:%S"), message, colour))
        del self.events[:-60]
        log(message, key="event:" + message, every=0)
        self.ui(self.render_events)

    def show_status(self, message, colour=C.INK_2):
        item = getattr(self, "status_line", None)
        if item is None:
            return
        self.cv.itemconfigure(item, text=self.fit(message, self.font(10.5),
                                                  self.status_room),
                              fill=colour)

    def fit(self, value, font, room):
        if measure(font, value) <= room:
            return value
        while value and measure(font, value + "\u2026") > room:
            value = value[:-1]
        return value.rstrip() + "\u2026"

    def preview(self, frame=None, chase=None):
        self.verdict_state = (list(self.hits), chase)

    def set_verdict(self, chase=None):
        self.verdict_state = (list(self.hits), chase)

    def show_capture(self):
        if self.capture_used == "fallback":
            self.ui(self.set_text, "capture_value", text="Screen (fallback)",
                    fill=C.WARN)
        elif self.capture_used == "print":
            self.ui(self.set_text, "capture_value", text="PrintWindow",
                    fill=C.INK)
        elif self.capture_used == "screen":
            self.ui(self.set_text, "capture_value", text="Screen", fill=C.INK)


    def live_loop(self):
        period = 1.0 / LIVE_FPS
        if self.capture_fps:
            period = 1.0 / float(self.capture_fps)
        while not self.closing.is_set():
            period = 1.0 / float(self.capture_fps or LIVE_FPS)
            started = time.perf_counter()
            try:
                if self.game_active:
                    frame = self.grab()
                    if frame is not None:
                        self.live_frame = frame
                        self.live_at = time.monotonic()
                        self.live_seq += 1
                        self.new_frame.set()
            except Exception as e:
                log("Live feed error: %s" % e, key="live", every=30)
            spent = time.perf_counter() - started
            self.closing.wait(max(0.002, period - spent))

    def current_frame(self):
        frame = self.live_frame
        if frame is not None and time.monotonic() - self.live_at < LIVE_STALE:
            return frame
        fresh = self.grab(wait=0.05)
        if fresh is not None:
            self.live_frame = fresh
            self.live_at = time.monotonic()
            self.live_seq += 1
            return fresh
        if frame is not None and \
                time.monotonic() - self.live_at < FRAME_TOO_OLD:
            return frame
        return None

    def next_frame(self, timeout=0.12):
        seq = self.live_seq
        end = time.monotonic() + timeout
        while self.live_seq == seq and time.monotonic() < end:
            if self.closing.wait(0.004):
                break
        return self.live_frame

    def live_tick(self):
        try:
            try:
                self.visible = self.root.state() != "iconic"
            except Exception:
                self.visible = True
            seq = self.live_seq
            if seq != self.live_shown and self.live_frame is not None \
                    and not self.crop_mode and self.visible:
                self.live_shown = seq
                self.show_preview(self.live_frame, self.verdict_state)
                self.fps_frames += 1
            now = time.perf_counter()
            if now - self.fps_since >= 1.0:
                fps = self.fps_frames / (now - self.fps_since)
                self.fps_frames, self.fps_since = 0, now
                self.render_live(fps)
        except Exception as e:
            log("Live view error: %s" % e, key="liveview", every=30)
        if not self.shutting_down:
            self.root.after(8, self.live_tick)


    def apply_window_chrome(self):
        self.frameless = False
        try:
            self.root.update_idletasks()
            self.own_hwnds.add(self.root.winfo_id())
            hwnd = win32gui.GetParent(self.root.winfo_id()) \
                or self.root.winfo_id()
            self.parent_hwnd = hwnd
            self.own_hwnds.add(hwnd)
            dwm = ctypes.windll.dwmapi
            on = ctypes.c_int(1)
            for attr in (20, 19):
                if dwm.DwmSetWindowAttribute(hwnd, attr, ctypes.byref(on),
                                             ctypes.sizeof(on)) == 0:
                    break
            r, g, b = _rgb(C.BG)
            caption = ctypes.c_int(r | (g << 8) | (b << 16))
            dwm.DwmSetWindowAttribute(hwnd, 35, ctypes.byref(caption),
                                      ctypes.sizeof(caption))
        except Exception as e:
            log("Title bar styling unavailable: %s" % e, key="chrome", every=0)
        try:
            self.root.attributes("-topmost", False)
        except Exception:
            pass

    def max_fitting_scale(self):
        try:
            room_w = self.root.winfo_screenwidth() - 20
            room_h = self.root.winfo_screenheight() - 90
            return clamp(min(room_w / float(W), room_h / float(H)),
                         MIN_SCALE, MAX_SCALE)
        except Exception:
            return MAX_SCALE

    def centre_window(self):
        try:
            sw = self.root.winfo_screenwidth()
            sh = self.root.winfo_screenheight()
            self.root.geometry("%dx%d+%d+%d" % (
                int(s(W)), int(s(H)),
                max(0, (sw - int(s(W))) // 2),
                max(0, (sh - int(s(H))) // 2 - 30)))
        except Exception:
            self.root.geometry("%dx%d" % (int(s(W)), int(s(H))))

    def on_configure(self, event):
        if event.widget is not self.root:
            return
        if self._fit_job is not None:
            try:
                self.root.after_cancel(self._fit_job)
            except Exception:
                pass
        self._fit_job = self.root.after(90, self.fit_to_window)

    def fit_to_window(self):
        global SCALE
        self._fit_job = None
        try:
            w, h = self.root.winfo_width(), self.root.winfo_height()
        except Exception:
            return
        if w < 50 or h < 50 or (w, h) == self._fit_size:
            return
        self._fit_size = (w, h)
        try:
            self.maximized = self.root.state() == "zoomed"
        except Exception:
            self.maximized = False
        scale = round(clamp(min(w / float(W), h / float(H)), MIN_SCALE,
                            MAX_SCALE), 3)
        SCALE = scale
        if not self.maximized:
            self.ui_scale = scale
        self.rebuild()
        self.save_config_soon()

    def rebuild(self):
        if self.crop_mode:
            self.end_crop()
        self._fonts.clear()
        self._photo = None
        self._preview_item = None
        try:
            self.cv.destroy()
        except Exception:
            pass
        self.build_ui()
        if self.card_open:
            self.draw_profile_card()
        self.live_shown = -1
        if self.last_frame is not None:
            self.show_preview(self.last_frame, self.verdict_state)
        if self.hotkeys_blocked:
            self.cv.itemconfigure(self.legend_note,
                                  text="Hotkeys blocked \u2014 run as administrator")

    def apply_scale(self, value):
        value = round(clamp(value, MIN_SCALE, self.max_fitting_scale()), 3)
        try:
            if self.root.state() == "zoomed":
                return False
        except Exception:
            pass
        if abs(value - SCALE) < 1e-6:
            return False
        self.root.geometry("%dx%d" % (int(W * value), int(H * value)))
        return True

    def zoom_by(self, notches):
        self.root.after(1, lambda: self.apply_scale(SCALE + notches * SCALE_STEP))

    def minimize(self):
        self.root.iconify()

    def toggle_topmost(self):
        self.topmost = False
        self.root.attributes("-topmost", False)

    def style_pin(self):
        pass

    def select_on(self, btn):
        btn.restyle(fill=C.SURFACE_2, hover=C.SURFACE_3, pressed=C.RAISED_LO,
                    fg=C.INK, border=C.LINE_2, dot=C.GOOD)

    def select_off(self, btn):
        btn.restyle(fill=C.SURFACE_2, hover=C.SURFACE_3, pressed=C.RAISED_LO,
                    fg=C.INK_3, border=C.LINE_2, dot=C.INK_OFF)

    def live_on(self, btn):
        btn.restyle(fill=C.SURFACE_2, hover=C.SURFACE_3, pressed=C.RAISED_LO,
                    fg=C.INK, border=C.LINE_2)


    def resolve_font(self, candidates, fallback):
        available = set(tkfont.families())
        for family in candidates:
            if family in available:
                return family
        return fallback

    def font(self, size, weight="normal", numeric=False):
        key = (size, weight, numeric)
        if key not in self._fonts:
            self._fonts[key] = tkfont.Font(
                family=self.num_family if numeric else self.ui_family,
                size=-max(6, int(round(size * SCALE))), weight=weight)
        return self._fonts[key]

    def label(self, x, y, value, colour=C.INK_3, size=9.5, anchor="w"):
        return text(self.cv, x, y, text=value, fill=colour,
                    font=self.font(size), anchor=anchor)

    def readout(self, x, y, value="", colour=C.INK, size=10.5, anchor="w",
                weight="normal"):
        return text(self.cv, x, y, text=value, fill=colour,
                    font=self.font(size, weight, numeric=True), anchor=anchor)

    def heading(self, x, y, title):
        tf = self.font(12, "bold")
        text(self.cv, x, y, text=title, fill=C.INK, font=tf, anchor="w")
        return x + measure(tf, title) + 12

    def badge(self, x, title, cy):
        f = self.font(8.5, numeric=True)
        w = measure(f, title) + 12
        rounded(self.cv, x, cy - 9, w, 18, 4, fill=C.SURFACE_2,
                outline=C.LINE_2, width=max(1, s(1)))
        text(self.cv, x + w / 2, cy, text=title, fill=C.INK_2, font=f,
             anchor="center")
        return w

    def stepper(self, x, y, width, name, on_step):
        cv = self.cv
        self.label(x, y + 12, name, C.INK_2, size=9.5)
        bw, bx = 82, x + width - 82
        well(cv, bx, y, bw, 24, r=5, fill=C.SURFACE, outline=C.LINE_2)
        Btn(cv, bx + 1, y + 1, 22, 22, "\u2212", lambda: on_step(-1),
            self.font(11), fill=C.SURFACE, border=C.SURFACE, fg=C.INK_3,
            radius=4)
        Btn(cv, bx + bw - 23, y + 1, 22, 22, "+", lambda: on_step(1),
            self.font(11), fill=C.SURFACE, border=C.SURFACE, fg=C.INK_3,
            radius=4)
        return self.readout(bx + bw / 2, y + 12, anchor="center", size=10)

    def panel(self, x, y, w, h, title):
        well(self.cv, x, y, w, h, r=8)
        end = self.heading(x + 20, y + 22, title)
        hline(self.cv, x + 1, x + w - 1, y + 44, C.LINE)
        return end

    def build_ui(self):
        try:
            cw, ch = self.root.winfo_width(), self.root.winfo_height()
        except Exception:
            cw = ch = 0
        cw, ch = max(cw, int(s(W))), max(ch, int(s(H)))
        self.cv = tk.Canvas(self.root, width=cw, height=ch, bg=C.BG,
                            highlightthickness=0, bd=0)
        self.cv.pack(fill="both", expand=True)

        self.build_header()
        self.build_status_strip()
        self.build_preview(20, 144, 620, 432)
        self.build_settings(656, 144, 364, 432)
        self.build_items(20, 588, 620, 164)
        self.build_activity(656, 588, 364, 164)
        self.build_footer(772)

        self.ox = max(0, int((cw - s(W)) / 2))
        self.oy = max(0, int((ch - s(H)) / 2))
        if self.ox or self.oy:
            self.cv.move("all", self.ox, self.oy)
        self.cv.bind("<Control-MouseWheel>",
                     lambda e: self.zoom_by(1 if e.delta > 0 else -1))
        self.render_all()

    def render_all(self):
        self.render_armed()
        self.render_window_status()
        self.render_settings()
        self.render_items()
        self.render_target(self.chase)
        self.render_counts()
        self.render_events()
        self.render_ping()
        self.apply_profile()

    def build_header(self):
        cv = self.cv
        self.avatar_item = None
        self.title_text = text(cv, 24, 32, text="Auto Event", fill=C.INK,
                               font=self.font(15, "bold"), anchor="w")
        self.subtitle_text = self.readout(
            24 + measure(self.font(15, "bold"), "Auto Event") + 10, 33,
            "v%s" % self.VERSION, C.INK_3, size=9.5)

        self.arm_btn = Btn(cv, W - 24 - 116, 15, 116, 34, "Start",
                           self.toggle_armed, self.font(11, "bold"), radius=6)
        self.arm_key = self.readout(W - 24 - 14, 32, "F1", C.INK_3, size=9,
                                    anchor="e")
        self.arm_btn.adopt(self.arm_key)
        cv.coords(self.arm_btn.label, s(W - 24 - 116 + 16), s(32))
        cv.itemconfigure(self.arm_btn.label, anchor="w")

        self.state_dot = dot(cv, W - 24 - 116 - 100, 32, 4, fill=C.INK_OFF,
                             outline="")
        self.state_text = text(cv, W - 24 - 116 - 90, 32, text="Stopped",
                               fill=C.INK_2, font=self.font(10.5), anchor="w")
        hline(cv, 0, W, 64, C.LINE)

    def apply_profile(self):
        profile = self.profile
        if not profile or getattr(self, "title_text", None) is None:
            return
        cv = self.cv
        disc = avatar_disc(int(round(s(38))), C.BG) \
            if os.path.exists(AVATAR_PATH) else None
        x = 24
        if disc is not None:
            rgb = cv2.cvtColor(disc, cv2.COLOR_BGR2RGB)
            h, w = rgb.shape[:2]
            ppm = ("P6\n%d %d\n255\n" % (w, h)).encode("ascii") + rgb.tobytes()
            try:
                self._avatar_photo = tk.PhotoImage(data=ppm)
                if self.avatar_item is None:
                    self.avatar_item = cv.create_image(
                        s(24 + 19) + self.ox, s(32) + self.oy,
                        image=self._avatar_photo, anchor="center")
                else:
                    cv.itemconfigure(self.avatar_item, image=self._avatar_photo)
                x = 24 + 38 + 12
            except Exception as e:
                log("Avatar could not be shown: %s" % e, key="avatar", every=0)
        name = profile.get("name") or "Auto Event"
        cv.itemconfigure(self.title_text, text=name, anchor="w")
        cv.coords(self.title_text, s(x) + self.ox, s(24) + self.oy)
        cv.itemconfigure(self.subtitle_text,
                         text="Auto Event \u00b7 v%s" % self.VERSION)
        cv.coords(self.subtitle_text, s(x) + self.ox, s(44) + self.oy)
        for item in (self.avatar_item, self.title_text, self.subtitle_text):
            if item is None:
                continue
            cv.tag_bind(item, "<Button-1>", self.open_profile_card)
            cv.tag_bind(item, "<Enter>",
                        lambda e: self.cv.configure(cursor="hand2"))
            cv.tag_bind(item, "<Leave>",
                        lambda e: self.cv.configure(cursor=""))

    def build_status_strip(self):
        cv = self.cv
        top, val = 86, 110
        names = ["Game", "Caught", "Per hour", "Prizes claimed", "Missed",
                 "Session", "Ping"]
        xs = [24 + i * 143 for i in range(len(names))]
        for x, name in zip(xs, names):
            self.label(x, top, name, C.INK_3, size=9.5)
        for x in xs[1:]:
            vline(cv, x - 18, 80, 118, C.LINE)

        self.window_lamp = dot(cv, xs[0] + 4, val, 4, fill=C.RED, outline="")
        self.window_value = text(cv, xs[0] + 15, val, text="Not found",
                                 fill=C.INK, font=self.font(12, "bold"),
                                 anchor="w")
        big = dict(size=15, weight="bold")
        self.rounds_value = self.readout(xs[1], val, "0", **big)
        self.rate_value = self.readout(xs[2], val, "\u2014", **big)
        self.claimed_value = self.readout(xs[3], val, "0", **big)
        self.missed_value = self.readout(xs[4], val, "0", **big)
        self.uptime_value = self.readout(xs[5], val, "0:00", **big)
        self.ping_dot = dot(cv, xs[6] + 4, val, 4, fill=C.INK_OFF, outline="")
        self.ping_value = text(cv, xs[6] + 15, val, text="\u2014", fill=C.INK_3,
                               font=self.font(12, "bold"), anchor="w")
        self.ping_note = self.label(xs[6], val + 16, "", C.INK_3, size=8.5)
        hline(cv, 20, W - 20, 132, C.LINE)

    def build_preview(self, x, y, w, h):
        cv = self.cv
        left, right = x + 20, x + w - 20
        end = self.panel(x, y, w, h, "Live view")
        self.live_dot = dot(cv, end + 4, y + 22, 3.5, fill=C.INK_OFF,
                            outline="")
        self.live_text = self.readout(end + 14, y + 23, "", C.INK_3, size=9)

        f = self.font(9.5)
        Btn(cv, right - 84, y + 9, 84, 26, "Look once", self.test_once, f)
        Btn(cv, right - 174, y + 9, 84, 26, "Save image", self.dump_debug, f)
        Btn(cv, right - 272, y + 9, 92, 26, "Forget spots", self.forget_spots, f)

        bx, by, bw, bh = left, y + 58, right - left, 314
        self.preview_box = (bx, by, bw, bh)
        rect(cv, bx, by, bw, bh, fill=C.SUNK, outline=C.LINE)
        self.preview_hint = text(cv, bx + bw / 2, by + bh / 2,
                                 text="Waiting for the game window",
                                 fill=C.INK_3, font=self.font(10.5),
                                 anchor="center")
        rounded(cv, bx + 10, by + 10, 56, 20, 4, fill=C.SUNK, outline=C.LINE_2,
                width=max(1, s(1)), tags=("hud",))
        self.hud_live_dot = dot(cv, bx + 21, by + 20, 3, fill=C.INK_3,
                                outline="", tags=("hud",))
        self.hud_live = text(cv, bx + 29, by + 20, text="LIVE", fill=C.INK_2,
                             font=self.font(8.5, "bold", numeric=True),
                             anchor="w", tags=("hud",))

        self.verdict_value = text(cv, left, by + bh + 22, text="\u2014",
                                  fill=C.INK, font=self.font(13, "bold"),
                                  anchor="w")
        self.verdict_note = self.readout(left, by + bh + 42, "", C.INK_3,
                                         size=9.5)

    def build_settings(self, x, y, w, h):
        cv = self.cv
        left, right = x + 20, x + w - 20
        inner = right - left
        self.panel(x, y, w, h, "Detection")

        self.label(left, y + 64, "Match confidence", C.INK_3, size=9.5)
        self.best_value = self.readout(left, y + 92, "\u2014", size=24,
                                       weight="bold")
        self.best_note = self.readout(right, y + 95, "", C.INK_3, size=9,
                                      anchor="e")
        top = y + 114
        self.score_geom = (left, inner, top, 6)
        rounded(cv, left, top, inner, 6, 3, fill=C.SUNK, outline="")
        self.score_fill = rect(cv, left, top, 0, 6, fill=C.INK_3, outline="")
        self.score_tick = vline(cv, left, top - 4, top + 10, C.INK_2, 1.5)

        hline(cv, left, right, y + 136, C.LINE)

        half = (inner - 20) / 2
        col2 = left + half + 20
        rows = [y + 148, y + 178, y + 208, y + 238]
        self.min_score_value = self.stepper(left, rows[0], half, "Min score",
                                            self.step_min_score)
        self.poll_value = self.stepper(col2, rows[0], half, "Frames/s",
                                       self.step_poll)
        self.click_every_value = self.stepper(left, rows[1], half,
                                              "Click every",
                                              self.step_click_every)
        self.patience_value = self.stepper(col2, rows[1], half, "Give up",
                                           self.step_patience)
        self.margin_value = self.stepper(left, rows[2], half, "Edge margin",
                                         self.step_margin)
        self.spread_value = self.stepper(col2, rows[2], half, "Spread",
                                         self.step_spread)
        self.prize_value = self.stepper(left, rows[3], half, "Prize score",
                                        self.step_prize_score)
        self.prize_wait_value = self.stepper(col2, rows[3], half, "Prize wait",
                                             self.step_prize_wait)

        hline(cv, left, right, y + 278, C.LINE)

        cw = (inner - 16) / 3.0
        f = self.font(9.5)
        r1, r2 = y + 290, y + 324
        self.auto_btn = Btn(cv, left, r1, cw, 26, "Clicking on",
                            self.toggle_auto, f, dot=True)
        self.order_btn = Btn(cv, left + cw + 8, r1, cw, 26, "Nearest first",
                             self.toggle_order, f, dot=True)
        self.method_btn = Btn(cv, left + 2 * (cw + 8), r1, cw, 26,
                              "PrintWindow", self.toggle_method, f, dot=True)
        self.button_btn = Btn(cv, left, r2, cw, 26, "Left button",
                              self.cycle_button, f, dot=True)
        self.prize_btn = Btn(cv, left + cw + 8, r2, cw, 26, "Answer prizes",
                             self.toggle_prize, f, dot=True)
        self.return_btn = Btn(cv, left + 2 * (cw + 8), r2, cw, 26,
                              "Return to spot", self.toggle_return, f, dot=True)
        r3 = y + 358
        self.lag_btn = Btn(cv, left, r3, cw, 26, "Lag guard",
                           self.toggle_lag_guard, f, dot=True)
        self.me_btn = Btn(cv, left + cw + 8, r3, cw, 26, "Find me",
                          self.toggle_find_me, f, dot=True)
        self.alert_btn = Btn(cv, left + 2 * (cw + 8), r3, cw, 26, "Alerts",
                             self.toggle_alerts, f, dot=True)

        self.detect_note = text(cv, left, y + 396, text="", fill=C.INK_3,
                                font=self.font(9.5), anchor="nw",
                                width=int(s(inner)))

    def build_items(self, x, y, w, h):
        cv = self.cv
        left, right = x + 20, x + w - 20
        end = self.panel(x, y, w, h, "Items")
        self.items_note = self.label(end, y + 23, "", C.INK_3, size=9.5)
        f = self.font(9.5)
        Btn(cv, right - 96, y + 9, 96, 26, "Reset session", self.reset_counts,
            f)
        self.add_btn = Btn(cv, right - 96 - 8 - 84, y + 9, 84, 26,
                           "Add item \u25be", self.add_menu, f)

        self.item_rows = {}
        count = max(1, len(self.templates))
        per_col = 4
        cols = max(1, -(-count // per_col))
        if cols > 3:
            cols = 3
            per_col = -(-count // cols)
        gap = 22
        col_w = (right - left - gap * (cols - 1)) / float(cols)
        top, bottom = y + 56, y + h - 8
        step = min(27.0, (bottom - top) / float(per_col))
        wide = col_w >= 250
        for i, template in enumerate(self.templates):
            c, r = divmod(i, per_col)
            cx = left + c * (col_w + gap)
            cy = top + r * step + step / 2.0
            first = rect(cv, cx, cy - step / 2.0 + 1, col_w, step - 2,
                         fill=C.SURFACE, outline="")
            if c:
                vline(cv, cx - gap / 2.0, top + 2, bottom - 2, C.LINE)
            if r:
                hline(cv, cx, cx + col_w, cy - step / 2.0, C.LINE)
            btn = Btn(cv, cx, cy - 10, 20, 20, "",
                      lambda t=template: self.toggle_item(t), f, radius=4,
                      dot=True)
            cv.coords(btn.dot, s(cx + 6.5), s(cy - 3.5), s(cx + 13.5),
                      s(cy + 3.5))
            name_room = (cx + col_w - (92 if wide else 46) - 24) - (cx + 28)
            shown_name = self.fit(nice(template.name),
                                  self.font(10.5, "bold"), name_room)
            name = text(cv, cx + 28, cy, text=shown_name, fill=C.INK,
                        font=self.font(10.5, "bold"), anchor="w")
            rank = Btn(cv, cx + col_w - 38, cy - 10, 38, 20, "1st",
                       lambda t=template: self.promote(t),
                       self.font(8.5, numeric=True), radius=4)
            total = None
            if wide:
                total = self.readout(cx + col_w - 46, cy, "0", C.INK_3,
                                     size=9, anchor="e")
                sx = cx + col_w - 92
            else:
                sx = cx + col_w - 46
            session = self.readout(sx, cy, "0", C.INK, size=11.5,
                                   weight="bold", anchor="e")
            nx = cx + 28 + measure(self.font(10.5, "bold"), shown_name) + 8
            floor = self.readout(nx, cy, "", C.GOOD, size=8.5)
            marker_fits = nx + measure(self.font(8.5, numeric=True),
                                       "\u25cf 2 now") <= sx - 24
            self.item_rows[template.name] = {
                "name": name, "state": floor, "session": session,
                "total": total, "btn": btn, "rank": rank,
                "marker": marker_fits,
            }
            tag = "irow%d" % i
            for item in cv.find_all():
                if item >= first:
                    cv.addtag_withtag(tag, item)
            cv.tag_bind(tag, "<Button-3>",
                        lambda e, t=template: self.item_menu(t, e))
            cv.tag_bind(name, "<Button-1>",
                        lambda e, t=template: self.item_menu(t, e))
            cv.tag_bind(name, "<Enter>",
                        lambda e: self.cv.configure(cursor="hand2"))
            cv.tag_bind(name, "<Leave>",
                        lambda e: self.cv.configure(cursor=""))

    def build_activity(self, x, y, w, h):
        cv = self.cv
        left, right = x + 20, x + w - 20
        self.panel(x, y, w, h, "Activity")
        self.status_room = right - left
        self.status_line = text(cv, left, y + 64, text="Ready", fill=C.INK_2,
                                font=self.font(10.5), anchor="w")
        hline(cv, left, right, y + 80, C.LINE)
        self.event_rows = []
        for i in range(4):
            ry = y + 96 + i * 19
            when = self.readout(left, ry, "", C.INK_OFF, size=9)
            what = text(cv, left + 62, ry, text="", fill=C.INK_2,
                        font=self.font(9.5), anchor="w")
            self.event_rows.append((when, what))
        self.event_room = right - left - 62

    def build_footer(self, y):
        cv = self.cv
        pairs = [("F1", "start / stop"), ("F2", "look once"),
                 ("F3", "save image"), ("F5", "stop"), ("Ctrl + Wheel", "zoom")]
        f = self.font(9.5)
        cx = 24
        for key, name in pairs:
            cx += self.badge(cx, key, cy=y) + 7
            text(cv, cx, y, text=name, fill=C.INK_3, font=f, anchor="w")
            cx += measure(f, name) + 20
        self.legend_note = text(cv, W - 24, y, text="", fill=C.WARN, font=f,
                                anchor="e")


    def render_armed(self):
        live = self.armed.is_set()
        cv = self.cv
        if live:
            self.arm_btn.set_label("Stop")
            self.arm_btn.restyle(fill=C.SURFACE_2, hover=C.SURFACE_3,
                                 pressed=C.RAISED_LO, fg=C.INK,
                                 border=C.LINE_2)
            cv.itemconfigure(self.arm_key, fill=C.INK_3)
            cv.itemconfigure(self.state_dot, fill=C.GOOD)
            cv.itemconfigure(self.state_text, fill=C.INK,
                             text="Running" if self.auto_click else "Watching")
        else:
            self.arm_btn.set_label("Start")
            self.arm_btn.restyle(fill=C.INK, hover="#FFFFFF",
                                 pressed=C.INK_2, fg=C.BG, border=C.INK)
            cv.itemconfigure(self.arm_key, fill=C.INK_3)
            cv.itemconfigure(self.state_dot, fill=C.INK_OFF)
            cv.itemconfigure(self.state_text, fill=C.INK_2, text="Stopped")

    def render_live(self, fps=None):
        flowing = self.game_active and fps is not None and fps > 0.5
        colour = C.GOOD if flowing else C.INK_OFF
        self.cv.itemconfigure(self.live_dot, fill=colour)
        self.cv.itemconfigure(self.hud_live_dot, fill=colour)
        self.cv.itemconfigure(self.hud_live,
                              fill=C.INK if flowing else C.INK_3)
        since = time.monotonic() - self.last_look if self.last_look else None
        watching = since is not None and since <= WATCH_AT_LEAST + 1.0
        if watching:
            looks = self.looks_counted / max(0.2, time.perf_counter()
                                               - self.looks_since)
            self.looks_counted, self.looks_since = 0, time.perf_counter()
        if not self.game_active:
            label = "no game window"
        elif flowing:
            label = ("%d fps \u00b7 %d looks/s" % (round(fps), round(looks))
                     if watching else "%d fps \u00b7 not watching" % round(fps))
        else:
            label = "paused"
        self.cv.itemconfigure(self.live_text, text=label,
                              fill=C.INK_3 if watching or not flowing
                              else C.WARN)

    def render_window_status(self):
        if not self.game_active:
            self.cv.itemconfigure(self.window_lamp, fill=C.RED)
            self.cv.itemconfigure(self.window_value, text="Not found")
            self.cv.itemconfigure(self.preview_hint,
                                  text="Waiting for the game window")
            self.render_live(0)
            return
        self.cv.itemconfigure(self.window_lamp, fill=C.GOOD)
        self.cv.itemconfigure(self.window_value, text="Connected")

    def render_ping(self):
        ping = self.ping
        if ping is None:
            self.cv.itemconfigure(self.ping_dot, fill=C.INK_OFF)
            self.cv.itemconfigure(self.ping_value, text="\u2014", fill=C.INK_3)
            self.cv.itemconfigure(self.ping_note, text="overlay not showing",
                                  fill=C.INK_3)
            return
        colour = {PING_GOOD: C.GOOD, PING_SLOW: C.WARN, PING_BAD: C.RED}[
            ping.level]
        self.cv.itemconfigure(self.ping_dot, fill=colour)
        self.cv.itemconfigure(self.ping_value, text=PING_WORDS[ping.level],
                              fill=C.INK)
        note = ping.range_text
        if self.lag_hold:
            note = "holding clicks"
        self.cv.itemconfigure(self.ping_note, text=note,
                              fill=C.WARN if self.lag_hold else C.INK_3)

    def render_score_tick(self):
        x0, inner, top, tall = self.score_geom
        tx = x0 + inner * clamp(self.min_score, 0.0, 1.0)
        self.cv.coords(self.score_tick, s(tx) + self.ox, s(top - 4) + self.oy,
                       s(tx) + self.ox, s(top + tall + 4) + self.oy)

    def render_score_bar(self, value, ok):
        x0, inner, top, tall = self.score_geom
        value = clamp(value, 0.0, 1.0)
        self.cv.coords(self.score_fill, s(x0) + self.ox, s(top) + self.oy,
                       s(x0 + inner * value) + self.ox,
                       s(top + tall) + self.oy)
        if not ok:
            colour = C.RED
        elif value >= self.min_score:
            colour = C.GOOD
        else:
            colour = C.INK_3
        self.cv.itemconfigure(self.score_fill, fill=colour)

    def render_counts(self):
        self.cv.itemconfigure(self.rounds_value, text=commas(self.taken))
        spent = time.perf_counter() - self.session_start
        if spent >= 300 and self.taken:
            rate = self.taken * 3600.0 / spent
            self.cv.itemconfigure(self.rate_value, text="%.0f" % rate
                                  if rate >= 10 else "%.1f" % rate)
        else:
            self.cv.itemconfigure(self.rate_value, text="\u2014")
        self.cv.itemconfigure(self.claimed_value, text=commas(self.claimed),
                              fill=C.WARN if self.unclaimed else C.INK)
        missed = self.dropped + self.stolen
        self.cv.itemconfigure(self.missed_value, text=commas(missed),
                              fill=C.INK if not missed else C.INK_2)
        self.cv.itemconfigure(self.uptime_value, text=clock(spent))

    def render_settings(self):
        self.cv.itemconfigure(self.min_score_value, text="%.2f" % self.min_score)
        self.cv.itemconfigure(self.poll_value, text="%d" % self.capture_fps)
        self.cv.itemconfigure(self.click_every_value,
                              text="%g s" % self.click_every)
        self.cv.itemconfigure(self.patience_value, text="%g s" % self.patience)
        self.cv.itemconfigure(self.margin_value,
                              text="off" if not self.margin
                              else "%d px" % self.margin)
        self.cv.itemconfigure(self.spread_value,
                              text="off" if not self.spread
                              else "\u00b1%d px" % self.spread)
        self.cv.itemconfigure(self.prize_value, text="%.2f" % self.prize_score)
        self.cv.itemconfigure(self.prize_wait_value,
                              text="at once" if not self.prize_wait
                              else "%g s" % self.prize_wait)
        pairs = [(self.lag_btn, self.lag_guard, "Lag guard", "Lag guard off"),
                 (self.me_btn, self.find_me_on, "Find me", "Find me off"),
                 (self.alert_btn, self.alerts, "Alerts", "Alerts off"),
                 (self.prize_btn, self.answer_prize, "Answer prizes",
                  "Leave prizes"),
                 (self.return_btn, self.return_home, "Return to spot",
                  "Stay put"),
                 (self.auto_btn, self.auto_click, "Clicking on", "Watch only"),
                 (self.order_btn, self.nearest_first, "Nearest first",
                  "Best match")]
        for btn, on, yes, no in pairs:
            btn.set_label(yes if on else no)
            (self.select_on if on else self.select_off)(btn)
        self.method_btn.set_label("PrintWindow" if self.capture_method == "print"
                                  else "Screen grab")
        self.select_on(self.method_btn)
        self.button_btn.set_label("%s button" % self.button.capitalize())
        self.select_on(self.button_btn)
        self.render_score_tick()
        self.render_armed()

    def render_items(self):
        hits = list(self.hits)
        totals = self.stats.get("all_time", {})
        for template in self.templates:
            row = self.item_rows.get(template.name)
            if not row:
                continue
            on = template.enabled
            (self.select_on if on else self.select_off)(row["btn"])
            found = sum(1 for h in hits if h.name == template.name)
            self.cv.itemconfigure(row["name"], fill=(
                C.INK_OFF if not on else
                C.GOOD if found and not row["marker"] else C.INK))
            place = template.rank + 1
            row["rank"].set_label("%d%s" % (place, ORDINALS.get(place, "th")))
            row["rank"].set_enabled(on and template.rank > 0)
            found = sum(1 for h in hits if h.name == template.name)
            self.cv.itemconfigure(row["state"], text=(
                "\u25cf %d now" % found if found > 1 else "\u25cf now")
                if found and on and row["marker"] else "")
            self.cv.itemconfigure(
                row["session"],
                text=commas(self.session_counts.get(template.name, 0)),
                fill=C.INK if on else C.INK_3)
            if row["total"] is not None:
                self.cv.itemconfigure(row["total"],
                                      text=commas(totals.get(template.name, 0)))
        notes = ["click a name for options"]
        odd = self.session_counts.get("unidentified", 0)
        if odd:
            notes.append("+%d unidentified" % odd)
        shelved = len(self.shelf)
        if shelved:
            notes.append("%d spot%s skipped"
                         % (shelved, "" if shelved == 1 else "s"))
        home = self.home_item()
        if home is not None and self.return_home:
            n = self.spawns.count(home.name)
            notes.append("%d spawn%s mapped" % (n, "" if n == 1 else "s"))
        self.cv.itemconfigure(self.items_note, text=self.fit(
            "  \u00b7  ".join(notes), self.font(9.5), 340))


    def menu(self):
        return tk.Menu(self.root, tearoff=0, bg=C.SURFACE_2, fg=C.INK,
                       activebackground=C.SURFACE_3, activeforeground=C.INK,
                       disabledforeground=C.INK_3, bd=0, relief="flat",
                       font=self.font(10))

    def popup(self, menu, event=None):
        if event is not None:
            x, y = event.x_root, event.y_root
        else:
            x, y = self.root.winfo_pointerx(), self.root.winfo_pointery()
        try:
            menu.tk_popup(x, y)
        finally:
            menu.grab_release()

    def item_menu(self, template, event=None):
        if self.by_name.get(template.name) is not template:
            return
        m = self.menu()
        m.add_command(label=nice(template.name), state="disabled")
        m.add_separator()
        m.add_command(label="Turn off" if template.enabled else "Turn on",
                      command=lambda: self.toggle_item(template))
        first, last = template.rank == 0, \
            template.rank == len(self.templates) - 1
        m.add_command(label="Move to top", state="disabled" if first
                      else "normal",
                      command=lambda: self.move_item(template, "top"))
        m.add_command(label="Move up", state="disabled" if first else "normal",
                      command=lambda: self.move_item(template, -1))
        m.add_command(label="Move down", state="disabled" if last
                      else "normal",
                      command=lambda: self.move_item(template, 1))
        m.add_command(label="Move to bottom", state="disabled" if last
                      else "normal",
                      command=lambda: self.move_item(template, "bottom"))
        m.add_separator()
        m.add_command(label="Rename…",
                      command=lambda: self.rename_item(template))
        m.add_command(label="Look for it now",
                      command=lambda: self.look_for(template))
        m.add_command(label="Show its picture",
                      command=lambda: self.show_picture(template))
        m.add_separator()
        m.add_command(label="Remove…",
                      command=lambda: self.remove_item(template))
        self.popup(m, event)

    def add_menu(self):
        m = self.menu()
        m.add_command(label="From the live view — drag a box…",
                      command=self.start_crop)
        m.add_command(label="From a picture file…", command=self.add_item)
        if self.removed:
            back = self.menu()
            for name in sorted(self.removed, key=nice):
                back.add_command(label=nice(name),
                                 command=lambda n=name: self.restore_item(n))
            m.add_separator()
            m.add_cascade(label="Bring back a removed item", menu=back)
        self.popup(m)

    def move_item(self, template, where):
        order = [t.name for t in self.templates]
        order.remove(template.name)
        if where == "top":
            order.insert(0, template.name)
        elif where == "bottom":
            order.append(template.name)
        else:
            order.insert(clamp(template.rank + where, 0, len(order)),
                         template.name)
        self.set_order(order)
        self.render_items()
        self.save_config_soon()

    def rename_item(self, template):
        name = simpledialog.askstring(
            "Rename", "What should %s be called on the panel?\n"
            "(Leave it empty to go back to the original name.)"
            % nice(template.name), parent=self.root,
            initialvalue=nice(template.name))
        if name is None:
            return
        name = " ".join(name.split())
        if name and name != DISPLAY_NAMES.get(template.name,
                                              template.name.title()):
            CUSTOM_NAMES[template.name] = name
        else:
            CUSTOM_NAMES.pop(template.name, None)
        self.save_config_soon()
        self.rebuild()

    def item_files(self, name, folder):
        try:
            entries = os.listdir(folder)
        except OSError:
            return []
        out = []
        for entry in entries:
            stem, ext = os.path.splitext(entry)
            if ext.lower() in (".png", ".bmp", ".jpg", ".jpeg") and \
                    stem in (name, name + "_mask", name + "_prize",
                             name + "_prize_mask"):
                out.append(entry)
        return out

    def remove_item(self, template):
        name = template.name
        own = self.item_files(name, ITEM_DIR)
        builtin = name in builtin_names()
        if own and builtin:
            what = ("Your picture of it is put aside, and the built-in one is "
                    "hidden too.")
        elif own:
            what = "Its picture is put aside in items\\removed."
        else:
            what = "It is built in, so it is only hidden."
        if not messagebox.askyesno(
                "Remove item",
                "Remove %s from the panel?\n\n%s It stops being looked for. "
                "Its catch counts are kept, and you can bring it back from "
                "Add item ▾." % (nice(name), what), parent=self.root):
            return
        try:
            if own:
                os.makedirs(REMOVED_DIR, exist_ok=True)
            for entry in own:
                target = os.path.join(REMOVED_DIR, entry)
                if os.path.exists(target):
                    os.remove(target)
                shutil.move(os.path.join(ITEM_DIR, entry), target)
        except Exception as e:
            self.status("Could not move its picture: %s" % e, C.RED)
            return
        self.removed.add(name)
        if self.chase is not None and self.chase.name == name:
            self.chase = None
        self.reload_items()
        self.event("Removed %s — bring it back from Add item."
                   % nice(name), C.INK_2)

    def restore_item(self, name):
        try:
            for entry in self.item_files(name, REMOVED_DIR):
                target = os.path.join(ITEM_DIR, entry)
                if not os.path.exists(target):
                    shutil.move(os.path.join(REMOVED_DIR, entry), target)
        except Exception as e:
            self.status("Could not bring its picture back: %s" % e, C.RED)
            return
        self.removed.discard(name)
        self.reload_items()
        if name in self.by_name:
            self.event("Brought back %s." % nice(name), C.GOOD)
        else:
            self.status("%s has no picture left to bring back." % nice(name),
                        C.WARN)

    def look_for(self, template):
        def work():
            frame = self.current_frame()
            if frame is None:
                self.status("No game picture to look at.", C.RED)
                return
            started = time.perf_counter()
            hits = find_items(frame, [template], threshold=0.5,
                              margin=self.margin, everything=True)
            took = (time.perf_counter() - started) * 1000
            if not hits:
                self.status("No %s on the floor right now (%.0f ms)."
                            % (nice(template.name), took), C.INK_2)
                return
            best = hits[0]
            ok = best.score >= self.min_score
            if self.chase is None:
                self.verdict_state = (hits[:4], None)
            self.status("%s: best match %.2f at %d, %d — %s (%.0f ms)."
                        % (nice(template.name), best.score, best.click[0],
                           best.click[1], "would be picked up" if ok else
                           "below the %.2f needed" % self.min_score, took),
                        C.GOOD if ok else C.WARN)
        threading.Thread(target=work, daemon=True).start()


    def dialog(self, title):
        win = tk.Toplevel(self.root)
        win.title(title)
        win.configure(bg=C.SURFACE)
        win.resizable(False, False)
        win.transient(self.root)

        def centre():
            try:
                win.update_idletasks()
                x = self.root.winfo_rootx() + (self.root.winfo_width()
                                               - win.winfo_reqwidth()) // 2
                y = self.root.winfo_rooty() + (self.root.winfo_height()
                                               - win.winfo_reqheight()) // 3
                win.geometry("+%d+%d" % (max(0, x), max(0, y)))
            except Exception:
                pass
        win.after_idle(centre)
        return win

    def photo_of(self, bgr, zoom):
        big = cv2.resize(bgr, None, fx=zoom, fy=zoom,
                         interpolation=cv2.INTER_NEAREST)
        rgb = cv2.cvtColor(big, cv2.COLOR_BGR2RGB)
        h, w = rgb.shape[:2]
        photo = tk.PhotoImage(data=("P6\n%d %d\n255\n" % (w, h)).encode(
            "ascii") + rgb.tobytes())
        self._dialog_photos = (self._dialog_photos + [photo])[-8:]
        return photo

    def item_sheet(self, image, mask, click):
        tint = image.copy()
        on = mask > 128
        tint[on] = (0.45 * tint[on] + np.array(_bgr(C.GOOD)) * 0.55).astype(
            np.uint8)
        x, y = int(round(click[0])), int(round(click[1]))
        if 0 <= y < tint.shape[0] and 0 <= x < tint.shape[1]:
            cv2.circle(tint, (x, y), 2, _bgr(C.RED), -1)
        gap = np.zeros((image.shape[0], 4, 3), np.uint8)
        gap[:] = _bgr(C.SURFACE)
        return np.hstack([image, gap, tint])

    def zoom_for(self, image):
        h, w = image.shape[:2]
        return max(1, min(6, int(260 / max(h, w * 2 + 4))))

    def show_picture(self, template):
        win = self.dialog(nice(template.name))
        sheet = self.item_sheet(template.image, template.mask, template.click)
        photo = self.photo_of(sheet, self.zoom_for(template.image))
        tk.Label(win, image=photo, bg=C.SURFACE).pack(padx=16, pady=(16, 8))
        lines = ["Left: the picture it looks for.  Right: the part that "
                 "counts (green) and where it clicks (red)."]
        if template.prize is not None:
            lines.append("It has its own picture for the prize popup too.")
        tk.Label(win, text="\n".join(lines), bg=C.SURFACE, fg=C.INK_2,
                 font=self.font(9.5), justify="left",
                 wraplength=int(s(320))).pack(padx=16)
        tk.Button(win, text="Close", command=win.destroy, bg=C.SURFACE_2,
                  fg=C.INK, activebackground=C.SURFACE_3, relief="flat",
                  font=self.font(10), padx=14).pack(pady=12)
        win.bind("<Escape>", lambda e: win.destroy())

    def confirm_new_item(self, image, mask, click, suggested=""):
        win = self.dialog("Add item")
        photo = self.photo_of(self.item_sheet(image, mask, click),
                              self.zoom_for(image))
        tk.Label(win, image=photo, bg=C.SURFACE).pack(padx=16, pady=(16, 6))
        tk.Label(win, text="Green is what it will look for; red is where it "
                 "will click. If the green spills onto the floor, cancel and "
                 "drag a tighter box.", bg=C.SURFACE, fg=C.INK_2,
                 font=self.font(9.5), justify="left",
                 wraplength=int(s(320))).pack(padx=16)
        row = tk.Frame(win, bg=C.SURFACE)
        row.pack(fill="x", padx=16, pady=(12, 0))
        tk.Label(row, text="Name", bg=C.SURFACE, fg=C.INK_3,
                 font=self.font(9.5)).pack(side="left")
        entry = tk.Entry(row, bg=C.SUNK, fg=C.INK, insertbackground=C.INK,
                         relief="flat", font=self.font(10.5))
        entry.pack(side="left", fill="x", expand=True, padx=(8, 0), ipady=3)
        entry.insert(0, suggested)
        entry.focus_set()
        answer = {"name": None}

        def ok(_e=None):
            name = " ".join(entry.get().lower().split())
            name = "".join(ch for ch in name if ch not in '\\/:*?"<>|')
            if not name or name.endswith(("_mask", "_prize")):
                entry.configure(bg=C.RED_DIM)
                return
            answer["name"] = name
            win.destroy()

        buttons = tk.Frame(win, bg=C.SURFACE)
        buttons.pack(pady=12)
        for label, cmd in (("Add", ok), ("Cancel", win.destroy)):
            tk.Button(buttons, text=label, command=cmd, bg=C.SURFACE_2,
                      fg=C.INK, activebackground=C.SURFACE_3, relief="flat",
                      font=self.font(10), padx=14).pack(side="left", padx=4)
        win.bind("<Return>", ok)
        win.bind("<Escape>", lambda e: win.destroy())
        win.grab_set()
        self.root.wait_window(win)
        return answer["name"]


    def add_item(self):
        path = filedialog.askopenfilename(
            parent=self.root, title="Picture of the new item",
            filetypes=[("Pictures", "*.png *.jpg *.jpeg *.bmp"),
                       ("All files", "*.*")])
        if not path:
            return
        try:
            image = cv2.imdecode(np.fromfile(path, dtype=np.uint8),
                                 cv2.IMREAD_COLOR)
        except Exception:
            image = None
        if image is None:
            self.status("That picture could not be read.", C.RED)
            return
        stem = os.path.splitext(os.path.basename(path))[0]
        self.learn_item(image, stem.replace("_", " ").replace("-", " ").strip())

    def learn_item(self, image, suggested=""):
        prepared = prepare_item(image)
        if prepared is None:
            self.status("Could not tell the item from its background — "
                        "try a box with a little floor round the item.",
                        C.RED)
            return
        image, mask, click = prepared
        name = self.confirm_new_item(image, mask, click, suggested)
        if not name:
            return
        if name in self.by_name and not messagebox.askyesno(
                "Add item", "There is already a %s. Replace its picture with "
                "this one?" % nice(name), parent=self.root):
            return
        try:
            os.makedirs(ITEM_DIR, exist_ok=True)
            for suffix, data in (("", image), ("_mask", mask)):
                ok, buf = cv2.imencode(".png", data)
                if not ok:
                    raise ValueError("could not encode it")
                buf.tofile(os.path.join(ITEM_DIR, name + suffix + ".png"))
        except Exception as e:
            self.status("Could not save the picture: %s" % e, C.RED)
            return
        self.removed.discard(name)
        self.reload_items()
        self.event("Added %s — it is now looked for." % nice(name),
                   C.GOOD)


    def start_crop(self):
        frame = self.live_frame if self.live_frame is not None \
            else self.last_frame
        if frame is None or self.preview_geom is None:
            self.status("No game picture yet — open the game first.",
                        C.WARN)
            return
        self.close_profile_card()
        self.crop_mode = True
        self.crop_frame = frame.copy()
        self.crop_start = None
        self.show_preview(self.crop_frame, ([], None))
        bx, by, bw, bh = self.preview_box
        cv = self.cv
        rounded(cv, bx + bw / 2 - 170, by + bh - 34, 340, 24, 5, fill=C.SUNK,
                outline=C.LINE_2, width=max(1, s(1)), tags=("cropui",))
        text(cv, bx + bw / 2, by + bh - 22,
             text="Drag a box round the item  ·  Esc to cancel",
             fill=C.INK, font=self.font(9.5), anchor="center",
             tags=("cropui",))
        if self.ox or self.oy:
            cv.move("cropui", self.ox, self.oy)
        cv.configure(cursor="crosshair")
        cv.bind("<ButtonPress-1>", self.crop_press)
        cv.bind("<B1-Motion>", self.crop_drag)
        cv.bind("<ButtonRelease-1>", self.crop_release)
        self.status("Drag a box round the item on the live view.", C.INK)

    def end_crop(self):
        self.crop_mode = False
        self.crop_start = None
        try:
            self.cv.delete("cropui")
            self.cv.delete("cropbox")
            for seq in ("<ButtonPress-1>", "<B1-Motion>", "<ButtonRelease-1>"):
                self.cv.unbind(seq)
            self.cv.configure(cursor="")
        except Exception:
            pass
        self.live_shown = -1

    def _on_picture(self, event):
        gx, gy, gw, gh, _f = self.preview_geom
        x = clamp(self.cv.canvasx(event.x), gx, gx + gw)
        y = clamp(self.cv.canvasy(event.y), gy, gy + gh)
        return x, y

    def crop_press(self, event):
        gx, gy, gw, gh, _f = self.preview_geom
        x, y = self.cv.canvasx(event.x), self.cv.canvasy(event.y)
        if not (gx <= x <= gx + gw and gy <= y <= gy + gh):
            return
        self.crop_start = (x, y)
        self.cv.delete("cropbox")
        self.cv.create_rectangle(x, y, x, y, outline=C.GOOD, dash=(4, 3),
                                 width=max(1, s(1.5)), tags=("cropbox",))

    def crop_drag(self, event):
        if self.crop_start is None:
            return
        x, y = self._on_picture(event)
        self.cv.coords("cropbox", self.crop_start[0], self.crop_start[1], x, y)

    def crop_release(self, event):
        if self.crop_start is None:
            return
        x1, y1 = self._on_picture(event)
        x0, y0 = self.crop_start
        gx, gy, _gw, _gh, factor = self.preview_geom
        frame = self.crop_frame
        self.end_crop()
        fx0, fx1 = sorted(((x0 - gx) / factor, (x1 - gx) / factor))
        fy0, fy1 = sorted(((y0 - gy) / factor, (y1 - gy) / factor))
        if fx1 - fx0 < 8 or fy1 - fy0 < 8:
            self.status("That box was too small — Add item to try again.",
                        C.WARN)
            return
        pad = int(0.12 * max(fx1 - fx0, fy1 - fy0)) + 2
        fh, fw = frame.shape[:2]
        crop = frame[max(0, int(fy0) - pad):min(fh, int(fy1) + pad),
                     max(0, int(fx0) - pad):min(fw, int(fx1) + pad)]
        self.learn_item(np.ascontiguousarray(crop))

    def on_escape(self, _event=None):
        if self.crop_mode:
            self.end_crop()
            self.status("Cancelled.", C.INK_2)
        else:
            self.close_profile_card()


    def alert(self, key, message, every=20.0):
        if not self.alerts:
            return
        now = time.monotonic()
        if now - self._alerted.get(key, -1e9) < every:
            return
        self._alerted[key] = now
        try:
            import winsound
            winsound.MessageBeep(0x30)
        except Exception:
            pass
        try:
            hwnd = self.parent_hwnd or self.root.winfo_id()
            win32gui.FlashWindowEx(hwnd, 0x3 | 0xC, 0, 0)
        except Exception:
            pass
        self.event(message, C.WARN)

    def toggle_alerts(self):
        self.alerts = not self.alerts
        self.render_settings()
        self.cv.itemconfigure(
            self.detect_note,
            text="Beeps and flashes the taskbar button when it needs you: a "
                 "prize popup it cannot answer for sure, or the game window "
                 "going away while it is running.")
        self.save_config_soon()

    def reload_items(self):
        order = [t.name for t in self.templates]
        off = {t.name for t in self.templates if not t.enabled}
        self.templates = [t for t in gather_templates()
                          if t.name not in self.removed]
        self.by_name = {t.name: t for t in self.templates}
        for template in self.templates:
            template.enabled = template.name not in off
        self.set_order(order)
        self.hits = []
        self.chase = None
        self.pending = None
        self.verdict_state = ([], None)
        self.save_config_soon()
        self.rebuild()

    def render_events(self):
        rows = getattr(self, "event_rows", None)
        if not rows:
            return
        recent = list(reversed(self.events[-len(rows):]))
        f = self.font(9.5)
        for i, (when_item, what_item) in enumerate(rows):
            if i < len(recent):
                when, what, colour = recent[i]
                self.cv.itemconfigure(when_item, text=when)
                self.cv.itemconfigure(what_item, fill=colour if i == 0
                                      else C.INK_3,
                                      text=self.fit(what, f, self.event_room))
            else:
                self.cv.itemconfigure(when_item, text="")
                self.cv.itemconfigure(what_item, text="")

    def render_target(self, chase):
        if chase is None:
            live = list(self.hits)
            if live:
                headline = "%d item%s in sight" % (len(live),
                                                   "" if len(live) == 1 else "s")
                colour = C.INK
                detail = "   ".join("%s %.2f" % (nice(h.name), h.score)
                                    for h in live[:4])
                best = max(h.score for h in live)
            elif self.pending is not None:
                headline, colour = "Waiting for the prize popup", C.INK
                detail = "the %s has gone" % nice(self.pending[0].name)
                best = 0.0
            else:
                headline, colour = "Nothing on the floor", C.INK_2
                detail = "watching for something to drop"
                best = 0.0
            self.cv.itemconfigure(self.verdict_value, text=headline, fill=colour)
            self.cv.itemconfigure(self.verdict_note, text=detail)
            self.cv.itemconfigure(self.best_value,
                                  text="\u2014" if not best else "%.2f" % best,
                                  fill=C.INK if best else C.INK_OFF)
            self.cv.itemconfigure(self.best_note, text="needs %.2f"
                                  % self.min_score)
            self.render_score_bar(best, True)
            return
        hit = chase.hit
        x, y = hit.click
        self.cv.itemconfigure(self.verdict_value,
                              text="Going for the %s" % nice(chase.name),
                              fill=C.INK)
        self.cv.itemconfigure(
            self.verdict_note,
            text="clicking %d, %d \u00b7 %d click%s \u00b7 %.1fs so far"
                 % (x, y, chase.clicks, "" if chase.clicks == 1 else "s",
                    chase.age))
        self.cv.itemconfigure(self.best_value, text="%.2f" % hit.score,
                              fill=C.INK)
        self.cv.itemconfigure(self.best_note,
                              text="needs %.2f \u00b7 drawn at %.2f\u00d7"
                                   % (self.min_score, hit.scale))
        self.render_score_bar(hit.score, True)

    def show_preview(self, frame, verdict=None):
        self.last_frame = frame
        self.last_verdict = verdict
        if frame is None or frame.size == 0:
            return
        bx, by, bw, bh = self.preview_box
        fh, fw = frame.shape[:2]
        factor = min(bw * SCALE / float(fw), bh * SCALE / float(fh))
        factor = max(0.05, factor)
        shown = cv2.resize(frame, (max(1, int(fw * factor)),
                                   max(1, int(fh * factor))),
                           interpolation=cv2.INTER_AREA)
        shown = self.annotate(shown, verdict, factor)
        rgb = cv2.cvtColor(shown, cv2.COLOR_BGR2RGB)
        h, w = rgb.shape[:2]
        ppm = ("P6\n%d %d\n255\n" % (w, h)).encode("ascii") + rgb.tobytes()
        try:
            photo = tk.PhotoImage(data=ppm)
        except Exception as e:
            log("Preview failed: %s" % e, key="preview")
            return
        cx = s(bx + bw / 2.0) + self.ox
        cy = s(by + bh / 2.0) + self.oy
        if self._preview_item is None:
            self._preview_item = self.cv.create_image(cx, cy, image=photo,
                                                      anchor="center")
            self.cv.tag_raise("hud")
            if self.card_open:
                self.cv.tag_raise("pcard")
            self.cv.itemconfigure(self.preview_hint, text="")
        else:
            self.cv.itemconfigure(self._preview_item, image=photo)
        self._photo = photo
        self.preview_geom = (cx - w / 2.0, cy - h / 2.0, w, h, factor)

    def annotate(self, img, state, factor=1.0):
        img = img.copy()
        hits, chase = state if state else ([], None)
        k = float(factor)
        good, seen = _bgr(C.GOOD), _bgr(C.INK)
        plate, ink = _bgr(C.SUNK), _bgr(C.INK)

        def at(p):
            return int(round(p[0] * k)), int(round(p[1] * k))

        if self.margin:
            m = int(self.margin * k)
            cv2.rectangle(img, (m, m), (img.shape[1] - m, img.shape[0] - m),
                          (90, 90, 90), 1, cv2.LINE_AA)

        full = (int(img.shape[1] / k), int(img.shape[0] / k))

        now = time.monotonic()
        seen = self.me is not None and now - self.me[2] < ME_TRUST
        sent = self.me_walk is not None and now - self.me_walk[2] < 20.0
        if seen or sent:
            px, py = at(self.player_point(full))
            mark = _bgr(C.INK) if seen else _bgr(C.INK_3)
            cv2.ellipse(img, (px, py), (11, 5), 0, 0, 360, mark, 1,
                        cv2.LINE_AA)
            cv2.putText(img, "you", (px - 9, py + 16),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.36, mark, 1, cv2.LINE_AA)

        spot = self.home_spot(full) if self.return_home else None
        if spot is not None and spot[2] >= SPAWN_MIN:
            hx, hy = at(spot[:2])
            r = 7
            cv2.circle(img, (hx, hy), r, good, 1, cv2.LINE_AA)
            cv2.line(img, (hx - r - 4, hy), (hx - r + 2, hy), good, 1)
            cv2.line(img, (hx + r - 2, hy), (hx + r + 4, hy), good, 1)
            cv2.putText(img, "home", (hx - 14, hy + r + 12),
                        cv2.FONT_HERSHEY_SIMPLEX, 0.36, good, 1, cv2.LINE_AA)

        for hit in hits:
            x, y, w, h = hit.box
            x0, y0 = at((x, y))
            x1, y1 = at((x + w, y + h))
            chased = chase is not None and (chase.hit is hit or
                                            chase.hit.overlaps(hit, 0.5))
            colour = good if chased else seen
            cv2.rectangle(img, (x0 - 2, y0 - 2), (x1 + 2, y1 + 2), colour, 1,
                          cv2.LINE_AA)
            tag = "%s %.2f" % (nice(hit.name), hit.score)
            fs = 0.36
            (tw, th), _b = cv2.getTextSize(tag, cv2.FONT_HERSHEY_SIMPLEX, fs, 1)
            ty = max(th + 5, y0 - 6)
            cv2.rectangle(img, (x0 - 2, ty - th - 4), (x0 + tw + 4, ty + 3),
                          plate, -1)
            cv2.putText(img, tag, (x0 + 1, ty), cv2.FONT_HERSHEY_SIMPLEX, fs,
                        colour if chased else ink, 1, cv2.LINE_AA)
        if chase is not None:
            cx, cy = at(chase.hit.click)
            cv2.circle(img, (cx, cy), 8, good, 1, cv2.LINE_AA)
            cv2.circle(img, (cx, cy), 2, good, -1, cv2.LINE_AA)
        return img


    def find_game_window(self):
        found = []

        def collect(hwnd, out):
            if hwnd in self.own_hwnds or not win32gui.IsWindowVisible(hwnd):
                return
            title = win32gui.GetWindowText(hwnd)
            if not title:
                return
            low = title.lower()
            if "clicker" in low or "watcher" in low or "grabber" in low \
                    or "auto event" in low:
                return
            if title == "BoomBang":
                out.append((0, hwnd))
            elif "BoomBang Launcher" in title:
                out.append((2, hwnd))
            elif "boombang" in low:
                out.append((1, hwnd))

        try:
            win32gui.EnumWindows(collect, found)
        except Exception as e:
            log("Window enumeration failed: %s" % e)
            return None
        if not found:
            return None
        found.sort(key=lambda p: p[0])
        return found[0][1]

    def client_size(self, hwnd):
        try:
            _, _, cw, ch = win32gui.GetClientRect(hwnd)
            return cw, ch
        except Exception:
            return None

    def watchdog_loop(self):
        while not self.shutting_down:
            try:
                hwnd = self.game_hwnd
                alive = False
                if hwnd:
                    try:
                        alive = bool(win32gui.IsWindow(hwnd))
                    except Exception:
                        alive = False
                if not alive:
                    hwnd = self.find_game_window()
                    self.game_hwnd = hwnd

                size = self.client_size(hwnd) if hwnd else None
                key = (hwnd, size)
                if key != self._target_key:
                    self._target_key = key
                    self._target_cache = {}

                if not hwnd:
                    self.client_now = None
                elif size and size[0] > 0 and size[1] > 0:
                    if tuple(size) != self.client_now:
                        self.client_now = tuple(size)
                        self.ui(self.render_window_status)

                was_active = self.game_active
                self.game_active = hwnd is not None
                if was_active and not self.game_active:
                    self.chase = None
                    self.pending = None
                    self.awaiting = None
                    self.prize_state = None
                    self.hits = []
                    self.shelf.clear()
                    self.attempts.clear()
                    self.ui(self.render_items)
                    self.ui(self.render_target, None)
                    if self.armed.is_set():
                        self.armed.clear()
                        self.ui(self.render_armed)
                        self.alert("game", "The game window went away.")
                        self.status("Disarmed: the game window went away.",
                                    C.RED)
                if was_active != self.game_active:
                    self.ui(self.render_window_status)
            except Exception as e:
                log("Watchdog error: %s" % e)
            if self.closing.wait(0.5):
                return


    def client_origin_offset(self, hwnd):
        wl, wt, _, _ = win32gui.GetWindowRect(hwnd)
        cl, ct = win32gui.ClientToScreen(hwnd, (0, 0))
        return cl - wl, ct - wt

    def print_window(self, hwnd):
        try:
            l, t, r, b = win32gui.GetWindowRect(hwnd)
        except Exception:
            return None
        ww, wh = r - l, b - t
        if ww <= 0 or wh <= 0 or ww * wh > 40000000:
            return None

        hdc_src = None
        try:
            hdc_src = _user32.GetWindowDC(hwnd)
            if not hdc_src:
                return None
            pw = self._pw
            if pw is None or pw["size"] != (ww, wh) or pw["hwnd"] != hwnd:
                self.free_capture()
                hdc_mem = _gdi32.CreateCompatibleDC(hdc_src)
                if hdc_mem:
                    _gdi32.SetLayout(hdc_mem, 0)
                hbm = _gdi32.CreateCompatibleBitmap(hdc_src, ww, wh)
                if not hdc_mem or not hbm:
                    if hbm:
                        _gdi32.DeleteObject(hbm)
                    if hdc_mem:
                        _gdi32.DeleteDC(hdc_mem)
                    return None
                info = _BITMAPINFO()
                info.bmiHeader.biSize = ctypes.sizeof(_BITMAPINFOHEADER)
                info.bmiHeader.biWidth = ww
                info.bmiHeader.biHeight = -wh
                info.bmiHeader.biPlanes = 1
                info.bmiHeader.biBitCount = 32
                info.bmiHeader.biCompression = 0
                pw = self._pw = {"size": (ww, wh), "hwnd": hwnd,
                                 "dc": hdc_mem, "bmp": hbm, "info": info,
                                 "buf": (ctypes.c_ubyte * (ww * wh * 4))()}
            old = _gdi32.SelectObject(pw["dc"], pw["bmp"])
            try:
                if not _user32.PrintWindow(hwnd, pw["dc"], 2):
                    _user32.PrintWindow(hwnd, pw["dc"], 0)
            finally:
                _gdi32.SelectObject(pw["dc"], old)
            got = _gdi32.GetDIBits(pw["dc"], pw["bmp"], 0, wh, pw["buf"],
                                   ctypes.byref(pw["info"]), 0)
            if not got:
                return None
            arr = np.frombuffer(pw["buf"], dtype=np.uint8).reshape(wh, ww, 4)
            return np.ascontiguousarray(arr[:, :, :3])
        except Exception as e:
            log("PrintWindow failed: %s" % e, key="printwindow", every=30)
            return None
        finally:
            if hdc_src:
                try:
                    _user32.ReleaseDC(hwnd, hdc_src)
                except Exception:
                    pass

    def free_capture(self):
        pw, self._pw = self._pw, None
        if pw is None:
            return
        try:
            _gdi32.DeleteObject(pw["bmp"])
            _gdi32.DeleteDC(pw["dc"])
        except Exception:
            pass

    def screen_grab(self, hwnd, x, y, w, h):
        try:
            sx, sy = win32gui.ClientToScreen(hwnd, (int(x), int(y)))
        except Exception:
            return None
        try:
            sct = getattr(self._sct_local, "sct", None)
            if sct is None:
                sct = self._sct_local.sct = mss.mss()
            shot = sct.grab({"left": int(sx), "top": int(sy),
                             "width": int(w), "height": int(h)})
            return cv2.cvtColor(np.array(shot), cv2.COLOR_BGRA2BGR)
        except Exception as e:
            self._sct_local.sct = None
            log("Screen grab failed: %s" % e, key="grab", every=30)
            return None

    def grab(self, wait=None):
        if wait is None:
            self._cap_lock.acquire()
        elif not self._cap_lock.acquire(timeout=wait):
            return None
        try:
            return self._grab()
        finally:
            self._cap_lock.release()

    def _grab(self):
        hwnd = self.game_hwnd
        if not hwnd:
            return None
        size = self.client_size(hwnd)
        if not size or size[0] < 8 or size[1] < 8:
            return None
        w, h = size
        if self.capture_method == "print":
            shot = self.print_window(hwnd)
            if shot is not None:
                try:
                    ox, oy = self.client_origin_offset(hwnd)
                except Exception:
                    ox = oy = 0
                if self.window_mirrored(hwnd, shot, ox, oy, w, h):
                    shot = shot[:, ::-1]
                crop = shot[oy:oy + h, ox:ox + w]
                if crop.size and crop.shape[0] == h and crop.shape[1] == w \
                        and int(crop.max()) > 8:
                    self.capture_used = "print"
                    return np.ascontiguousarray(crop)
            self.capture_used = "fallback"
        else:
            self.capture_used = "screen"
        return self.screen_grab(hwnd, 0, 0, w, h)

    def window_mirrored(self, hwnd, shot, ox, oy, w, h):
        state = self._mirror.get(hwnd)
        if state is None:
            state = self._mirror[hwnd] = {"flip": False, "sure": False,
                                          "checked": 0.0}
        now = time.monotonic()
        if state["sure"] or now - state["checked"] < 3.0:
            return state["flip"]
        state["checked"] = now
        screen = self.screen_grab(hwnd, 0, 0, w, h)
        if screen is None or screen.shape[:2] != (h, w):
            return state["flip"]

        def small(img):
            img = cv2.resize(img, (160, 90), interpolation=cv2.INTER_AREA)
            return cv2.cvtColor(img, cv2.COLOR_BGR2GRAY).astype(np.float32)

        def alike(a, b):
            return float(cv2.matchTemplate(a, b, cv2.TM_CCOEFF_NORMED)[0, 0])

        seen = small(screen)
        straight = alike(seen, small(shot[oy:oy + h, ox:ox + w]))
        turned = alike(seen, small(shot[:, ::-1][oy:oy + h, ox:ox + w]))
        best = max(straight, turned)
        if best < 0.6 or abs(straight - turned) < 0.15:
            return state["flip"]
        flip = turned > straight
        state["sure"] = True
        if flip != state["flip"] or flip:
            log("PrintWindow picture is %s (straight %.2f, flipped %.2f)"
                % ("mirrored, turning it round" if flip else "the right way",
                   straight, turned))
        state["flip"] = flip
        return flip

    def toggle_armed(self):
        if self.armed.is_set():
            self.armed.clear()
            self.home_plan = []
            self.event("Stopped.", C.INK_2)
        else:
            if not self.game_active:
                self.status("No game window to watch.", C.RED)
                return
            if not any(t.enabled for t in self.templates):
                self.status("Every item is turned off.", C.WARN)
                return
            self.armed.set()
            self.event("Started \u2014 watching the floor.", C.GOOD)
        self.idle_shown = 0.0
        self.ui(self.render_armed)

    def disarm(self):
        if self.armed.is_set():
            self.armed.clear()
            self.home_plan = []
            self.ui(self.render_armed)
            self.event("Stopped.", C.INK_2)


    CARD_STALE = 10 * 60

    def open_profile_card(self, _event=None):
        if not self.profile:
            self.status("No profile yet \u2014 set \"profile\" in "
                        "item_config.json, or check the connection.", C.WARN)
            return
        self.close_profile_card()
        self.card_open = True
        self.draw_profile_card()
        age = time.time() - (self.profile.get("fetched") or 0)
        if age > self.CARD_STALE and not self.card_fetching:
            self.card_fetching = True
            self.draw_profile_card()
            threading.Thread(target=self.profile_loop, kwargs={"force": True},
                             daemon=True).start()

    def close_profile_card(self, _event=None):
        self.card_open = False
        try:
            self.cv.delete("pcard")
        except Exception:
            pass

    def draw_profile_card(self):
        cv = self.cv
        cv.delete("pcard")
        if not self.card_open:
            return
        profile = self.profile or {}
        stats = profile.get("stats") or {}
        cw, ch = int(cv.winfo_width()), int(cv.winfo_height())
        veil = cv.create_rectangle(0, 0, max(cw, s(W) + 2 * self.ox),
                                   max(ch, s(H) + 2 * self.oy), fill=C.BG,
                                   stipple="gray50", outline="",
                                   tags=("pcard",))
        cv.tag_bind(veil, "<Button-1>", self.close_profile_card)
        first = cv.create_line(0, 0, 0, 0, fill="")

        x, y, w, h = (W - 600) / 2.0, (H - 580) / 2.0, 600, 580
        left, right = x + 28, x + w - 28
        body = well(cv, x, y, w, h, r=10, fill=C.SURFACE, outline=C.LINE_2)
        cv.tag_bind(body, "<Button-1>", lambda e: "break")

        disc = avatar_disc(int(round(s(64))), C.SURFACE) \
            if os.path.exists(AVATAR_PATH) else None
        tx = left
        if disc is not None:
            rgb = cv2.cvtColor(disc, cv2.COLOR_BGR2RGB)
            hh, ww = rgb.shape[:2]
            ppm = ("P6\n%d %d\n255\n" % (ww, hh)).encode("ascii") + rgb.tobytes()
            self._card_photo = tk.PhotoImage(data=ppm)
            cv.create_image(s(left + 32), s(y + 60), image=self._card_photo,
                            anchor="center")
            tx = left + 64 + 18
        text(cv, tx, y + 42, text=profile.get("name", "?"), fill=C.INK,
             font=self.font(20, "bold"), anchor="w")
        if stats.get("motto"):
            text(cv, tx, y + 67, text=stats["motto"], fill=C.INK_2,
                 font=self.font(10.5), anchor="w")
        if stats.get("vip"):
            text(cv, tx, y + 86, text=stats["vip"], fill=C.INK_3,
                 font=self.font(9.5), anchor="w")
        Btn(cv, right - 30, y + 16, 30, 28, "\u2715", self.close_profile_card,
            self.font(10), fill=C.SURFACE, border=C.SURFACE, fg=C.INK_3)
        hline(cv, x + 1, x + w - 1, y + 118, C.LINE)

        if not stats:
            text(cv, x + w / 2, y + h / 2,
                 text="Fetching your stats\u2026" if self.card_fetching
                 else "No stats kept yet \u2014 press Refresh.",
                 fill=C.INK_3, font=self.font(11), anchor="center")
        else:
            self._draw_card_stats(stats, x, y, w, left, right)

        hline(cv, x + 1, x + w - 1, y + h - 56, C.LINE)
        fetched = profile.get("fetched") or 0
        if self.card_fetching:
            when = "updating\u2026"
        elif fetched:
            mins = int((time.time() - fetched) / 60)
            when = "updated just now" if mins < 1 else \
                "updated %d min ago" % mins if mins < 60 else \
                "updated %d h ago" % (mins // 60)
        else:
            when = ""
        bits = []
        if "posts" in stats:
            bits.append("Forum: %s posts \u00b7 %s topics"
                        % (commas(stats["posts"]), commas(stats.get("topics", 0))))
        if when:
            bits.append(when)
        text(cv, left, y + h - 28, text="   \u00b7   ".join(bits), fill=C.INK_3,
             font=self.font(9.5), anchor="w")
        f = self.font(9.5)
        Btn(cv, right - 84, y + h - 41, 84, 26, "Refresh",
            self.refresh_card, f)
        Btn(cv, right - 84 - 8 - 124, y + h - 41, 124, 26, "Open in browser",
            self.open_profile_page, f)

        for item in cv.find_all():
            if item >= first:
                cv.addtag_withtag("pcard", item)
        cv.delete(first)
        if self.ox or self.oy:
            for item in cv.find_withtag("pcard"):
                if item != veil:
                    cv.move(item, self.ox, self.oy)
        cv.tag_raise("pcard")

    def _draw_card_stats(self, stats, x, y, w, left, right):
        cv = self.cv
        gap = 12
        tw = (right - left - 2 * gap) / 3.0
        ty = y + 136
        tiles = []
        if stats.get("achievements") is not None:
            tiles.append(("Achievement points", commas(stats["achievements"]),
                          None, True))
        ring = stats.get("ring")
        if ring:
            tiles.append(("Ring season", ring["rank"],
                          "%s pts \u00b7 %s of %s fights"
                          % (commas(ring["points"]), commas(ring["fights"]),
                             commas(ring["of"])), False))
        if stats.get("island"):
            tiles.append(("On the island for", stats["island"], None, False))
        for i, (label, value, note, numeric) in enumerate(tiles[:3]):
            tx = left + i * (tw + gap)
            well(cv, tx, ty, tw, 84, r=8, fill=C.SURFACE_2, outline=C.LINE_2)
            self.label(tx + 16, ty + 20, label, C.INK_3, size=9.5)
            font = self.font(18, "bold", numeric=True) if numeric \
                else self.font(14, "bold")
            text(cv, tx + 16, ty + 48,
                 text=self.fit(value, font, tw - 28), fill=C.INK, font=font,
                 anchor="w")
            if note:
                self.readout(tx + 16, ty + 70, note, C.INK_3, size=8.5)

        games = sorted(stats.get("games") or [], key=lambda g: -(g["wins"] or 0))
        if not games:
            return
        gy = y + 246
        text(cv, left, gy, text="Minigames", fill=C.INK,
             font=self.font(12, "bold"), anchor="w")
        cols = {"bar": left + 150, "bar_w": 230, "wins": right - 96,
                "best": right}
        self.label(cols["wins"], gy + 26, "Wins", C.INK_3, size=9, anchor="e")
        self.label(cols["best"], gy + 26, "Best place", C.INK_3, size=9,
                   anchor="e")
        most = max(g["wins"] or 0 for g in games) or 1
        step = min(28.0, 196.0 / max(1, len(games)))
        for i, g in enumerate(games[:8]):
            ry = gy + 52 + i * step
            text(cv, left, ry, text=self.fit(g["name"], self.font(10.5), 140),
                 fill=C.INK, font=self.font(10.5), anchor="w")
            rounded(cv, cols["bar"], ry - 4, cols["bar_w"], 8, 4,
                    fill=C.SUNK, outline="")
            share = (g["wins"] or 0) / float(most)
            bw = max(4.0, cols["bar_w"] * share)
            rounded(cv, cols["bar"], ry - 4, bw, 8, 4, fill=C.DATA,
                    outline="")
            self.readout(cols["wins"], ry, commas(g["wins"] or 0), C.INK,
                         size=10.5, anchor="e")
            best = g.get("best")
            self.readout(cols["best"], ry, "#" + commas(best) if best else
                         "\u2014", C.INK_2, size=10, anchor="e")

    def refresh_card(self):
        if self.card_fetching:
            return
        self.card_fetching = True
        self.draw_profile_card()
        threading.Thread(target=self.profile_loop, kwargs={"force": True},
                         daemon=True).start()

    def open_profile_page(self):
        import webbrowser
        try:
            webbrowser.open(PROFILE_URL % self.profile_slug)
        except Exception as e:
            self.status("Could not open the browser: %s" % e, C.RED)


    def update_ping(self, frame):
        try:
            ping = read_ping(frame)
        except Exception as e:
            log("Ping overlay could not be read: %s" % e, key="ping", every=60)
            ping = None
        now = time.monotonic()
        if ping is not None:
            self.ping, self.ping_seen = ping, now
            self.ping_recent = (self.ping_recent + [ping.level])[-3:]
        elif now - self.ping_seen > 3.0:
            self.ping, self.ping_recent = None, []
        bad = len(self.ping_recent) >= 2 and \
            all(level == PING_BAD for level in self.ping_recent[-2:])
        if self.lag_guard and bad and not self.lag_hold:
            self.lag_hold, self.lag_clear_since = True, None
            self.event("Lag spike \u2014 ping %s, holding clicks." % self.ping.range_text,
                       C.WARN)
        elif self.lag_hold:
            calm = self.ping is None or self.ping.level != PING_BAD
            if not self.lag_guard:
                self.lag_hold = False
            elif calm:
                if self.lag_clear_since is None:
                    self.lag_clear_since = now
                elif now - self.lag_clear_since >= 1.5:
                    self.lag_hold = False
                    self.event("Lag over \u2014 carrying on.", C.INK_2)
            else:
                self.lag_clear_since = None
        self.ui(self.render_ping)

    def lag_extra(self):
        if not self.lag_guard or self.ping is None:
            return 0.0
        return {PING_GOOD: 0.0, PING_SLOW: 0.35, PING_BAD: 0.8}[self.ping.level]

    def holding_for_lag(self):
        if self.lag_hold and self.lag_guard:
            self.status("The game is lagging \u2014 holding the click.",
                        C.WARN)
            return True
        return False

    def update_me(self, frame):
        if not self.find_me_on or not self.profile:
            return
        now = time.monotonic()
        if now - self.me_looked < ME_EVERY:
            return
        self.me_looked = now
        colours = self.profile.get("colours") or []
        if len(telling_colours(colours)) < 2:
            return
        scale = 1.0
        if self.hits:
            scale = float(np.median([h.scale for h in self.hits]))
        size = (int(56 * scale), int(96 * scale))
        near = self.me[:2] if self.me and now - self.me[2] < ME_TRUST \
            else (self.me_walk[:2] if self.me_walk else None)
        try:
            found = find_me(frame, colours, size=size, margin=self.margin,
                            near=near, avoid=[h.box for h in self.hits])
        except Exception as e:
            log("Could not look for the player: %s" % e, key="me", every=60)
            return
        if found is not None:
            self.me = (found[0], found[1], now)
            if not self.me_announced:
                self.me_announced = True
                self.event("Found you on the floor.", C.INK_2)

    def player_point(self, size):
        now = time.monotonic()
        if self.me is not None and now - self.me[2] < ME_TRUST:
            return (float(self.me[0]), float(self.me[1]))
        if self.me_walk is not None and now - self.me_walk[2] < 20.0:
            return (float(self.me_walk[0]), float(self.me_walk[1]))
        return (size[0] / 2.0, size[1] / 2.0)

    def profile_loop(self, force=False):
        if not self.profile_slug:
            return
        before = self.profile
        try:
            profile = fetch_profile(self.profile_slug, force=force)
        finally:
            self.card_fetching = False
            self.ui(self.draw_profile_card)
        if profile:
            self.profile = profile
            self.ui(self.apply_profile)
            if force and profile is before or \
                    (before and profile.get("fetched") == before.get("fetched")
                     and force):
                self.event("Could not reach boombang.tv \u2014 kept the saved "
                           "profile.", C.WARN)
            else:
                self.event("Profile loaded: %s." % profile.get("name", "?"),
                           C.INK_2)
        else:
            self.event("Could not load the profile for '%s'."
                       % self.profile_slug, C.WARN)

    def refresh_profile(self):
        self.status("Fetching the profile\u2026", C.INK_2)
        threading.Thread(target=self.profile_loop, kwargs={"force": True},
                         daemon=True).start()

    def toggle_lag_guard(self):
        self.lag_guard = not self.lag_guard
        if not self.lag_guard:
            self.lag_hold = False
        self.render_settings()
        self.cv.itemconfigure(
            self.detect_note,
            text="Reads the game's ping overlay. Slow: waits a little longer "
                 "between clicks and before answering popups. Lagging: holds "
                 "clicks until it settles. Needs the overlay switched on.")
        self.save_config_soon()

    def toggle_find_me(self):
        self.find_me_on = not self.find_me_on
        if not self.find_me_on:
            self.me = None
        self.render_settings()
        self.cv.itemconfigure(
            self.detect_note,
            text="Finds your avatar on the floor by its colours from your "
                 "profile, so 'nearest' means nearest to you. Off, it goes by "
                 "where it last sent you, then the middle of the view.")
        self.save_config_soon()

    def pick(self, hits, frame):
        live = [h for h in hits if h.template.enabled
                and not self.shelf.holds(h.click)]
        if not live:
            return None
        h, w = frame.shape[:2]
        mid = self.player_point((w, h))

        def distance(hit):
            return ((hit.click[0] - mid[0]) ** 2
                    + (hit.click[1] - mid[1]) ** 2)

        if self.nearest_first:
            live.sort(key=lambda hit: (hit.template.rank, distance(hit)))
        else:
            live.sort(key=lambda hit: (hit.template.rank, -hit.score))
        return live[0]

    def chase_loop(self):
        started = time.perf_counter()
        seen = -1
        while True:
            every = min(self.poll, WATCH_AT_LEAST)
            spent = time.perf_counter() - started
            if every - spent > 0 and self.closing.wait(every - spent):
                return
            if self.live_seq == seen and self.game_active:
                self.new_frame.clear()
                if self.live_seq == seen:
                    self.new_frame.wait(0.25)
            if self.closing.is_set():
                return
            seen = self.live_seq
            started = time.perf_counter()
            self._stage_list, self._stage_at = [], started
            try:
                if not self.game_active:
                    self.pending = None
                    self.popup_since = None
                    self.prize_state = None
                    self.popup_open = False
                    if self.chase is not None:
                        self.chase = None
                        self.hits = []
                        self.ui(self.render_items)
                        self.ui(self.render_target, None)
                    continue
                armed = self.armed.is_set()
                if not armed and (self.chase is not None
                                  or self.pending is not None):
                    self.chase = None
                    self.pending = None
                    self.hits = []
                    self.ui(self.render_items)
                    self.ui(self.render_target, None)

                frame = self.current_frame()
                if frame is None:
                    continue
                self.show_capture()
                now = time.monotonic()
                self.last_look = now
                self.looks_counted += 1
                self.stage("capture")
                if now - self.ping_read >= PING_EVERY:
                    self.ping_read = now
                    self.update_ping(frame)
                self.stage("ping overlay")
                self.update_me(frame)
                self.stage("finding you")

                boxes = choice_row(frame, margin=self.margin)
                self.stage("popup check")
                if boxes:
                    self.popup_missing = None
                    if not self.popup_open:
                        self.popup_open = True
                        self.popup_since = now
                        self.popup_item = None
                        self.popup_seen()
                    if self.answer_prize:
                        self.on_popup(frame, boxes)
                    else:
                        self.hits = []
                        self.set_verdict()
                        self.status("Prize popup up \u2014 left for you "
                                    "(answering is off).", C.INK_2)
                    continue

                if self.popup_open:
                    if self.popup_missing is None:
                        self.popup_missing = now
                    if now - self.popup_missing < POPUP_CLOSE:
                        continue
                    self.popup_gone()
                self.popup_missing = None
                self.popup_since = None

                if not armed:
                    self.idle_shown = now
                    self.hits = floor_memory.find(frame, self.templates,
                                                  self.min_score, self.margin)
                    self.remember_floor(self.hits, frame)
                    self.set_verdict()
                    self.ui(self.render_items)
                    self.ui(self.render_target, None)
                elif self.chase is not None:
                    self.follow(frame)
                elif self.pending is not None:
                    self.wait_for_prize(frame)
                else:
                    self.expire_prize()
                    self.look(frame)
            except Exception as e:
                log("Chase loop error: %s" % e, key="chase")
            finally:
                self.report_slow(started)

    def stage(self, name):
        now = time.perf_counter()
        self._stage_list.append((name, now - self._stage_at))
        self._stage_at = now

    def report_slow(self, started):
        now = time.perf_counter()
        total = now - started
        if total < SLOW_PASS:
            return
        parts = self._stage_list + [("items and clicks", now - self._stage_at)]
        name, _took = max(parts, key=lambda p: p[1])
        detail = ", ".join("%s %.2fs" % p for p in parts if p[1] >= 0.05)
        log("Slow pass: %.2fs (%s)" % (total, detail), key="slowpass",
            every=10)
        self.status("One look took %.1f s \u2014 mostly the %s."
                    % (total, name), C.WARN)

    def idle_look(self):
        pass

    def count_catch(self, hit, clicks):
        self.attempts.forget(hit.click)
        self.popup_item = hit.template
        self.status("Caught the %s after %d click%s \u2014 prize popup up."
                    % (nice(hit.name), clicks, "" if clicks == 1 else "s"),
                    C.AMBER)
        self.ui(self.render_items)
        self.ui(self.render_target, None)

    def record_catch(self, template, claimed):
        name = template.name if template is not None else None
        key = name or "unidentified"
        self.session_counts[key] = self.session_counts.get(key, 0) + 1
        totals = self.stats.setdefault("all_time", {})
        totals[key] = totals.get(key, 0) + 1
        self.save_stats()
        self.taken += 1
        self.catch_times.append(time.monotonic())
        if template is None:
            self.event("Caught an item \u2014 could not tell which.", C.INK_2)
        else:
            self.event("Caught a %s%s." % (nice(name),
                                          " \u00b7 prize claimed" if claimed
                                          else ""), C.AMBER)
        self.ui(self.render_counts)
        self.ui(self.render_items)

    def load_stats(self):
        try:
            with open(STATS_PATH, "r", encoding="utf-8") as f:
                data = json.load(f)
            if isinstance(data, dict):
                data["all_time"] = {RENAMED.get(k, k): int(v) for k, v in
                                    dict(data.get("all_time", {})).items()}
                return data
        except FileNotFoundError:
            pass
        except Exception as e:
            log("Item stats unreadable, starting fresh: %s" % e)
        return {"all_time": {}, "since": time.strftime("%Y-%m-%d")}

    def save_stats(self):
        try:
            tmp = STATS_PATH + ".tmp"
            with open(tmp, "w", encoding="utf-8") as f:
                json.dump(self.stats, f, indent=1)
            os.replace(tmp, STATS_PATH)
        except Exception as e:
            log("Could not save item stats: %s" % e, key="stats", every=60)

    def reset_counts(self):
        self.session_counts = {}
        self.taken = self.claimed = self.dropped = self.stolen = 0
        self.unclaimed = self.seen = 0
        self.catch_times = []
        self.session_start = time.perf_counter()
        self.event("Session counts reset.", C.INK_2)
        self.render_counts()
        self.render_items()

    def popup_seen(self):
        now = time.monotonic()
        loss, self.recent_loss = self.recent_loss, None
        if loss is not None and now - loss[3] < LOSS_UNDO \
                and self.pending is None \
                and (self.chase is None or self.chase.age < 3.0):
            kind, hit, _clicks, _when = loss
            if kind == "dropped":
                self.dropped = max(0, self.dropped - 1)
                self.shelf.drop(hit.click)
            else:
                self.stolen = max(0, self.stolen - 1)
            self.attempts.forget(hit.click)
            self.popup_item = hit.template
            if self.answer_prize and self.awaiting is None:
                self.awaiting = (hit.template, now, 0)
            self.chase = None
            self.event("The %s was caught after all \u2014 corrected."
                       % nice(hit.name), C.INK_2)
            self.ui(self.render_counts)
            self.ui(self.render_items)
            return
        if self.chase is not None:
            chase, self.chase = self.chase, None
            if self.awaiting is None and self.answer_prize:
                self.awaiting = (chase.hit.template, time.monotonic(), 0)
            self.count_catch(chase.hit, chase.clicks)
        elif self.pending is not None:
            hit, clicks, _since = self.pending
            self.pending = None
            self.count_catch(hit, clicks)

    def wait_for_prize(self, frame):
        hit, clicks, since = self.pending
        self.hits = floor_memory.find(frame, self.templates, self.min_score,
                                      self.margin)
        self.remember_floor(self.hits, frame)
        self.ui(self.render_items)
        again = track(frame, hit, self.min_score * CONFIRM_EASE)
        if again is not None:
            self.pending = None
            self.awaiting = None
            started = self.attempts.begin(again.click)
            chase = Chase(again, started)
            chase.clicks = clicks
            self.chase = chase
            self.hits = [again]
            self.status("The %s is still there \u2014 carrying on."
                        % nice(hit.name), C.AMBER)
            self.ui(self.render_target, chase)
            self.preview(frame, chase)
            return
        if time.monotonic() - since >= POPUP_GRACE + 3 * self.lag_extra():
            self.pending = None
            self.awaiting = None
            self.walked_to = []
            self.stolen += 1
            self.attempts.forget(hit.click)
            self.recent_loss = ("stolen", hit, clicks, time.monotonic())
            self.event("The %s went with no prize popup \u2014 somebody "
                       "else got it." % nice(hit.name), C.INK_2)
            self.ui(self.render_counts)
            self.ui(self.render_items)
            self.ui(self.render_target, None)
        self.preview(frame)

    def expire_prize(self):
        if self.awaiting is not None and \
                time.monotonic() - self.awaiting[1] > PRIZE_WINDOW:
            self.awaiting = None
            self.ui(self.render_items)

    def remember_floor(self, hits, frame):
        now = time.monotonic()
        for old in self.floor_last:
            if not any(h.name == old.name and h.near(old.click)
                       for h in hits):
                self.vanished.append((old.template, old.click, now))
        self.vanished = [v for v in self.vanished
                         if now - v[2] < CLAIM_WINDOW][-20:]
        self.floor_last = list(hits)
        self.floor_last_at = now
        if frame is not None:
            fh, fw = frame.shape[:2]
            self.floor_mid = self.player_point((fw, fh))
            if self.spawns.observe(hits, (fw, fh)):
                self.spawns.save()

    def prize_candidates(self):
        now = time.monotonic()
        mid = self.floor_mid or (0.0, 0.0)

        def far(point):
            return (point[0] - mid[0]) ** 2 + (point[1] - mid[1]) ** 2

        out, seen = [], set()

        def add(template, why):
            if template is not None and template.name not in seen:
                seen.add(template.name)
                out.append((template, why))

        if self.awaiting is not None:
            add(self.awaiting[0], "caught")
        for t, _w in sorted((p for p in self.walked_to
                             if now - p[1] < CLAIM_WINDOW),
                            key=lambda p: -p[1]):
            add(t, "walked to")
        for t, _c, _w in sorted(self.vanished,
                                key=lambda v: (-round(v[2], 1), far(v[1]))):
            add(t, "just went from the floor")
        if now - self.floor_last_at < CLAIM_WINDOW:
            for h in sorted(self.floor_last, key=lambda h: far(h.click)):
                add(h.template, "was lying nearby")
        for t in sorted(self.templates,
                        key=lambda t: (not t.enabled, t.rank)):
            add(t, "guess")
        return out

    def popup_gone(self):
        self.popup_open = False
        self.popup_since = None
        st, self.prize_state = self.prize_state, None
        claimed = bool(st and st["clicks"] and self.auto_click
                       and not st["gave_up"])
        if claimed:
            template = st["template"]
        elif self.popup_item is not None:
            template = self.popup_item
        elif self.awaiting is not None:
            template = self.awaiting[0]
        elif st and st["template"] is not None:
            template = st["template"]
        else:
            template = None
        self.popup_item = None
        self.record_catch(template, claimed)
        self.awaiting = None
        self.walked_to = []
        self.vanished = []
        self.floor_last = []
        self.spawns.caught()
        if self.return_home and self.armed.is_set():
            now = time.monotonic()
            self.home_plan = [now + d for d in HOME_CLICKS]
        self.ui(self.render_items)

    def on_popup(self, frame, boxes):
        self.hits = []
        self.preview(frame)
        self.ui(self.render_target, None)
        now = time.monotonic()

        st = self.prize_state
        if st is None:
            st = self.prize_state = {"tries": 0, "clicks": 0,
                                     "clicked_at": None, "template": None,
                                     "point": None, "gave_up": False}

        settling = self.prize_wait + self.lag_extra() \
            - (now - (self.popup_since or now))
        if settling > 0 and not st["clicks"]:
            self.status("Letting the prize popup settle (%.1fs)..." % settling,
                        C.INK_2)
            return

        if st["clicks"]:
            if now - st["clicked_at"] < PRIZE_RECHECK + self.lag_extra():
                self.status("Clicked the %s \u2014 checking the popup closed..."
                            % nice(st["template"].name), C.INK_2)
                return
            if st["clicks"] > PRIZE_RECLICKS:
                if not st["gave_up"]:
                    st["gave_up"] = True
                    self.unclaimed += 1
                    self.alert("prize", "A prize popup needs you.")
                    self.ui(self.render_counts)
                self.status("Clicked the %s %d times and the popup is still "
                            "up \u2014 left for you." % (nice(st["template"].name),
                                                        st["clicks"]), C.RED)
                return
            pick, scored = identify(frame, st["template"].prize
                                    or st["template"], boxes,
                                    self.prize_score * PRIZE_RELAX,
                                    lead=PRIZE_RELAX_LEAD)
            point = st["point"]
            if pick is not None:
                x, y, w, h = pick[:4]
                point = (int(x + w / 2), int(y + h / 2))
            self.click_prize(st, st["template"], point, again=True)
            return

        st["tries"] += 1
        strict = st["tries"] <= PRIZE_TRIES
        choice = self.choose_prize(frame, boxes, strict)
        if choice is None:
            if strict:
                self.status("Working out which picture it is (look %d)..."
                            % st["tries"], C.INK_2)
            else:
                if not st["gave_up"]:
                    st["gave_up"] = True
                    self.unclaimed += 1
                    self.alert("prize", "A prize popup needs you.")
                    self.ui(self.render_counts)
                best = self.last_prize[1][0][0] \
                    if self.last_prize and self.last_prize[1] else 0.0
                self.status("Cannot match any picture to a known item "
                            "(best %.2f) \u2014 still trying." % best, C.RED)
            return

        template, why, pick, scored = choice
        x, y, w, h = pick[:4]
        point = (int(x + w / 2), int(y + h / 2))
        if st["gave_up"]:
            st["gave_up"] = False
            self.unclaimed = max(0, self.unclaimed - 1)
        self.click_prize(st, template, point, why=why, scored=scored)

    def choose_prize(self, frame, boxes, strict):
        candidates = self.prize_candidates()
        known = [c for c in candidates if c[1] in ("caught", "walked to")]
        st = self.prize_state
        if known and not (st and st["tries"] > 2 * PRIZE_TRIES):
            candidates = known
        passes = [(self.prize_score, 0.06)]
        if not strict:
            passes.append((self.prize_score * PRIZE_RELAX, PRIZE_RELAX_LEAD))
        first = None
        sure_what = len(candidates) == 1 and \
            candidates[0][1] in ("caught", "walked to")
        for threshold, lead in passes:
            for template, why in candidates:
                pick, scored = identify(frame, template.prize or template,
                                        boxes, threshold, lead=lead)
                if first is None:
                    first = (boxes, scored, None)
                if pick is None and template.prize is None and \
                        (sure_what or threshold < self.prize_score):
                    pick, scored = identify_by_colour(frame, template, boxes)
                if pick is not None:
                    self.last_prize = (boxes, scored, pick)
                    return template, why, pick, scored
        if first is not None:
            self.last_prize = first
        return None

    def click_prize(self, st, template, point, why=None, scored=None,
                    again=False):
        if self.holding_for_lag():
            return
        st["template"], st["point"] = template, point
        st["clicked_at"] = time.monotonic()
        st["clicks"] += 1
        if not self.auto_click:
            st["clicks"] = PRIZE_RECLICKS + 1
            st["gave_up"] = True
            self.status("Would claim the %s \u2014 clicking is off."
                        % nice(template.name), C.INK_2)
            return
        debug("click %d,%d  claiming the %s from the prize popup%s"
              % (point[0], point[1], template.name,
                 " (again)" if again else ""))
        self.click_client(point[0], point[1],
                          hover=self.hover_ms / 1000.0,
                          dwell=self.dwell_ms / 1000.0,
                          button=self.button, use_cache=False)
        if again:
            self.status("Popup still up \u2014 clicked the %s again (%d)."
                        % (nice(template.name), st["clicks"]), C.AMBER)
            return
        self.claimed += 1
        top = scored[0][0] if scored else 0.0
        nxt = scored[1][0] if scored and len(scored) > 1 else 0.0
        note = "" if why == "caught" else " \u2014 %s" % why
        self.status("Claimed the %s (%.2f, next best %.2f)%s."
                    % (nice(template.name), top, nxt, note), C.AMBER)
        self.ui(self.render_counts)
        self.ui(self.render_items)

    def home_item(self):
        live = [t for t in self.templates if t.enabled]
        return min(live, key=lambda t: t.rank) if live else None

    def home_spot(self, size):
        template = self.home_item()
        if template is None or not size:
            return None
        found = self.spawns.home(template.name)
        if found is None:
            return None
        fx, fy, n = found
        w, h = size
        m = self.margin
        x = int(clamp(fx * w, m, w - 1 - m))
        y = int(clamp(fy * h, m, h - 1 - m))
        return x, y, n, template

    def maybe_go_home(self, frame):
        if not self.home_plan or not self.return_home:
            return
        now = time.monotonic()
        if now < self.home_plan[0]:
            return
        if self.holding_for_lag():
            return
        self.home_plan.pop(0)
        fh, fw = frame.shape[:2]
        spot = self.home_spot((fw, fh))
        if spot is None or spot[2] < SPAWN_MIN:
            self.home_plan = []
            template = self.home_item()
            if template is not None:
                have = self.spawns.count(template.name)
                self.status("No home spot for the %s yet \u2014 %d sighting%s "
                            "so far." % (nice(template.name), have,
                                         "" if have == 1 else "s"), C.INK_2)
            return
        x, y, n, template = spot
        if not self.auto_click:
            self.status("Would head back to the %s spot \u2014 clicking is off."
                        % nice(template.name), C.INK_2)
            self.home_plan = []
            return
        fresh = self.next_frame()
        if fresh is not None and choice_row(fresh, margin=self.margin):
            self.home_plan = []
            return
        debug("click %d,%d  heading back to the %s spot" % (x, y, template.name))
        self.me_walk = (x, y, time.monotonic())
        self.click_client(x, y, hover=self.hover_ms / 1000.0,
                          dwell=self.dwell_ms / 1000.0, button=self.button,
                          use_cache=False)
        self.status("Heading back to the %s spot (%d drops seen there)."
                    % (nice(template.name), n), C.AMBER)

    def look(self, frame):
        self.hits = floor_memory.find(frame, self.templates, self.min_score,
                                      self.margin)
        self.remember_floor(self.hits, frame)
        self.ui(self.render_items)
        self.preview(frame)
        chosen = self.pick(self.hits, frame)
        if chosen is None:
            self.ui(self.render_target, None)
            self.maybe_go_home(frame)
            return
        self.home_plan = []
        started = self.attempts.begin(chosen.click)
        self.seen += 1
        self.chase = Chase(chosen, started)
        self.last_line = "%s at %d, %d" % (chosen.name, chosen.click[0],
                                           chosen.click[1])
        self.status("Spotted a %s." % nice(chosen.name), C.AMBER)
        self.ui(self.render_counts)
        self.act()
        self.ui(self.render_target, self.chase)

    def give_up(self, chase):
        self.dropped += 1
        self.shelf.add(chase.hit.click)
        self.attempts.forget(chase.hit.click)
        self.chase = None
        self.recent_loss = ("dropped", chase.hit, chase.clicks,
                            time.monotonic())
        self.event("Could not reach the %s \u2014 leaving it for a minute."
                   % nice(chase.name), C.WARN)
        self.ui(self.render_counts)
        self.ui(self.render_items)
        self.ui(self.render_target, None)

    def follow(self, frame):
        chase = self.chase
        floor = floor_memory.find(frame, self.templates, self.min_score,
                                  self.margin)
        self.remember_floor(floor, frame)
        better = self.pick([h for h in floor
                            if not h.overlaps(chase.hit, 0.25)], frame)
        if better is not None and \
                better.template.rank < chase.hit.template.rank:
            self.event("A %s dropped \u2014 going for it instead of the %s."
                       % (nice(better.name), nice(chase.name)), C.INK)
            chase = Chase(better, self.attempts.begin(better.click))
            self.chase = chase
            self.seen += 1
            self.hits = floor
            self.ui(self.render_counts)
            self.ui(self.render_items)
            self.act()
            self.preview(frame, chase)
            self.ui(self.render_target, chase)
            return
        self.hits = floor
        self.ui(self.render_items)
        again = track(frame, chase.hit, self.min_score * 0.86)
        if again is None:
            chase.lost_for += 1
            if chase.lost_for >= LOST_LOOKS:
                seen = find_items(frame, [chase.hit.template],
                                  threshold=self.min_score * CONFIRM_EASE,
                                  margin=self.margin, everything=True)
                still = [h for h in seen if h.overlaps(chase.hit, 0.25)]
                self.hits = [h for h in self.hits
                             if not any(h.overlaps(k) for k in still)] + still
                if still:
                    chase.hit = still[0]
                    chase.lost_for = 0
                    if chase.age > self.patience:
                        self.give_up(chase)
                        return
                else:
                    if self.answer_prize:
                        self.awaiting = (chase.hit.template, time.monotonic(), 0)
                    self.pending = (chase.hit, chase.clicks, time.monotonic())
                    self.chase = None
                    self.status("The %s has gone \u2014 waiting for the prize "
                                "question." % nice(chase.name), C.INK_2)
                    self.ui(self.render_items)
                    self.ui(self.render_target, None)
                    self.preview(frame)
                    return
            self.preview(frame, chase)
            self.ui(self.render_target, chase)
            return

        chase.lost_for = 0
        if chase.age > self.patience:
            self.give_up(chase)
            return
        self.hits = [h for h in self.hits if not h.overlaps(again, 0.25)] \
            + [again]
        chase.hit = again

        if chase.due(self.click_every + self.lag_extra()):
            self.act()
        self.preview(frame, chase)
        self.ui(self.render_target, chase)

    def act(self):
        chase = self.chase
        if chase is None:
            return
        if not self.auto_click:
            chase.clicked()
            self.status("Would click the %s \u2014 clicking is off." % nice(chase.name),
                        C.INK_2)
            return
        if self.holding_for_lag():
            return
        fresh = self.next_frame()
        if fresh is not None and choice_row(fresh, margin=self.margin):
            self.status("Popup opened just now \u2014 holding the click.",
                        C.INK_2)
            return
        x, y = chase.hit.click
        if self.client_now:
            cw, ch = self.client_now
            x, y = int(clamp(x, 0, cw - 1)), int(clamp(y, 0, ch - 1))
        if self.spread:
            x += random.randint(-self.spread, self.spread)
            y += random.randint(-self.spread, self.spread)
        debug("click %d,%d  walking to the %s at %s"
              % (x, y, chase.name, chase.hit.box))
        sent = self.click_client(x, y, hover=self.hover_ms / 1000.0,
                                 dwell=self.dwell_ms / 1000.0,
                                 button=self.button, use_cache=False)
        chase.clicked()
        now = time.monotonic()
        self.walked_to = [(t, when) for t, when in self.walked_to
                          if now - when < CLAIM_WINDOW
                          and t is not chase.hit.template]
        self.walked_to.append((chase.hit.template, now))
        self.me_walk = (x, y, now)
        if not sent:
            self.status("Click could not be posted.", C.RED)
            return
        self.clicks += 1

    def test_once(self):
        if not self.game_active:
            self.status("No game window to look at.", C.RED)
            return
        threading.Thread(target=self._test_thread, daemon=True).start()

    def _test_thread(self):
        frame = self.current_frame()
        if frame is None:
            self.status("Nothing could be captured.", C.RED)
            return
        self.show_capture()
        started = time.perf_counter()
        hits = find_items(frame, self.templates, threshold=self.min_score,
                          margin=self.margin)
        took = time.perf_counter() - started
        if self.chase is None:
            self.hits = hits
            self.ui(self.render_items)
            self.ui(self.render_target, None)
        self.verdict_state = (list(hits), None)
        if hits:
            self.status("Found %d: %s  (%.0f ms)"
                        % (len(hits),
                           ", ".join("%s %.2f" % (nice(h.name), h.score)
                                     for h in hits[:4]), took * 1000),
                        C.AMBER)
        else:
            self.status("Nothing on the floor right now (%.0f ms). Press F3 to "
                        "save what it saw." % (took * 1000), C.INK_2)

    def dump_debug(self):
        frame = self.last_frame
        if frame is None:
            self.status("Nothing captured yet \u2014 press F2 first.", C.WARN)
            return
        try:
            cv2.imwrite(DEBUG_IMAGE, self.annotate(frame, self.last_verdict))
            cv2.imwrite(DEBUG_IMAGE.replace(".png", "_raw.png"), frame)
        except Exception as e:
            self.status("Could not write the image: %s" % e, C.RED)
            return
        print("\n=== AUTO EVENT ===")
        print("capture method: %s (last frame: %s)"
              % (self.capture_method, self.capture_used))
        if self.client_now:
            print("client area: %d x %d" % self.client_now)
        dpi = window_dpi(self.game_hwnd)
        if dpi:
            print("window DPI: %d (%d%% scaling)" % (dpi, round(dpi / 0.96)))
        print("needs at least %.2f, ignoring %d px of edge"
              % (self.min_score, self.margin))
        for template in self.templates:
            print("  %-13s %s  hue %d..%d, sat >= %d, val >= %d"
                  % (template.name, "on " if template.enabled else "off",
                     template.hue[0], template.hue[1], template.sat_min,
                     template.val_min))
        hits = (self.last_verdict or ([], None))[0]
        if not hits:
            print("nothing was found in this frame")
        for hit in hits:
            print("  %-13s %.3f at %s, drawn at %.2fx, click %s"
                  % (hit.name, hit.score, hit.box, hit.scale, hit.click))
        if self.last_prize:
            boxes, scored, pick = self.last_prize
            print("last prize popup: %d pictures, scores %s, picked %s"
                  % (len(boxes), ", ".join("%.2f" % sc for sc, _b in scored),
                     pick))
        print("image written to %s" % DEBUG_IMAGE)
        self.status("Image written to %s" % os.path.basename(DEBUG_IMAGE),
                    C.AMBER)


    def _target_alive(self, hwnd, target):
        if target == hwnd:
            return True
        try:
            return bool(win32gui.IsChild(hwnd, target))
        except Exception:
            return False

    def resolve_target(self, hwnd, cx, cy, depth=4, use_cache=True):
        key = (hwnd, cx, cy)
        now = time.monotonic()
        if use_cache:
            hit = self._target_cache.get(key)
            if hit is not None:
                target, tx, ty, stamp = hit
                if now - stamp < TARGET_TTL and self._target_alive(hwnd, target):
                    return target, tx, ty

        sx, sy = win32gui.ClientToScreen(hwnd, (cx, cy))
        target = hwnd
        for _ in range(depth):
            tx, ty = win32gui.ScreenToClient(target, (sx, sy))
            child = _user32.RealChildWindowFromPoint(target, _POINT(tx, ty))
            if not child or int(child) == target:
                break
            child = int(child)
            try:
                if win32gui.GetClassName(child) in DEAD_CLASSES:
                    break
            except Exception:
                break
            target = child
        tx, ty = win32gui.ScreenToClient(target, (sx, sy))
        cache = self._target_cache
        if len(cache) > 4096:
            cache = self._target_cache = {}
        cache[key] = (target, tx, ty, now)
        return target, tx, ty

    def click_client(self, cx, cy, hover=0.0, dwell=0.0, button="left",
                     use_cache=True):
        hwnd = self.game_hwnd
        if not hwnd:
            return False
        down, up, flag = BUTTONS.get(button, BUTTONS["left"])
        for attempt in (0, 1):
            posted = False
            try:
                target, tx, ty = self.resolve_target(
                    hwnd, int(cx), int(cy), use_cache=use_cache and attempt == 0)
                lp = lparam(tx, ty)
                win32api.PostMessage(target, win32con.WM_MOUSEMOVE, 0, lp)
                posted = True
                if hover:
                    precise_sleep(hover)
                win32api.PostMessage(target, down, flag, lp)
                if dwell:
                    precise_sleep(dwell)
                win32api.PostMessage(target, up, 0, lp)
                return True
            except Exception as e:
                self._target_cache.pop((hwnd, int(cx), int(cy)), None)
                if attempt == 0 and not posted:
                    continue
                log("Click failed: %s" % e, key="click")
                return False
        return False

    def step_min_score(self, direction):
        self.min_score = ladder(SCORE_STEPS, self.min_score, direction)
        self.render_settings()
        self.cv.itemconfigure(
            self.detect_note,
            text="How closely something has to match the picture of the item. "
                 "Lower it if items are being walked past, raise it if it "
                 "starts chasing the furniture.")
        self.save_config_soon()

    def step_poll(self, direction):
        self.poll = 0.0
        self.capture_fps = int(ladder(CAPTURE_STEPS, self.capture_fps,
                                      direction))
        self.render_settings()
        self.cv.itemconfigure(
            self.detect_note,
            text="How many times a second the game is captured and looked at. "
                 "Every frame is checked the moment it arrives. Higher sees a "
                 "drop a little sooner but works the graphics driver harder; "
                 "20 is plenty. Go lower if the PC struggles.")
        self.save_config_soon()

    def step_click_every(self, direction):
        self.click_every = ladder(CLICK_EVERY_STEPS, self.click_every, direction)
        self.render_settings()
        self.cv.itemconfigure(
            self.detect_note,
            text="How often to click again while walking. Too often and the "
                 "walk keeps restarting; too rarely and the player stops "
                 "short when something gets in the way.")
        self.save_config_soon()

    def step_patience(self, direction):
        self.patience = ladder(PATIENCE_STEPS, self.patience, direction)
        self.render_settings()
        self.cv.itemconfigure(
            self.detect_note,
            text="How long to keep trying for one item before leaving it and "
                 "moving on. Its spot is then left alone for a minute.")
        self.save_config_soon()

    def step_margin(self, direction):
        self.margin = int(ladder(MARGIN_STEPS, self.margin, direction))
        self.render_settings()
        self.cv.itemconfigure(
            self.detect_note,
            text="How much of the edge of the window to ignore, to keep it "
                 "off the panels and the chat rather than the floor.")
        self.save_config_soon()

    def step_prize_score(self, direction):
        self.prize_score = ladder(PRIZE_STEPS, self.prize_score, direction)
        self.render_settings()
        self.cv.itemconfigure(
            self.detect_note,
            text="How sure it has to be before answering which prize it "
                 "caught. A wrong pick loses the prize, so when it cannot "
                 "tell it clicks nothing and says so.")
        self.save_config_soon()

    def step_prize_wait(self, direction):
        self.prize_wait = ladder(PRIZE_WAIT_STEPS, self.prize_wait, direction)
        self.render_settings()
        self.cv.itemconfigure(
            self.detect_note,
            text="How long to let the prize popup finish arriving before "
                 "answering it. Clicking while it is still sliding into "
                 "place is what makes it go wrong.")
        self.save_config_soon()

    def toggle_prize(self):
        self.answer_prize = not self.answer_prize
        if not self.answer_prize:
            self.awaiting = None
            self.prize_state = None
        self.render_settings()
        self.render_items()
        self.save_config_soon()

    def step_spread(self, direction):
        self.spread = int(ladder(SPREAD_STEPS, self.spread, direction))
        self.render_settings()
        self.save_config_soon()

    def step_hover(self, direction):
        self.hover_ms = int(ladder(HOVER_STEPS, self.hover_ms, direction))
        self.save_config_soon()

    def step_dwell(self, direction):
        self.dwell_ms = int(ladder(DWELL_STEPS, self.dwell_ms, direction))
        self.save_config_soon()

    def toggle_auto(self):
        self.auto_click = not self.auto_click
        self.render_settings()
        self.save_config_soon()

    def toggle_return(self):
        self.return_home = not self.return_home
        if not self.return_home:
            self.home_plan = []
        self.render_settings()
        self.cv.itemconfigure(
            self.detect_note,
            text="After each catch, with nothing else on the floor, walk back "
                 "to where the most wanted item turns up most, shown as HOME "
                 "on the preview. It needs %d sightings in one place first."
                 % SPAWN_MIN)
        self.save_config_soon()

    def forget_spots(self):
        self.spawns.clear()
        self.home_plan = []
        self.render_items()
        self.status("Spawn spots forgotten \u2014 learning afresh.", C.INK_2)
        self.cv.itemconfigure(
            self.detect_note,
            text="Do this after changing rooms: the old spots belong to the "
                 "old room.")

    def toggle_order(self):
        self.nearest_first = not self.nearest_first
        self.render_settings()
        self.cv.itemconfigure(
            self.detect_note,
            text="Nearest first goes for whatever is closest to the middle of "
                 "the view, where the player usually is. Best match goes for "
                 "whatever it is surest about, wherever it is.")
        self.save_config_soon()

    def toggle_method(self):
        self.capture_method = ("screen" if self.capture_method == "print"
                               else "print")
        self.render_settings()
        self.cv.itemconfigure(
            self.detect_note,
            text="PrintWindow reads the game even when it is behind something "
                 "else. Screen grab needs the window actually visible, but "
                 "works with renderers that refuse PrintWindow.")
        self.save_config_soon()

    def cycle_button(self):
        order = ("left", "middle", "right")
        self.button = order[(order.index(self.button) + 1) % len(order)]
        self.render_settings()
        self.save_config_soon()

    def toggle_item(self, template):
        template.enabled = not template.enabled
        if not template.enabled:
            if self.chase is not None and self.chase.name == template.name:
                self.chase = None
                self.ui(self.render_target, None)
            if self.pending is not None and self.pending[0].name == template.name:
                self.pending = None
        self.hits = [h for h in self.hits if h.template.enabled]
        self.render_items()
        self.save_config_soon()

    def bind_hotkeys(self):
        bindings = {
            "f1": self.toggle_armed,
            "f2": self.test_once,
            "f3": self.dump_debug,
            "f5": self.disarm,
        }
        failed = []
        for key, fn in bindings.items():
            try:
                keyboard.add_hotkey(key, lambda fn=fn: self.ui(fn))
            except Exception as e:
                failed.append(key)
                log("Could not bind %s: %s. Try running as administrator."
                    % (key, e), key="bind-" + key)
        self.hotkeys_blocked = bool(failed)
        if failed:
            self.cv.itemconfigure(self.legend_note,
                                  text="Hotkeys blocked \u2014 run as administrator")

    def tick_loop(self):
        while not self.closing.wait(1.0):
            try:
                self.ui(self.render_counts)
            except Exception as e:
                log("Tick error: %s" % e)


    def shutdown(self):
        if self.shutting_down:
            return
        self.shutting_down = True
        self.armed.clear()
        self.closing.set()
        self._cancel_save_job()
        if self._pump_job is not None:
            try:
                self.root.after_cancel(self._pump_job)
            except Exception:
                pass
            self._pump_job = None
        self.save_config()
        self.spawns.save()
        try:
            with self._cap_lock:
                self.free_capture()
        except Exception:
            pass
        try:
            keyboard.unhook_all()
        except Exception:
            pass
        if self.hires_timer:
            release_timer_resolution()
        try:
            self.root.destroy()
        except Exception:
            pass


if __name__ == "__main__":
    class _NullStream:
        def write(self, _text):
            return 0

        def flush(self):
            pass

    if sys.stdout is None:
        sys.stdout = _NullStream()
    if sys.stderr is None:
        sys.stderr = _NullStream()
    for _stream in (sys.stdout, sys.stderr):
        try:
            _stream.reconfigure(errors="replace")
        except Exception:
            pass

    DEBUG = "--debug" in sys.argv
    root = tk.Tk()
    app = ItemGrabber(root)
    root.mainloop()
