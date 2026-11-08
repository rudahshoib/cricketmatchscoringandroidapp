"""Cricket Match Scoring App (Tkinter + SQLite)
Android Smartphone Edition - Crisp High-DPI Native Render (420x750)
Warm Minimalist Sage Theme

Features:
- Windows High-DPI Acceleration (100% Crisp Sharp Text & Graphics, No Blur)
- Spacious Mobile Dimensions (420x750, minsize 380x680)
- Android Status Bar (Live Clock, Punch-hole, 5G, Battery)
- Android App Header & Bottom Gesture Navigation
- Mobile Tabbed Team & Player Management
- Bowler Over Quotas & Exhaustion Tracking (Standard Cricket Rules)
- Player Roles (Batsman, Bowler, All-Rounder, Wicketkeeper)
- Match History, Shareable Scorecard, DLS Calculator, Undo Last Ball, Cloud Sync
"""

import ctypes
import json
import math
import os
import sqlite3
import tkinter as tk
from contextlib import contextmanager
from datetime import datetime
from tkinter import filedialog, messagebox, ttk

# --- PREVENTS WINDOWS BLURRINESS (NATIVE HIGH-DPI SHARPNESS) ---
try:
  ctypes.windll.shcore.SetProcessDpiAwareness(1)
except Exception:
  try:
    ctypes.windll.user32.SetProcessDPIAware()
  except Exception:
    pass

# --- ABSOLUTE PATH: ALWAYS STORES DB INSIDE THIS SCRIPT'S FOLDER ---
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
DB_NAME = os.path.join(BASE_DIR, "cricket_app.db")

FONT = "Segoe UI"
MAX_PLAYERS = 11
MAX_OVERS = 50

PLAYER_ROLES = ["All-Rounder", "Batsman", "Bowler", "Wicketkeeper"]

# Match Format Presets
FORMAT_PRESETS = {
    "T20 (20 Overs)": 20,
    "ODI (50 Overs)": 50,
    "T10 (10 Overs)": 10,
    "The 100 (16.4 Overs)": 16,
    "Super Over (1 Over)": 1,
    "Custom Overs": 5,
}


# --- DATABASE HELPERS ---
@contextmanager
def connect():
  """Yields a cursor; commits on success, rolls back on error, always closes."""
  conn = sqlite3.connect(DB_NAME)
  try:
    yield conn.cursor()
    conn.commit()
  except Exception:
    conn.rollback()
    raise
  finally:
    conn.close()


def init_db():
  """Creates tables safely without ever dropping match data."""
  with connect() as cur:
    cur.execute("""
        CREATE TABLE IF NOT EXISTS teams (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT UNIQUE NOT NULL
        )""")
    cur.execute("""
        CREATE TABLE IF NOT EXISTS players (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            team_id INTEGER,
            name TEXT NOT NULL,
            role TEXT DEFAULT 'All-Rounder',
            FOREIGN KEY (team_id) REFERENCES teams (id)
        )""")
    cur.execute("""
        CREATE TABLE IF NOT EXISTS matches (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            team1_id INTEGER,
            team2_id INTEGER,
            team1_name TEXT,
            team2_name TEXT,
            venue TEXT,
            total_overs INTEGER,
            status TEXT, -- 'Ongoing', 'Completed'
            target INTEGER,
            result TEXT,
            format TEXT DEFAULT 'Custom Overs'
        )""")
    cur.execute("""
        CREATE TABLE IF NOT EXISTS match_state (
            match_id INTEGER PRIMARY KEY,
            innings INTEGER, -- 1 or 2
            batting_team_id INTEGER,
            bowling_team_id INTEGER,
            runs INTEGER,
            wickets INTEGER,
            legal_balls INTEGER,
            striker_id INTEGER,
            non_striker_id INTEGER,
            bowler_id INTEGER,
            FOREIGN KEY (match_id) REFERENCES matches (id)
        )""")
    cur.execute("""
        CREATE TABLE IF NOT EXISTS player_match_stats (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            match_id INTEGER,
            player_id INTEGER,
            runs INTEGER DEFAULT 0,
            balls_faced INTEGER DEFAULT 0,
            ones INTEGER DEFAULT 0,
            twos INTEGER DEFAULT 0,
            threes INTEGER DEFAULT 0,
            fours INTEGER DEFAULT 0,
            sixes INTEGER DEFAULT 0,
            out_status TEXT DEFAULT 'Not Out',
            overs_bowled REAL DEFAULT 0,
            legal_balls_bowled INTEGER DEFAULT 0,
            runs_conceded INTEGER DEFAULT 0,
            wickets_taken INTEGER DEFAULT 0,
            batted INTEGER DEFAULT 0
        )""")
    cur.execute("""
        CREATE TABLE IF NOT EXISTS delivery_history (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            match_id INTEGER,
            innings INTEGER,
            striker_id INTEGER,
            non_striker_id INTEGER,
            bowler_id INTEGER,
            bat_runs INTEGER,
            extra_type TEXT,
            extra_runs INTEGER,
            is_wicket INTEGER,
            wicket_type TEXT,
            out_player_id INTEGER,
            prev_runs INTEGER,
            prev_wickets INTEGER,
            prev_legal INTEGER,
            prev_striker_id INTEGER,
            prev_non_striker_id INTEGER,
            FOREIGN KEY (match_id) REFERENCES matches (id)
        )""")

    # Migrations
    cur.execute("PRAGMA table_info(players);")
    p_cols = [c[1] for c in cur.fetchall()]
    if "role" not in p_cols:
      cur.execute("ALTER TABLE players ADD COLUMN role TEXT DEFAULT 'All-Rounder'")

    cur.execute("PRAGMA table_info(player_match_stats);")
    stat_cols = [c[1] for c in cur.fetchall()]
    if "batted" not in stat_cols:
      cur.execute("ALTER TABLE player_match_stats ADD COLUMN batted INTEGER DEFAULT 0")

    cur.execute("PRAGMA table_info(matches);")
    match_cols = [c[1] for c in cur.fetchall()]
    if "format" not in match_cols:
      cur.execute("ALTER TABLE matches ADD COLUMN format TEXT DEFAULT 'Custom Overs'")
    if "team1_name" not in match_cols:
      cur.execute("ALTER TABLE matches ADD COLUMN team1_name TEXT")
      cur.execute("ALTER TABLE matches ADD COLUMN team2_name TEXT")
      cur.execute("UPDATE matches SET team1_name = (SELECT name FROM teams WHERE teams.id = matches.team1_id)")
      cur.execute("UPDATE matches SET team2_name = (SELECT name FROM teams WHERE teams.id = matches.team2_id)")


def seed_test_data():
  """Adds sample teams with explicit roles only if empty."""
  with connect() as cur:
    cur.execute("SELECT COUNT(*) FROM teams")
    if cur.fetchone()[0] > 0:
      return
    samples = {
        "Eagles": [
            ("Aiman", "Batsman"),
            ("Babar", "Batsman"),
            ("Shaheen", "Bowler"),
            ("Rizwan", "Wicketkeeper"),
        ],
        "Falcons": [
            ("Shadab", "All-Rounder"),
            ("Haris", "Bowler"),
            ("Naseem", "Bowler"),
            ("Fakhar", "Batsman"),
        ],
    }
    for team, players in samples.items():
      cur.execute("INSERT INTO teams (name) VALUES (?)", (team,))
      team_id = cur.lastrowid
      cur.executemany(
          "INSERT INTO players (team_id, name, role) VALUES (?, ?, ?)",
          [(team_id, name, role) for name, role in players],
      )


def ensure_stats(cur, match_id, player_id, batted=False):
  cur.execute(
      "SELECT id FROM player_match_stats WHERE match_id = ? AND player_id = ?",
      (match_id, player_id),
  )
  if cur.fetchone() is None:
    cur.execute(
        "INSERT INTO player_match_stats (match_id, player_id, batted) VALUES (?, ?, ?)",
        (match_id, player_id, 1 if batted else 0),
    )
  elif batted:
    cur.execute(
        "UPDATE player_match_stats SET batted = 1 WHERE match_id = ? AND player_id = ?",
        (match_id, player_id),
    )


def max_wickets_for(cur, team_id):
  cur.execute("SELECT COUNT(*) FROM players WHERE team_id = ?", (team_id,))
  n = cur.fetchone()[0]
  return max(1, min(10, n - 1))


def delete_match_rows(cur, match_id):
  cur.execute("DELETE FROM delivery_history WHERE match_id = ?", (match_id,))
  cur.execute("DELETE FROM player_match_stats WHERE match_id = ?", (match_id,))
  cur.execute("DELETE FROM match_state WHERE match_id = ?", (match_id,))
  cur.execute("DELETE FROM matches WHERE id = ?", (match_id,))


def fmt_overs(legal_balls):
  return f"{legal_balls // 6}.{legal_balls % 6}"


class AutoScrollbar(ttk.Scrollbar):
  """A scrollbar that automatically hides itself when content fits without geometry conflicts."""
  def set(self, lo, hi):
    if float(lo) <= 0.0 and float(hi) >= 1.0:
      self.grid_remove()
    else:
      self.grid()
    super().set(lo, hi)


# ================= MAIN MOBILE APPLICATION =================
class CricketApp:

  def __init__(self, root):
    self.root = root
    self.root.title("Cricket Scorer")

    # --- Spacious Handheld Dimensions (420x750) ---
    self.root.minsize(380, 680)
    w, h = 420, 750
    sw, sh = root.winfo_screenwidth(), root.winfo_screenheight()
    self.root.geometry(f"{w}x{h}+{(sw - w) // 2}+{max((sh - h) // 2 - 20, 0)}")
    self.root.resizable(False, False)

    # Warm Minimalist Sage Palette
    self.COLOR_BG = "#f5f3ef"          # Soft warm cream
    self.COLOR_CARD = "#ffffff"        # Crisp white mobile cards
    self.COLOR_TEXT_MAIN = "#2d3732"   # Slate text
    self.COLOR_TEXT_MUTED = "#6b7c73"  # Muted secondary text
    self.COLOR_PRIMARY = "#4a7c59"     # Sage green primary
    self.COLOR_ACCENT = "#3d6b4b"      # Darker sage accent
    self.COLOR_DANGER = "#b84a4a"      # Terracotta red
    self.COLOR_BORDER = "#e2ddd5"      # Soft warm border
    self.COLOR_INPUT_BG = "#faf9f6"    # Mobile input background
    self.COLOR_STATUS_BAR = "#eae6df"  # Android Status Bar

    self.root.config(bg=self.COLOR_BG)
    self.configure_styles()

    self.current_match_id = None
    self.selected_team_id = None
    self.setup = {}

    # Root Phone Layout Container
    self.build_phone_frame()
    self.show_main_menu()

  # ================= STYLING =================
  def configure_styles(self):
    style = ttk.Style()
    style.theme_use("clam")
    self.root.option_add("*TCombobox*Listbox.font", (FONT, 10))
    self.root.option_add("*TCombobox*Listbox.background", self.COLOR_CARD)
    self.root.option_add("*TCombobox*Listbox.foreground", self.COLOR_TEXT_MAIN)

    style.configure(
        "TCombobox",
        fieldbackground=self.COLOR_INPUT_BG,
        background=self.COLOR_INPUT_BG,
        foreground=self.COLOR_TEXT_MAIN,
        bordercolor=self.COLOR_BORDER,
        arrowcolor=self.COLOR_PRIMARY,
        padding=4,
    )
    style.map("TCombobox", fieldbackground=[("readonly", self.COLOR_INPUT_BG)])

    style.configure(
        "Vertical.TScrollbar",
        gripcount=0,
        background="#d8e2dc",
        darkcolor="#d8e2dc",
        lightcolor="#d8e2dc",
        troughcolor=self.COLOR_INPUT_BG,
        bordercolor=self.COLOR_BORDER,
        arrowcolor=self.COLOR_PRIMARY,
    )

    style.configure(
        "Treeview",
        background=self.COLOR_CARD,
        foreground=self.COLOR_TEXT_MAIN,
        rowheight=26,
        fieldbackground=self.COLOR_CARD,
        borderwidth=0,
        font=(FONT, 9),
    )
    style.configure(
        "Treeview.Heading",
        background="#eae5dc",
        foreground=self.COLOR_PRIMARY,
        font=(FONT, 9, "bold"),
        borderwidth=0,
        relief="flat",
    )
    style.map(
        "Treeview",
        background=[("selected", "#d8e2dc")],
        foreground=[("selected", self.COLOR_TEXT_MAIN)],
    )

  # ================= ANDROID PHONE HARDWARE SHELL =================
  def build_phone_frame(self):
    # 1. Android Top Status Bar (26px)
    self.status_bar = tk.Frame(self.root, bg=self.COLOR_STATUS_BAR, height=26)
    self.status_bar.pack(fill=tk.X, side=tk.TOP)
    self.status_bar.pack_propagate(False)

    self.clock_label = tk.Label(
        self.status_bar,
        text=datetime.now().strftime("%I:%M %p"),
        font=(FONT, 9, "bold"),
        bg=self.COLOR_STATUS_BAR,
        fg=self.COLOR_TEXT_MAIN,
    )
    self.clock_label.pack(side=tk.LEFT, padx=12)

    # Punch-hole front camera
    camera_dot = tk.Label(
        self.status_bar,
        text="●",
        font=(FONT, 8),
        bg=self.COLOR_STATUS_BAR,
        fg="#1a1a1a",
    )
    camera_dot.pack(side=tk.LEFT, expand=True)

    icons_label = tk.Label(
        self.status_bar,
        text="📶  5G  🔋",
        font=(FONT, 8, "bold"),
        bg=self.COLOR_STATUS_BAR,
        fg=self.COLOR_TEXT_MAIN,
    )
    icons_label.pack(side=tk.RIGHT, padx=12)

    self.update_clock()

    # 2. Android Bottom Gesture Navigation Bar (18px)
    self.bottom_bar = tk.Frame(self.root, bg=self.COLOR_STATUS_BAR, height=18)
    self.bottom_bar.pack(fill=tk.X, side=tk.BOTTOM)
    self.bottom_bar.pack_propagate(False)

    pill = tk.Frame(self.bottom_bar, bg="#8c9a92", width=110, height=4)
    pill.place(relx=0.5, rely=0.5, anchor="center")

    # 3. Mobile Viewport (420px wide)
    self.viewport = tk.Frame(self.root, bg=self.COLOR_BG)
    self.viewport.pack(fill=tk.BOTH, expand=True)

  def update_clock(self):
    if hasattr(self, "clock_label"):
      self.clock_label.config(text=datetime.now().strftime("%I:%M %p"))
      self.root.after(30000, self.update_clock)

  def make_scrollable_screen(self, title, show_back=True, back_cmd=None):
    """Creates a native mobile scrolling view with clean grid geometry to prevent pack conflicts."""
    for widget in self.viewport.winfo_children():
      widget.destroy()

    # Android App Bar (Top of screen)
    app_bar = tk.Frame(self.viewport, bg=self.COLOR_PRIMARY, height=42)
    app_bar.pack(fill=tk.X, side=tk.TOP)
    app_bar.pack_propagate(False)

    if show_back:
      cmd = back_cmd or self.show_main_menu
      back_btn = tk.Button(
          app_bar,
          text="←",
          command=cmd,
          font=(FONT, 13, "bold"),
          bg=self.COLOR_PRIMARY,
          fg="#ffffff",
          activebackground=self.COLOR_ACCENT,
          activeforeground="#ffffff",
          bd=0,
          cursor="hand2",
          padx=10,
      )
      back_btn.pack(side=tk.LEFT)

    title_lbl = tk.Label(
        app_bar,
        text=title,
        font=(FONT, 11, "bold"),
        bg=self.COLOR_PRIMARY,
        fg="#ffffff",
    )
    title_lbl.pack(side=tk.LEFT, padx=6)

    # Grid-managed scroll container pinned to 418px width
    container = tk.Frame(self.viewport, bg=self.COLOR_BG)
    container.pack(fill=tk.BOTH, expand=True)
    container.grid_rowconfigure(0, weight=1)
    container.grid_columnconfigure(0, weight=1)

    canvas = tk.Canvas(container, bg=self.COLOR_BG, highlightthickness=0)
    scrollbar = AutoScrollbar(container, orient="vertical", command=canvas.yview)
    content = tk.Frame(canvas, bg=self.COLOR_BG)

    content.bind(
        "<Configure>",
        lambda e: canvas.configure(scrollregion=canvas.bbox("all")),
    )
    canvas.create_window((0, 0), window=content, anchor="nw", width=418)
    canvas.configure(yscrollcommand=scrollbar.set)

    canvas.grid(row=0, column=0, sticky="nsew")
    scrollbar.grid(row=0, column=1, sticky="ns")

    def _on_mousewheel(event):
      canvas.yview_scroll(int(-1 * (event.delta / 120)), "units")

    canvas.bind_all("<MouseWheel>", _on_mousewheel)

    return content

  # ================= HELPER UI BUILDERS =================
  @staticmethod
  def _darken(hex_color, factor=0.85):
    hex_color = hex_color.lstrip("#")
    r, g, b = (int(hex_color[i : i + 2], 16) for i in (0, 2, 4))
    return "#{:02x}{:02x}{:02x}".format(
        int(r * factor), int(g * factor), int(b * factor)
    )

  def create_styled_button(
      self, parent, text, command, bg_color=None, fg_color="#ffffff", width=26, pady=8
  ):
    if bg_color is None:
      bg_color = self.COLOR_PRIMARY
    hover = self._darken(bg_color, 0.9)
    btn = tk.Button(
        parent,
        text=text,
        command=command,
        font=(FONT, 10, "bold"),
        bg=bg_color,
        fg=fg_color,
        activebackground=hover,
        activeforeground=fg_color,
        bd=0,
        relief="flat",
        padx=12,
        pady=pady,
        cursor="hand2",
        width=width,
    )
    btn.bind("<Enter>", lambda e: btn.config(bg=hover))
    btn.bind("<Leave>", lambda e: btn.config(bg=bg_color))
    return btn

  def make_card(self, parent, padx=14, pady=12):
    return tk.Frame(
        parent,
        bg=self.COLOR_CARD,
        highlightbackground=self.COLOR_BORDER,
        highlightthickness=1,
        padx=padx,
        pady=pady,
    )

  def make_label(self, parent, text, size=10, bold=False, fg=None, **kw):
    return tk.Label(
        parent,
        text=text,
        font=(FONT, size, "bold" if bold else "normal"),
        bg=kw.pop("bg", parent.cget("bg")),
        fg=fg or self.COLOR_TEXT_MAIN,
        **kw,
    )

  def make_styled_entry(self, parent, width=18):
    return tk.Entry(
        parent,
        font=(FONT, 10),
        width=width,
        bg=self.COLOR_INPUT_BG,
        fg=self.COLOR_TEXT_MAIN,
        insertbackground=self.COLOR_TEXT_MAIN,
        relief="flat",
        highlightbackground=self.COLOR_BORDER,
        highlightcolor=self.COLOR_PRIMARY,
        highlightthickness=1,
    )

  def make_table(self, parent, columns, widths, height):
    frame = tk.Frame(parent, bg=self.COLOR_CARD)
    frame.grid_rowconfigure(0, weight=1)
    frame.grid_columnconfigure(0, weight=1)

    tree = ttk.Treeview(frame, columns=columns, show="headings", height=height)
    sb = AutoScrollbar(frame, orient="vertical", command=tree.yview)
    tree.configure(yscrollcommand=sb.set)
    for col, w in zip(columns, widths):
      tree.heading(col, text=col)
      tree.column(
          col,
          width=w,
          anchor="w" if col in ("Player", "Team", "Status", "Bowler") else "center",
      )
    tree.grid(row=0, column=0, sticky="nsew")
    sb.grid(row=0, column=1, sticky="ns")
    return frame, tree

  def _modal(self, title, width=350, height=210):
    top = tk.Toplevel(self.root)
    top.title(title)
    top.config(bg=self.COLOR_BG)
    top.resizable(False, False)
    top.transient(self.root)
    self.root.update_idletasks()
    x = self.root.winfo_rootx() + (self.root.winfo_width() - width) // 2
    y = self.root.winfo_rooty() + (self.root.winfo_height() - height) // 3
    top.geometry(f"{width}x{height}+{x}+{y}")
    top.wait_visibility()
    top.grab_set()
    return top

  def ask_player(self, title, prompt, players, subtitle=""):
    result = {"id": None}
    top = self._modal(title, 350, 210)
    top.protocol("WM_DELETE_WINDOW", lambda: None)
    self.make_label(top, prompt, 11, True, fg=self.COLOR_PRIMARY).pack(pady=(14, 2))
    if subtitle:
      self.make_label(top, subtitle, 9, fg=self.COLOR_TEXT_MUTED).pack(pady=(0, 4))

    cb = ttk.Combobox(
        top, values=[p[1] for p in players], state="readonly", font=(FONT, 9), width=32
    )
    cb.pack(pady=5)
    if players:
      cb.current(0)

    def confirm():
      idx = cb.current()
      if idx >= 0:
        result["id"] = players[idx][0]
        top.destroy()

    self.create_styled_button(top, "Confirm", confirm, self.COLOR_ACCENT, width=14, pady=6).pack(pady=12)
    self.root.wait_window(top)
    return result["id"]

  def ask_runs(self, title, prompt, options):
    result = {"v": None}
    top = self._modal(title, 350, 180)
    top.protocol("WM_DELETE_WINDOW", top.destroy)
    self.make_label(top, prompt, 11, True, fg=self.COLOR_PRIMARY).pack(pady=(16, 10))
    row = tk.Frame(top, bg=self.COLOR_BG)
    row.pack()
    for v in options:
      btn = tk.Button(
          row,
          text=str(v),
          width=4,
          font=(FONT, 10, "bold"),
          bg="#eae5dc",
          fg=self.COLOR_TEXT_MAIN,
          activebackground=self.COLOR_PRIMARY,
          activeforeground="#ffffff",
          bd=0,
          pady=5,
          cursor="hand2",
          command=lambda x=v: (result.update({"v": x}), top.destroy()),
      )
      btn.pack(side=tk.LEFT, padx=3)
    self.create_styled_button(top, "Cancel", top.destroy, "#b5b0a8", width=10, pady=4).pack(pady=14)
    self.root.wait_window(top)
    return result["v"]

  # ================= SCREEN 1: MOBILE MAIN MENU =================
  def show_main_menu(self):
    screen = self.make_scrollable_screen("Cricket Scorer", show_back=False)

    hero = tk.Frame(screen, bg=self.COLOR_BG, pady=16)
    hero.pack(fill=tk.X)

    logo = tk.Canvas(hero, width=54, height=54, bg=self.COLOR_BG, highlightthickness=0)
    logo.create_oval(3, 3, 51, 51, fill=self.COLOR_PRIMARY, outline="")
    logo.create_text(27, 27, text="🏏", font=("Segoe UI", 20))
    logo.pack(pady=(0, 4))

    self.make_label(hero, "Cricket Scoring App", 16, True, fg=self.COLOR_PRIMARY).pack()
    self.make_label(hero, "Android Edition", 9, fg=self.COLOR_TEXT_MUTED).pack()

    card = self.make_card(screen, padx=16, pady=12)
    card.pack(fill=tk.X, padx=20, pady=8)
    self.make_label(card, "Quick Menu", 11, True, fg=self.COLOR_TEXT_MUTED).pack(anchor="w", pady=(0, 8))

    menu = [
        ("👥  Manage Teams & Roster", self.show_team_management, self.COLOR_PRIMARY),
        ("▶  Start Live Match", self.show_match_setup, self.COLOR_ACCENT),
        ("📊  Player Career Statistics", self.show_career_stats, "#506b5c"),
        ("📋  Past & Saved Matches", self.show_past_matches, "#5a6e62"),
        ("☁  Cloud Backup & Sync", self.show_cloud_sync_dialog, "#64776b"),
        ("✖  Exit Application", self.root.destroy, self.COLOR_DANGER),
    ]
    for text, cmd, color in menu:
      self.create_styled_button(card, text, cmd, color, width=28, pady=8).pack(pady=5)

  # ================= SCREEN 2: MOBILE TEAM & ROSTER TABS =================
  def show_team_management(self):
    self.team_manage_tab = "teams"
    self.render_team_tab_screen()

  def render_team_tab_screen(self):
    screen = self.make_scrollable_screen("Teams & Roster")

    tab_bar = tk.Frame(screen, bg="#eae5dc", height=36)
    tab_bar.pack(fill=tk.X, padx=18, pady=(10, 6))

    t_btn_bg = self.COLOR_PRIMARY if self.team_manage_tab == "teams" else "#eae5dc"
    t_btn_fg = "#ffffff" if self.team_manage_tab == "teams" else self.COLOR_TEXT_MAIN
    r_btn_bg = self.COLOR_PRIMARY if self.team_manage_tab == "roster" else "#eae5dc"
    r_btn_fg = "#ffffff" if self.team_manage_tab == "roster" else self.COLOR_TEXT_MAIN

    tk.Button(
        tab_bar,
        text="🏟  Teams",
        font=(FONT, 9, "bold"),
        bg=t_btn_bg,
        fg=t_btn_fg,
        bd=0,
        relief="flat",
        cursor="hand2",
        command=lambda: self.switch_team_tab("teams"),
    ).pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

    tk.Button(
        tab_bar,
        text="👥  Players Roster",
        font=(FONT, 9, "bold"),
        bg=r_btn_bg,
        fg=r_btn_fg,
        bd=0,
        relief="flat",
        cursor="hand2",
        command=lambda: self.switch_team_tab("roster"),
    ).pack(side=tk.LEFT, fill=tk.BOTH, expand=True)

    if self.team_manage_tab == "teams":
      self.render_teams_tab_content(screen)
    else:
      self.render_roster_tab_content(screen)

  def switch_team_tab(self, tab):
    self.team_manage_tab = tab
    self.render_team_tab_screen()

  def render_teams_tab_content(self, screen):
    card = self.make_card(screen, padx=14, pady=12)
    card.pack(fill=tk.X, padx=20, pady=6)

    self.make_label(card, "Add New Team", 11, True, fg=self.COLOR_PRIMARY).pack(anchor="w", pady=(0, 6))

    row = tk.Frame(card, bg=self.COLOR_CARD)
    row.pack(fill=tk.X, pady=(0, 8))
    self.team_name_entry = self.make_styled_entry(row, width=20)
    self.team_name_entry.pack(side=tk.LEFT, padx=(0, 6))
    self.team_name_entry.bind("<Return>", lambda e: self.add_team())
    self.create_styled_button(row, "+ Add", self.add_team, self.COLOR_ACCENT, width=8, pady=4).pack(side=tk.LEFT)

    self.team_header = self.make_label(card, "Registered Teams (0)", 10, True, fg=self.COLOR_TEXT_MUTED)
    self.team_header.pack(anchor="w", pady=(4, 4))

    t_box = tk.Frame(card, bg=self.COLOR_CARD)
    t_box.pack(fill=tk.X, pady=4)
    t_box.grid_rowconfigure(0, weight=1)
    t_box.grid_columnconfigure(0, weight=1)

    sb = AutoScrollbar(t_box, orient="vertical")
    self.team_listbox = tk.Listbox(
        t_box,
        font=(FONT, 9),
        height=7,
        bg=self.COLOR_INPUT_BG,
        fg=self.COLOR_TEXT_MAIN,
        bd=1,
        relief="solid",
        highlightbackground=self.COLOR_BORDER,
        highlightthickness=1,
        activestyle="none",
        exportselection=False,
        selectbackground="#d8e2dc",
        selectforeground=self.COLOR_TEXT_MAIN,
        yscrollcommand=sb.set,
    )
    sb.config(command=self.team_listbox.yview)
    self.team_listbox.grid(row=0, column=0, sticky="nsew")
    sb.grid(row=0, column=1, sticky="ns")
    self.team_listbox.bind("<<ListboxSelect>>", self.on_team_select_tab)

    btn_row = tk.Frame(card, bg=self.COLOR_CARD)
    btn_row.pack(fill=tk.X, pady=(10, 2))
    self.create_styled_button(
        btn_row, "Manage Players →", lambda: self.switch_team_tab("roster"), self.COLOR_PRIMARY, width=15, pady=6
    ).pack(side=tk.LEFT, padx=(0, 6))
    self.create_styled_button(
        btn_row, "Delete Team", self.delete_team, self.COLOR_DANGER, width=13, pady=6
    ).pack(side=tk.LEFT)

    self.populate_teams()

  def render_roster_tab_content(self, screen):
    card = self.make_card(screen, padx=14, pady=12)
    card.pack(fill=tk.X, padx=20, pady=6)

    with connect() as cur:
      cur.execute("SELECT id, name FROM teams ORDER BY id")
      teams = cur.fetchall()

    if not teams:
      self.make_label(card, "No teams registered yet. Add a team first!", 10, fg=self.COLOR_DANGER).pack(pady=8)
      return

    team_names = [t[1] for t in teams]
    cur_idx = 0
    if self.selected_team_id:
      for idx, t in enumerate(teams):
        if t[0] == self.selected_team_id:
          cur_idx = idx
          break
    else:
      self.selected_team_id = teams[0][0]

    select_row = tk.Frame(card, bg=self.COLOR_CARD)
    select_row.pack(fill=tk.X, pady=(0, 8))
    self.make_label(select_row, "Active Team:").pack(side=tk.LEFT, padx=(0, 6))
    team_sel_cb = ttk.Combobox(select_row, values=team_names, state="readonly", font=(FONT, 9), width=18)
    team_sel_cb.pack(side=tk.LEFT)
    team_sel_cb.current(cur_idx)

    def on_roster_team_change(e):
      idx = team_sel_cb.current()
      if idx >= 0:
        self.selected_team_id = teams[idx][0]
        self.refresh_roster()

    team_sel_cb.bind("<<ComboboxSelected>>", on_roster_team_change)

    self.make_label(card, "Add Player to Team", 11, True, fg=self.COLOR_PRIMARY).pack(anchor="w", pady=(6, 4))
    p_row = tk.Frame(card, bg=self.COLOR_CARD)
    p_row.pack(fill=tk.X, pady=(0, 6))
    self.player_name_entry = self.make_styled_entry(p_row, width=13)
    self.player_name_entry.pack(side=tk.LEFT, padx=(0, 4))
    self.player_name_entry.bind("<Return>", lambda e: self.add_player())

    self.player_role_cb = ttk.Combobox(p_row, values=PLAYER_ROLES, state="readonly", font=(FONT, 9), width=10)
    self.player_role_cb.pack(side=tk.LEFT, padx=(0, 4))
    self.player_role_cb.current(0)
    self.create_styled_button(p_row, "+ Add", self.add_player, self.COLOR_PRIMARY, width=7, pady=3).pack(side=tk.LEFT)

    self.roster_header = self.make_label(card, "Team Roster (Min 2 required)", 10, True, fg=self.COLOR_TEXT_MUTED)
    self.roster_header.pack(anchor="w", pady=(6, 4))

    p_box = tk.Frame(card, bg=self.COLOR_CARD)
    p_box.pack(fill=tk.X, pady=4)
    p_box.grid_rowconfigure(0, weight=1)
    p_box.grid_columnconfigure(0, weight=1)

    sb = AutoScrollbar(p_box, orient="vertical")
    self.player_listbox = tk.Listbox(
        p_box,
        font=(FONT, 9),
        height=7,
        bg=self.COLOR_INPUT_BG,
        fg=self.COLOR_TEXT_MAIN,
        bd=1,
        relief="solid",
        highlightbackground=self.COLOR_BORDER,
        highlightthickness=1,
        activestyle="none",
        exportselection=False,
        selectbackground="#d8e2dc",
        selectforeground=self.COLOR_TEXT_MAIN,
        yscrollcommand=sb.set,
    )
    sb.config(command=self.player_listbox.yview)
    self.player_listbox.grid(row=0, column=0, sticky="nsew")
    sb.grid(row=0, column=1, sticky="ns")

    self.create_styled_button(card, "Remove Selected Player", self.delete_player, "#8c9a92", width=28, pady=6).pack(
        pady=(8, 2)
    )
    self.refresh_roster()

  def on_team_select_tab(self, event=None):
    sel = self.team_listbox.curselection()
    if sel:
      self.selected_team_id = self.teams_data[sel[0]][0]

  def populate_teams(self, select_id=None):
    if not hasattr(self, "team_listbox"):
      return
    self.team_listbox.delete(0, tk.END)
    with connect() as cur:
      cur.execute("SELECT id, name FROM teams ORDER BY id")
      self.teams_data = cur.fetchall()
    for i, (_, name) in enumerate(self.teams_data, start=1):
      self.team_listbox.insert(tk.END, f"  {i}. {name}")
    if hasattr(self, "team_header"):
      self.team_header.config(text=f"Registered Teams ({len(self.teams_data)})")

    self.selected_team_id = None
    for idx, (tid, _) in enumerate(self.teams_data):
      if tid == select_id:
        self.team_listbox.selection_set(idx)
        self.team_listbox.see(idx)
        self.selected_team_id = tid
        break

  def refresh_roster(self):
    if not hasattr(self, "player_listbox"):
      return
    self.player_listbox.delete(0, tk.END)
    self.players_data = []
    if self.selected_team_id is None:
      return
    with connect() as cur:
      cur.execute(
          "SELECT id, name, COALESCE(role, 'All-Rounder') FROM players WHERE team_id = ? ORDER BY id",
          (self.selected_team_id,),
      )
      self.players_data = cur.fetchall()
    for _, name, role in self.players_data:
      icon = "🎯" if role == "Bowler" else ("🏏" if role == "Batsman" else ("🧤" if role == "Wicketkeeper" else "⚡"))
      self.player_listbox.insert(tk.END, f"  {icon}  {name}  [{role}]")
    if hasattr(self, "roster_header"):
      self.roster_header.config(text=f"Team Roster ({len(self.players_data)} players, min 2)")

  def add_team(self):
    name = self.team_name_entry.get().strip()
    if not name:
      messagebox.showerror("Error", "Team name cannot be empty!")
      return
    if len(name) > 30:
      messagebox.showerror("Error", "Team name is too long (max 30 characters).")
      return
    try:
      with connect() as cur:
        cur.execute("INSERT INTO teams (name) VALUES (?)", (name,))
        new_id = cur.lastrowid
    except sqlite3.IntegrityError:
      messagebox.showerror("Error", "Team name already exists!")
      return
    self.team_name_entry.delete(0, tk.END)
    self.populate_teams(select_id=new_id)

  def delete_team(self):
    sel = self.team_listbox.curselection()
    if not sel:
      messagebox.showerror("Error", "Please select a team to delete!")
      return
    team_id, team_name = self.teams_data[sel[0]]

    if not messagebox.askyesno(
        "Confirm Delete",
        f"Delete '{team_name}' from your active roster?\n\n(Past match records will remain safe in history!)",
    ):
      return

    with connect() as cur:
      cur.execute(
          "UPDATE matches SET team1_name = ? WHERE team1_id = ? AND (team1_name IS NULL OR team1_name = '')",
          (team_name, team_id),
      )
      cur.execute(
          "UPDATE matches SET team2_name = ? WHERE team2_id = ? AND (team2_name IS NULL OR team2_name = '')",
          (team_name, team_id),
      )
      cur.execute("DELETE FROM players WHERE team_id = ?", (team_id,))
      cur.execute("DELETE FROM teams WHERE id = ?", (team_id,))

    self.populate_teams()
    messagebox.showinfo("Success", f"Team '{team_name}' deleted.")

  def add_player(self):
    if self.selected_team_id is None:
      messagebox.showerror("Error", "Please select a team first!")
      return
    p_name = self.player_name_entry.get().strip()
    p_role = self.player_role_cb.get().strip() or "All-Rounder"
    if not p_name:
      messagebox.showerror("Error", "Player name cannot be empty!")
      return
    if len(self.players_data) >= MAX_PLAYERS:
      messagebox.showerror("Error", f"A team can have at most {MAX_PLAYERS} players.")
      return
    if any(p_name.lower() == n.lower() for _, n, _ in self.players_data):
      messagebox.showerror("Error", "That player is already in this team!")
      return
    with connect() as cur:
      cur.execute(
          "INSERT INTO players (team_id, name, role) VALUES (?, ?, ?)",
          (self.selected_team_id, p_name, p_role),
      )
    self.player_name_entry.delete(0, tk.END)
    self.refresh_roster()

  def delete_player(self):
    sel = self.player_listbox.curselection()
    if not sel:
      messagebox.showerror("Error", "Please select a player to remove!")
      return
    pid, pname, _ = self.players_data[sel[0]]
    if messagebox.askyesno("Confirm", f"Remove player '{pname}' from roster?"):
      with connect() as cur:
        cur.execute("DELETE FROM players WHERE id = ?", (pid,))
      self.refresh_roster()

  # ================= SCREEN 3: MOBILE MATCH SETUP =================
  def show_match_setup(self):
    with connect() as cur:
      cur.execute("SELECT id, name FROM teams ORDER BY id")
      teams = cur.fetchall()
    if len(teams) < 2:
      messagebox.showerror("Error", "You need at least 2 teams with players configured first!\nGo to Manage Teams & Players.")
      return
    self.setup_teams = teams

    screen = self.make_scrollable_screen("Match Setup")

    card = self.make_card(screen, padx=16, pady=12)
    card.pack(fill=tk.X, padx=20, pady=10)
    names = [t[1] for t in teams]

    self.make_label(card, "Team 1:").pack(anchor="w", pady=(2, 2))
    self.t1_cb = ttk.Combobox(card, values=names, font=(FONT, 10), state="readonly", width=28)
    self.t1_cb.pack(fill=tk.X, pady=(0, 8))

    self.make_label(card, "Team 2:").pack(anchor="w", pady=(2, 2))
    self.t2_cb = ttk.Combobox(card, values=names, font=(FONT, 10), state="readonly", width=28)
    self.t2_cb.pack(fill=tk.X, pady=(0, 8))

    self.make_label(card, "Venue:").pack(anchor="w", pady=(2, 2))
    self.venue_entry = self.make_styled_entry(card, width=28)
    self.venue_entry.pack(fill=tk.X, pady=(0, 8))
    self.venue_entry.insert(0, "National Stadium")

    self.make_label(card, "Format:").pack(anchor="w", pady=(2, 2))
    self.format_cb = ttk.Combobox(
        card, values=list(FORMAT_PRESETS.keys()), font=(FONT, 10), state="readonly", width=28
    )
    self.format_cb.pack(fill=tk.X, pady=(0, 8))
    self.format_cb.current(0)
    self.format_cb.bind("<<ComboboxSelected>>", self.on_format_change)

    self.make_label(card, f"Total Overs (1-{MAX_OVERS}):").pack(anchor="w", pady=(2, 2))
    self.overs_entry = self.make_styled_entry(card, width=28)
    self.overs_entry.pack(fill=tk.X, pady=(0, 10))
    self.overs_entry.insert(0, "20")

    self.create_styled_button(card, "Proceed to Toss  →", self.validate_setup, self.COLOR_ACCENT, width=28).pack(pady=4)

  def on_format_change(self, event=None):
    fmt = self.format_cb.get()
    if fmt in FORMAT_PRESETS and fmt != "Custom Overs":
      self.overs_entry.delete(0, tk.END)
      self.overs_entry.insert(0, str(FORMAT_PRESETS[fmt]))

  def validate_setup(self):
    i1, i2 = self.t1_cb.current(), self.t2_cb.current()
    if i1 < 0 or i2 < 0 or i1 == i2:
      messagebox.showerror("Error", "Please select two distinct teams!")
      return
    try:
      overs = int(self.overs_entry.get().strip())
      if not 1 <= overs <= MAX_OVERS:
        raise ValueError
    except ValueError:
      messagebox.showerror("Error", f"Overs must be a whole number from 1 to {MAX_OVERS}!")
      return

    t1, t2 = self.setup_teams[i1], self.setup_teams[i2]
    with connect() as cur:
      for tid, tname in (t1, t2):
        cur.execute("SELECT COUNT(*) FROM players WHERE team_id = ?", (tid,))
        if cur.fetchone()[0] < 2:
          messagebox.showerror("Error", f"Team '{tname}' needs at least 2 players registered!")
          return

    self.setup = {
        "t1": t1,
        "t2": t2,
        "venue": self.venue_entry.get().strip() or "Unknown Venue",
        "overs": overs,
        "format": self.format_cb.get(),
    }
    self.show_toss_screen()

  def show_toss_screen(self):
    screen = self.make_scrollable_screen("Match Toss")
    card = self.make_card(screen, padx=16, pady=14)
    card.pack(fill=tk.X, padx=20, pady=12)
    t1, t2 = self.setup["t1"], self.setup["t2"]

    self.make_label(card, "Toss Winner:").pack(anchor="w", pady=(2, 2))
    self.toss_winner_cb = ttk.Combobox(
        card, values=[t1[1], t2[1]], font=(FONT, 10), state="readonly", width=28
    )
    self.toss_winner_cb.pack(fill=tk.X, pady=(0, 10))
    self.toss_winner_cb.current(0)

    self.make_label(card, "Elected to:").pack(anchor="w", pady=(2, 2))
    self.toss_choice_cb = ttk.Combobox(
        card, values=["Bat", "Bowl"], font=(FONT, 10), state="readonly", width=28
    )
    self.toss_choice_cb.pack(fill=tk.X, pady=(0, 14))
    self.toss_choice_cb.current(0)

    self.create_styled_button(card, "Confirm & Select Lineup →", self.confirm_toss, self.COLOR_ACCENT, width=28).pack(pady=4)

  def confirm_toss(self):
    s = self.setup
    winner, loser = (s["t1"], s["t2"]) if self.toss_winner_cb.current() == 0 else (s["t2"], s["t1"])
    if self.toss_choice_cb.get() == "Bat":
      s["bat"], s["bowl"] = winner, loser
    else:
      s["bat"], s["bowl"] = loser, winner
    self.show_lineup_form(
        f"Innings 1 Lineup — {s['bat'][1]} Batting",
        s["bat"][0],
        s["bowl"][0],
        self.start_match,
    )

  def show_lineup_form(self, title, bat_team_id, bowl_team_id, on_confirm):
    with connect() as cur:
      cur.execute(
          "SELECT id, name, COALESCE(role, 'All-Rounder') FROM players WHERE team_id = ? ORDER BY id",
          (bat_team_id,),
      )
      batters = cur.fetchall()
      cur.execute(
          """
          SELECT id, name, COALESCE(role, 'All-Rounder') FROM players
          WHERE team_id = ?
          ORDER BY CASE WHEN role = 'Bowler' THEN 1 WHEN role = 'All-Rounder' THEN 2 ELSE 3 END, id
      """,
          (bowl_team_id,),
      )
      bowlers = cur.fetchall()

    screen = self.make_scrollable_screen("Team Lineup")
    card = self.make_card(screen, padx=16, pady=12)
    card.pack(fill=tk.X, padx=20, pady=10)

    self.make_label(card, title, 11, True, fg=self.COLOR_PRIMARY).pack(anchor="w", pady=(0, 8))

    def combo(label, people):
      self.make_label(card, label).pack(anchor="w", pady=(4, 2))
      cb = ttk.Combobox(
          card, values=[f"{p[1]} [{p[2]}]" for p in people], font=(FONT, 9), state="readonly", width=28
      )
      cb.pack(fill=tk.X, pady=(0, 6))
      return cb

    striker_cb = combo("Opening Striker:", batters)
    non_striker_cb = combo("Opening Non-Striker:", batters)
    bowler_cb = combo("Opening Bowler:", bowlers)

    def submit():
      si, ni, bi = striker_cb.current(), non_striker_cb.current(), bowler_cb.current()
      if min(si, ni, bi) < 0:
        messagebox.showerror("Error", "All player selections are required!")
        return
      if si == ni:
        messagebox.showerror("Error", "Striker and Non-Striker must be different players!")
        return
      on_confirm(batters[si][0], batters[ni][0], bowlers[bi][0])

    self.create_styled_button(card, "Start Live Match 🏏", submit, self.COLOR_ACCENT, width=28).pack(pady=(12, 4))

  def start_match(self, s_id, ns_id, b_id):
    s = self.setup
    with connect() as cur:
      cur.execute(
          "INSERT INTO matches (team1_id, team2_id, team1_name, team2_name, venue, total_overs, format, status)"
          " VALUES (?, ?, ?, ?, ?, ?, ?, 'Ongoing')",
          (s["t1"][0], s["t2"][0], s["t1"][1], s["t2"][1], s["venue"], s["overs"], s.get("format", "Custom Overs")),
      )
      mid = cur.lastrowid
      cur.execute(
          "INSERT INTO match_state (match_id, innings, batting_team_id,"
          " bowling_team_id, runs, wickets, legal_balls, striker_id,"
          " non_striker_id, bowler_id) VALUES (?, 1, ?, ?, 0, 0, 0, ?, ?, ?)",
          (mid, s["bat"][0], s["bowl"][0], s_id, ns_id, b_id),
      )
      ensure_stats(cur, mid, s_id, batted=True)
      ensure_stats(cur, mid, ns_id, batted=True)
      ensure_stats(cur, mid, b_id)
    self.current_match_id = mid
    self.show_scoring_dashboard()

  def start_innings2(self, s_id, ns_id, b_id):
    with connect() as cur:
      cur.execute(
          "UPDATE match_state SET striker_id = ?, non_striker_id = ?,"
          " bowler_id = ? WHERE match_id = ?",
          (s_id, ns_id, b_id, self.current_match_id),
      )
      ensure_stats(cur, self.current_match_id, s_id, batted=True)
      ensure_stats(cur, self.current_match_id, ns_id, batted=True)
      ensure_stats(cur, self.current_match_id, b_id)
    self.show_scoring_dashboard()

  # ================= SCREEN 4: MOBILE LIVE SCORING DASHBOARD =================
  def load_state(self):
    with connect() as cur:
      cur.execute(
          "SELECT innings, batting_team_id, bowling_team_id, runs, wickets,"
          " legal_balls, striker_id, non_striker_id, bowler_id FROM match_state"
          " WHERE match_id = ?",
          (self.current_match_id,),
      )
      state = cur.fetchone()
      if state is None:
        return False
      (
          self.innings, self.bat_id, self.bowl_id, self.runs, self.wickets,
          self.legal_balls, self.s_id, self.ns_id, self.b_id,
      ) = state

      cur.execute(
          "SELECT total_overs, target, format FROM matches WHERE id = ?",
          (self.current_match_id,),
      )
      self.total_overs, self.target, self.match_format = cur.fetchone()

      def one(sql, params, default="?"):
        cur.execute(sql, params)
        row = cur.fetchone()
        return row[0] if row else default

      self.bat_name = one("SELECT name FROM teams WHERE id = ?", (self.bat_id,))
      self.bowl_name = one("SELECT name FROM teams WHERE id = ?", (self.bowl_id,))
      self.s_name = one("SELECT name FROM players WHERE id = ?", (self.s_id,))
      self.ns_name = one("SELECT name FROM players WHERE id = ?", (self.ns_id,))
      self.b_name = one("SELECT name FROM players WHERE id = ?", (self.b_id,))
      self.b_role = one("SELECT role FROM players WHERE id = ?", (self.b_id,), "Bowler")
      self.max_wickets = max_wickets_for(cur, self.bat_id)

      striker_team = one("SELECT team_id FROM players WHERE id = ?", (self.s_id,), None)
      self.needs_lineup = striker_team != self.bat_id
      out = one(
          "SELECT out_status FROM player_match_stats WHERE match_id = ? AND player_id = ?",
          (self.current_match_id, self.s_id),
          "Not Out",
      )
      self.striker_out = out != "Not Out"
    return True

  def get_stats(self, cur, pid):
    cur.execute(
        "SELECT runs, balls_faced, runs_conceded, legal_balls_bowled,"
        " wickets_taken FROM player_match_stats WHERE match_id = ? AND player_id = ?",
        (self.current_match_id, pid),
    )
    return cur.fetchone() or (0, 0, 0, 0, 0)

  def show_scoring_dashboard(self):
    if not self.load_state():
      messagebox.showerror("Error", "This match has no saved scoring data.")
      self.show_main_menu()
      return

    if self.needs_lineup:
      self.show_lineup_form(
          f"Innings 2 Lineup — {self.bat_name} Chasing {self.target}",
          self.bat_id,
          self.bowl_id,
          self.start_innings2,
      )
      return
    if self.striker_out:
      if not self.replace_batter(self.s_id):
        return
      self.show_scoring_dashboard()
      return

    screen = self.make_scrollable_screen(f"{self.bat_name} (Inn {self.innings})")

    with connect() as cur:
      s_stats = self.get_stats(cur, self.s_id)
      ns_stats = self.get_stats(cur, self.ns_id)
      b_stats = self.get_stats(cur, self.b_id)

    max_ov = max(1, math.ceil(self.total_overs / 5))

    # Mobile Score Banner
    banner = tk.Frame(screen, bg=self.COLOR_PRIMARY, pady=10)
    banner.pack(fill=tk.X)

    self.make_label(
        banner, f"{self.bat_name.upper()} • {self.match_format}", 9, True, fg="#d8e2dc", bg=self.COLOR_PRIMARY
    ).pack()
    self.make_label(banner, f"{self.runs} / {self.wickets}", 26, True, fg="#ffffff", bg=self.COLOR_PRIMARY).pack()
    self.make_label(
        banner, f"Overs: {fmt_overs(self.legal_balls)} / {self.total_overs}", 10, fg="#e8f0ec", bg=self.COLOR_PRIMARY
    ).pack()

    ball_in_over = self.legal_balls % 6
    dots = "● " * ball_in_over + "○ " * (6 - ball_in_over)
    self.make_label(banner, dots.strip(), 11, fg="#ffffff", bg=self.COLOR_PRIMARY).pack(pady=(2, 0))

    run_rate = self.runs * 6 / self.legal_balls if self.legal_balls else 0.0
    extra_line = f"CRR: {run_rate:.2f}"
    if self.innings == 2 and self.target:
      balls_left = self.total_overs * 6 - self.legal_balls
      needed = self.target - self.runs
      rrr = f"{needed * 6 / balls_left:.2f}" if balls_left > 0 else "-"
      extra_line = f"Target: {self.target} | Need {needed} from {balls_left}b | RRR: {rrr}"
    self.make_label(banner, extra_line, 9, fg="#c7d2fe", bg=self.COLOR_PRIMARY).pack(pady=(2, 0))

    # Live Pitch Batter & Bowler Status
    status = self.make_card(screen, padx=12, pady=8)
    status.pack(fill=tk.X, padx=16, pady=6)
    p_info = (
        f"🏏 {self.s_name}*: {s_stats[0]} ({s_stats[1]}b)\n"
        f"🏃 {self.ns_name}: {ns_stats[0]} ({ns_stats[1]}b)\n"
        f"🎯 {self.b_name} [{self.b_role}]: {fmt_overs(b_stats[3])}/{max_ov} ov, {b_stats[2]}r, {b_stats[4]}w"
    )
    self.make_label(status, p_info, 10, justify=tk.LEFT).pack(anchor="w")

    # Mobile Scoring Keypad
    action = self.make_card(screen, padx=10, pady=8)
    action.pack(fill=tk.X, padx=16, pady=4)

    # Runs Grid (Touch Keypad)
    self.make_label(action, "Runs off Bat:", 9, True, fg=self.COLOR_TEXT_MUTED).pack(anchor="w", pady=(0, 2))
    runs_grid = tk.Frame(action, bg=self.COLOR_CARD)
    runs_grid.pack(fill=tk.X, pady=2)
    for idx, r in enumerate([0, 1, 2, 3, 4, 6]):
      tk.Button(
          runs_grid,
          text=str(r),
          font=(FONT, 10, "bold"),
          bg="#eae5dc",
          fg=self.COLOR_TEXT_MAIN,
          activebackground="#d8e2dc",
          bd=0,
          relief="flat",
          padx=6,
          pady=4,
          cursor="hand2",
          command=lambda val=r: self.record_ball(val, "normal", 0),
      ).grid(row=0, column=idx, padx=2, sticky="nsew")
      runs_grid.grid_columnconfigure(idx, weight=1)

    # Extras Grid
    self.make_label(action, "Extras:", 9, True, fg=self.COLOR_TEXT_MUTED).pack(anchor="w", pady=(6, 2))
    extras_grid = tk.Frame(action, bg=self.COLOR_CARD)
    extras_grid.pack(fill=tk.X, pady=2)
    extras_list = [
        ("Wide", self.extra_wide),
        ("No Ball", self.extra_noball),
        ("Bye", lambda: self.extra_bye("bye")),
        ("Leg Bye", lambda: self.extra_bye("legbye")),
    ]
    for idx, (lbl, cmd) in enumerate(extras_list):
      tk.Button(
          extras_grid,
          text=lbl,
          font=(FONT, 8, "bold"),
          bg="#f4eee4",
          fg="#5a6e62",
          activebackground="#e2ddd5",
          bd=0,
          relief="flat",
          padx=4,
          pady=4,
          cursor="hand2",
          command=cmd,
      ).grid(row=0, column=idx, padx=2, sticky="nsew")
      extras_grid.grid_columnconfigure(idx, weight=1)

    # Wickets Grid
    self.make_label(action, "Dismissal / Wicket:", 9, True, fg=self.COLOR_TEXT_MUTED).pack(anchor="w", pady=(6, 2))
    wicket_grid = tk.Frame(action, bg=self.COLOR_CARD)
    wicket_grid.pack(fill=tk.X, pady=2)
    wickets_list = ["Bowled", "Caught", "LBW", "Stumped", "Run Out"]
    for idx, w_type in enumerate(wickets_list):
      tk.Button(
          wicket_grid,
          text=w_type,
          font=(FONT, 8, "bold"),
          bg="#f9eaea",
          fg=self.COLOR_DANGER,
          activebackground="#f2d6d6",
          bd=0,
          relief="flat",
          padx=2,
          pady=4,
          cursor="hand2",
          command=lambda wt=w_type: self.record_wicket(wt),
      ).grid(row=0, column=idx, padx=2, sticky="nsew")
      wicket_grid.grid_columnconfigure(idx, weight=1)

    # Actions: Undo & DLS
    tools_row = tk.Frame(action, bg=self.COLOR_CARD)
    tools_row.pack(fill=tk.X, pady=(8, 2))
    self.create_styled_button(
        tools_row, "↶ Undo Ball", self.undo_last_ball, "#b8864a", width=14, pady=5
    ).pack(side=tk.LEFT, padx=(0, 4), expand=True, fill=tk.X)
    self.create_styled_button(
        tools_row, "🌧 DLS Target", self.show_dls_dialog, "#557876", width=14, pady=5
    ).pack(side=tk.LEFT, expand=True, fill=tk.X)

    # Bottom Screen Navigation Buttons
    nav = tk.Frame(screen, bg=self.COLOR_BG)
    nav.pack(fill=tk.X, padx=16, pady=8)
    self.create_styled_button(
        nav, "📄 View Scorecard", self.show_scorecard, self.COLOR_PRIMARY, width=14, pady=6
    ).pack(side=tk.LEFT, padx=(0, 4), expand=True, fill=tk.X)
    self.create_styled_button(
        nav, "🏠 Main Menu", self.show_main_menu, "#8c9a92", width=14, pady=6
    ).pack(side=tk.LEFT, expand=True, fill=tk.X)

  # ---------- EXTRAS ----------
  def extra_wide(self):
    add = self.ask_runs("Wide", "Extra runs taken on wide?", [0, 1, 2, 3, 4])
    if add is not None:
      self.record_ball(0, "wide", 1 + add)

  def extra_noball(self):
    r = self.ask_runs("No Ball", "Runs scored off the bat?", [0, 1, 2, 3, 4, 6])
    if r is not None:
      self.record_ball(r, "noball", 1)

  def extra_bye(self, kind):
    title = "Bye" if kind == "bye" else "Leg Bye"
    r = self.ask_runs(title, "How many runs were taken?", [1, 2, 3, 4])
    if r is not None:
      self.record_ball(0, kind, r)

  # ---------- RECORDING DELIVERIES & UNDO ----------
  def innings_over(self, runs, wickets, legal_balls):
    if self.innings == 2 and self.target and runs >= self.target:
      return True
    return wickets >= self.max_wickets or legal_balls >= self.total_overs * 6

  def record_ball(self, bat_runs, extra_type, extra_runs):
    is_legal = extra_type not in ("wide", "noball")
    new_runs = self.runs + bat_runs + extra_runs
    new_legal = self.legal_balls + (1 if is_legal else 0)

    if extra_type == "wide":
      ran = extra_runs - 1
    elif extra_type in ("bye", "legbye"):
      ran = extra_runs
    else:
      ran = bat_runs

    prev_s, prev_ns = self.s_id, self.ns_id
    s_id, ns_id = prev_s, prev_ns
    if ran % 2 == 1:
      s_id, ns_id = ns_id, s_id
    over_done = is_legal and new_legal % 6 == 0
    if over_done:
      s_id, ns_id = ns_id, s_id

    faced = 0 if extra_type == "wide" else 1
    conceded = bat_runs + (extra_runs if extra_type in ("wide", "noball") else 0)

    with connect() as cur:
      ensure_stats(cur, self.current_match_id, self.s_id, batted=True)
      ensure_stats(cur, self.current_match_id, self.b_id)
      cur.execute(
          """
          INSERT INTO delivery_history (
              match_id, innings, striker_id, non_striker_id, bowler_id,
              bat_runs, extra_type, extra_runs,
              is_wicket, wicket_type, out_player_id,
              prev_runs, prev_wickets, prev_legal, prev_striker_id, prev_non_striker_id
          ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, 0, '', NULL, ?, ?, ?, ?, ?)
      """,
          (
              self.current_match_id,
              self.innings,
              prev_s,
              prev_ns,
              self.b_id,
              bat_runs,
              extra_type,
              extra_runs,
              self.runs,
              self.wickets,
              self.legal_balls,
              prev_s,
              prev_ns,
          ),
      )

      cur.execute(
          "UPDATE player_match_stats SET runs = runs + ?, balls_faced ="
          " balls_faced + ?, ones = ones + ?, twos = twos + ?, threes = threes"
          " + ?, fours = fours + ?, sixes = sixes + ?, batted = 1 WHERE"
          " match_id = ? AND player_id = ?",
          (
              bat_runs,
              faced,
              int(bat_runs == 1),
              int(bat_runs == 2),
              int(bat_runs == 3),
              int(bat_runs == 4),
              int(bat_runs == 6),
              self.current_match_id,
              self.s_id,
          ),
      )
      cur.execute(
          "UPDATE player_match_stats SET runs_conceded = runs_conceded + ?,"
          " legal_balls_bowled = legal_balls_bowled + ? WHERE match_id = ? AND"
          " player_id = ?",
          (conceded, 1 if is_legal else 0, self.current_match_id, self.b_id),
      )
      cur.execute(
          "UPDATE match_state SET runs = ?, legal_balls = ?, striker_id = ?,"
          " non_striker_id = ? WHERE match_id = ?",
          (new_runs, new_legal, s_id, ns_id, self.current_match_id),
      )

    if self.innings_over(new_runs, self.wickets, new_legal):
      self.end_innings()
      return
    if over_done:
      messagebox.showinfo("Over Complete", f"End of over {new_legal // 6}! Select the next bowler.")
      self.prompt_new_bowler()
    self.show_scoring_dashboard()

  def record_wicket(self, wicket_type):
    out_id = self.s_id
    if wicket_type == "Run Out":
      out_id = self.ask_player(
          "Run Out",
          "Which batter is out?",
          [(self.s_id, self.s_name), (self.ns_id, self.ns_name)],
      )
    credit_bowler = wicket_type != "Run Out"
    new_wickets = self.wickets + 1
    new_legal = self.legal_balls + 1
    over_done = new_legal % 6 == 0

    with connect() as cur:
      ensure_stats(cur, self.current_match_id, self.s_id, batted=True)
      ensure_stats(cur, self.current_match_id, self.b_id)

      cur.execute(
          """
          INSERT INTO delivery_history (
              match_id, innings, striker_id, non_striker_id, bowler_id,
              bat_runs, extra_type, extra_runs,
              is_wicket, wicket_type, out_player_id,
              prev_runs, prev_wickets, prev_legal, prev_striker_id, prev_non_striker_id
          ) VALUES (?, ?, ?, ?, ?, 0, 'normal', 0, 1, ?, ?, ?, ?, ?, ?, ?)
      """,
          (
              self.current_match_id,
              self.innings,
              self.s_id,
              self.ns_id,
              self.b_id,
              wicket_type,
              out_id,
              self.runs,
              self.wickets,
              self.legal_balls,
              self.s_id,
              self.ns_id,
          ),
      )

      cur.execute(
          "UPDATE player_match_stats SET balls_faced = balls_faced + 1 WHERE"
          " match_id = ? AND player_id = ?",
          (self.current_match_id, self.s_id),
      )
      cur.execute(
          "UPDATE player_match_stats SET out_status = ? WHERE match_id = ? AND"
          " player_id = ?",
          (f"Out ({wicket_type})", self.current_match_id, out_id),
      )
      cur.execute(
          "UPDATE player_match_stats SET legal_balls_bowled ="
          " legal_balls_bowled + 1, wickets_taken = wickets_taken + ? WHERE"
          " match_id = ? AND player_id = ?",
          (1 if credit_bowler else 0, self.current_match_id, self.b_id),
      )
      cur.execute(
          "UPDATE match_state SET wickets = ?, legal_balls = ? WHERE match_id = ?",
          (new_wickets, new_legal, self.current_match_id),
      )

    if self.innings_over(self.runs, new_wickets, new_legal):
      self.end_innings()
      return
    if not self.replace_batter(out_id):
      return
    if over_done:
      self.swap_ends()
      messagebox.showinfo("Over Complete", f"End of over {new_legal // 6}! Select the next bowler.")
      self.prompt_new_bowler()
    self.show_scoring_dashboard()

  def undo_last_ball(self):
    with connect() as cur:
      cur.execute(
          "SELECT id, innings, striker_id, non_striker_id, bowler_id, bat_runs,"
          " extra_type, extra_runs, is_wicket, wicket_type, out_player_id,"
          " prev_runs, prev_wickets, prev_legal, prev_striker_id, prev_non_striker_id"
          " FROM delivery_history WHERE match_id = ? ORDER BY id DESC LIMIT 1",
          (self.current_match_id,),
      )
      last = cur.fetchone()
      if not last:
        messagebox.showinfo("Undo", "No deliveries to undo in this match!")
        return

      (
          did, inn, s_id, ns_id, b_id, bat_runs, extra_type, extra_runs,
          is_wkt, w_type, out_pid, p_runs, p_wkts, p_legal, p_s, p_ns,
      ) = last

      is_legal = extra_type not in ("wide", "noball")
      conceded = bat_runs + (extra_runs if extra_type in ("wide", "noball") else 0)
      faced = 0 if extra_type == "wide" else 1

      if is_wkt:
        credit_bowler = w_type != "Run Out"
        cur.execute(
            "UPDATE player_match_stats SET balls_faced = balls_faced - 1 WHERE match_id = ? AND player_id = ?",
            (self.current_match_id, s_id),
        )
        cur.execute(
            "UPDATE player_match_stats SET out_status = 'Not Out' WHERE match_id = ? AND player_id = ?",
            (self.current_match_id, out_pid),
        )
        cur.execute(
            "UPDATE player_match_stats SET legal_balls_bowled = legal_balls_bowled - 1, wickets_taken = wickets_taken - ? WHERE match_id = ? AND player_id = ?",
            (1 if credit_bowler else 0, self.current_match_id, b_id),
        )
      else:
        cur.execute(
            "UPDATE player_match_stats SET runs = runs - ?, balls_faced = balls_faced - ?,"
            " ones = ones - ?, twos = twos - ?, threes = threes - ?, fours = fours - ?,"
            " sixes = sixes - ? WHERE match_id = ? AND player_id = ?",
            (
                bat_runs,
                faced,
                int(bat_runs == 1),
                int(bat_runs == 2),
                int(bat_runs == 3),
                int(bat_runs == 4),
                int(bat_runs == 6),
                self.current_match_id,
                s_id,
            ),
        )
        cur.execute(
            "UPDATE player_match_stats SET runs_conceded = runs_conceded - ?,"
            " legal_balls_bowled = legal_balls_bowled - ? WHERE match_id = ? AND player_id = ?",
            (conceded, 1 if is_legal else 0, self.current_match_id, b_id),
        )

      cur.execute(
          "UPDATE match_state SET runs = ?, wickets = ?, legal_balls = ?,"
          " striker_id = ?, non_striker_id = ?, bowler_id = ? WHERE match_id = ?",
          (p_runs, p_wkts, p_legal, p_s, p_ns, b_id, self.current_match_id),
      )
      cur.execute("DELETE FROM delivery_history WHERE id = ?", (did,))

    messagebox.showinfo("Undo Success", "Last delivery successfully undone!")
    self.show_scoring_dashboard()

  def swap_ends(self):
    with connect() as cur:
      cur.execute(
          "UPDATE match_state SET striker_id = non_striker_id, non_striker_id = striker_id WHERE match_id = ?",
          (self.current_match_id,),
      )

  def replace_batter(self, out_id):
    with connect() as cur:
      cur.execute(
          "SELECT id, name, COALESCE(role, 'All-Rounder') FROM players WHERE team_id = ? AND id NOT IN (?, ?)"
          " AND id NOT IN (SELECT player_id FROM player_match_stats WHERE"
          " match_id = ? AND out_status != 'Not Out') ORDER BY id",
          (self.bat_id, self.s_id, self.ns_id, self.current_match_id),
      )
      available = cur.fetchall()
    if not available:
      self.end_innings()
      return False

    formatted = [(p[0], f"{p[1]} [{p[2]}]") for p in available]
    new_id = self.ask_player("Select New Batter", "Select New Batter:", formatted)
    column = "striker_id" if out_id == self.s_id else "non_striker_id"
    with connect() as cur:
      ensure_stats(cur, self.current_match_id, new_id, batted=True)
      cur.execute(
          f"UPDATE match_state SET {column} = ? WHERE match_id = ?",
          (new_id, self.current_match_id),
      )
    return True

  def prompt_new_bowler(self):
    max_ov = max(1, math.ceil(self.total_overs / 5))

    with connect() as cur:
      cur.execute(
          """
          SELECT p.id, p.name, COALESCE(p.role, 'All-Rounder'),
                 COALESCE(s.legal_balls_bowled, 0),
                 COALESCE(s.runs_conceded, 0),
                 COALESCE(s.wickets_taken, 0)
          FROM players p
          LEFT JOIN player_match_stats s ON p.id = s.player_id AND s.match_id = ?
          WHERE p.team_id = ? AND p.id != ?
          ORDER BY
              CASE
                  WHEN p.role = 'Bowler' THEN 1
                  WHEN p.role = 'All-Rounder' THEN 2
                  WHEN p.role = 'Batsman' THEN 3
                  ELSE 4
              END, p.name
      """,
          (self.current_match_id, self.bowl_id, self.b_id),
      )
      candidates = cur.fetchall()

    eligible = []
    for pid, name, role, balls, runs, wkts in candidates:
      if balls < max_ov * 6:
        label = f"{name} [{role}] — {fmt_overs(balls)}/{max_ov} ov ({runs}r, {wkts}w)"
        eligible.append((pid, label))

    if not eligible:
      for pid, name, role, balls, runs, wkts in candidates:
        label = f"{name} [{role}] — {fmt_overs(balls)} ov (Quota unlocked)"
        eligible.append((pid, label))

    if not eligible:
      return

    subtitle = f"Max quota: {max_ov} overs. Bowlers & All-Rounders listed first."
    new_id = self.ask_player("Select Next Bowler", "Select Next Bowler:", eligible, subtitle=subtitle)
    with connect() as cur:
      ensure_stats(cur, self.current_match_id, new_id)
      cur.execute(
          "UPDATE match_state SET bowler_id = ? WHERE match_id = ?",
          (new_id, self.current_match_id),
      )

  # ---------- DLS CALCULATOR ----------
  def show_dls_dialog(self):
    top = self._modal("DLS Calculator", 350, 300)
    self.make_label(top, "🌧  DLS Target Calculator", 11, True, fg=self.COLOR_PRIMARY).pack(pady=(12, 6))

    frame = tk.Frame(top, bg=self.COLOR_BG)
    frame.pack(pady=4)

    self.make_label(frame, "Inn 1 Total Runs:").grid(row=0, column=0, padx=4, pady=4, sticky="w")
    inn1_runs_entry = self.make_styled_entry(frame, width=10)
    inn1_runs_entry.grid(row=0, column=1, padx=4, pady=4)

    self.make_label(frame, "Inn 1 Overs:").grid(row=1, column=0, padx=4, pady=4, sticky="w")
    inn1_ov_entry = self.make_styled_entry(frame, width=10)
    inn1_ov_entry.grid(row=1, column=1, padx=4, pady=4)

    self.make_label(frame, "Revised Inn 2 Overs:").grid(row=2, column=0, padx=4, pady=4, sticky="w")
    inn2_ov_entry = self.make_styled_entry(frame, width=10)
    inn2_ov_entry.grid(row=2, column=1, padx=4, pady=4)

    with connect() as cur:
      cur.execute("SELECT runs FROM match_state WHERE match_id = ?", (self.current_match_id,))
      st = cur.fetchone()
      if self.innings == 2 and self.target:
        inn1_runs_entry.insert(0, str(self.target - 1))
      elif st:
        inn1_runs_entry.insert(0, str(st[0]))
      inn1_ov_entry.insert(0, str(self.total_overs))
      inn2_ov_entry.insert(0, str(max(1, self.total_overs - 2)))

    res_lbl = self.make_label(top, "Enter overs and calculate revised target", 9, fg=self.COLOR_TEXT_MUTED)
    res_lbl.pack(pady=6)

    def calc():
      try:
        r1 = float(inn1_runs_entry.get().strip())
        o1 = float(inn1_ov_entry.get().strip())
        o2 = float(inn2_ov_entry.get().strip())
        if o2 >= o1:
          target = int(r1 + 1)
        else:
          res1 = 100.0 * (1.0 - math.exp(-0.035 * o1 * 6))
          res2 = 100.0 * (1.0 - math.exp(-0.035 * o2 * 6))
          target = max(1, math.floor(r1 * (res2 / res1)) + 1)
        res_lbl.config(
            text=f"🎯 Target: {target} runs in {int(o2)} ov",
            font=(FONT, 10, "bold"),
            fg=self.COLOR_PRIMARY,
        )
        if self.innings == 2 and messagebox.askyesno("Apply DLS", f"Update match target to {target} in {int(o2)} overs?"):
          with connect() as c:
            c.execute("UPDATE matches SET target = ?, total_overs = ? WHERE id = ?", (target, int(o2), self.current_match_id))
          top.destroy()
          self.show_scoring_dashboard()
      except ValueError:
        messagebox.showerror("Error", "Please enter valid numeric figures.")

    btn_row = tk.Frame(top, bg=self.COLOR_BG)
    btn_row.pack(pady=6)
    self.create_styled_button(btn_row, "Calculate", calc, self.COLOR_PRIMARY, width=12, pady=5).pack(side=tk.LEFT, padx=3)
    self.create_styled_button(btn_row, "Close", top.destroy, "#8c9a92", width=10, pady=5).pack(side=tk.LEFT, padx=3)

  # ---------- END INNINGS & MATCH ----------
  def end_innings(self):
    with connect() as cur:
      cur.execute(
          "SELECT innings, runs, wickets, batting_team_id, bowling_team_id FROM"
          " match_state WHERE match_id = ?",
          (self.current_match_id,),
      )
      inn, runs, wkts, bat_id, bowl_id = cur.fetchone()

      if inn == 1:
        target = runs + 1
        cur.execute(
            "UPDATE matches SET target = ? WHERE id = ?",
            (target, self.current_match_id),
        )
        cur.execute(
            "UPDATE match_state SET innings = 2, batting_team_id = ?,"
            " bowling_team_id = ?, runs = 0, wickets = 0, legal_balls = 0 WHERE"
            " match_id = ?",
            (bowl_id, bat_id, self.current_match_id),
        )
        result = None
      else:
        cur.execute("SELECT target FROM matches WHERE id = ?", (self.current_match_id,))
        target = cur.fetchone()[0] or 1
        cur.execute("SELECT name FROM teams WHERE id = ?", (bat_id,))
        chasing = cur.fetchone()[0]
        cur.execute("SELECT name FROM teams WHERE id = ?", (bowl_id,))
        defending = cur.fetchone()[0]
        if runs >= target:
          left = max_wickets_for(cur, bat_id) - wkts
          result = f"{chasing} won by {left} wicket{'s' if left != 1 else ''}"
        elif runs == target - 1:
          result = "Match Tied!"
        else:
          margin = target - 1 - runs
          result = f"{defending} won by {margin} run{'s' if margin != 1 else ''}"
        cur.execute(
            "UPDATE matches SET status = 'Completed', result = ? WHERE id = ?",
            (result, self.current_match_id),
        )

    if result is None:
      messagebox.showinfo("Innings Complete", f"Innings 1 Finished!\nTarget for Innings 2 is: {target}")
      self.show_scoring_dashboard()
    else:
      messagebox.showinfo("Match Completed", f"Match Over!\n{result}")
      self.show_scorecard()

  # ================= SCREEN 5: MOBILE SCORECARD =================
  def show_scorecard(self):
    with connect() as cur:
      cur.execute(
          "SELECT venue, total_overs, target, result, status, format, team1_name, team2_name FROM matches WHERE id = ?",
          (self.current_match_id,),
      )
      row = cur.fetchone()
      if row is None:
        messagebox.showerror("Error", "Match not found.")
        self.show_main_menu()
        return

      venue, overs, target, result, status, m_format, t1_n, t2_n = row
      cur.execute(
          "SELECT p.id, p.name, t.name, s.runs, s.balls_faced, s.fours, s.sixes,"
          " s.out_status, COALESCE(p.role, 'All-Rounder') FROM player_match_stats s JOIN players p ON"
          " s.player_id = p.id JOIN teams t ON p.team_id = t.id WHERE"
          " s.match_id = ? AND s.batted = 1 ORDER BY s.id",
          (self.current_match_id,),
      )
      batting = cur.fetchall()
      cur.execute(
          "SELECT p.id, p.name, t.name, s.legal_balls_bowled, s.runs_conceded,"
          " s.wickets_taken, COALESCE(p.role, 'All-Rounder') FROM player_match_stats s JOIN players p ON"
          " s.player_id = p.id JOIN teams t ON p.team_id = t.id WHERE"
          " s.match_id = ? AND (s.legal_balls_bowled > 0 OR s.runs_conceded"
          " > 0 OR s.wickets_taken > 0) ORDER BY s.id",
          (self.current_match_id,),
      )
      bowling = cur.fetchall()
      cur.execute(
          "SELECT innings, runs, wickets, legal_balls FROM match_state WHERE match_id = ?",
          (self.current_match_id,),
      )
      live = cur.fetchone()

    screen = self.make_scrollable_screen("Match Scorecard")

    info = f"{t1_n or 'Team 1'} vs {t2_n or 'Team 2'}\n{venue} • {m_format} ({overs} Ov)"
    if target:
      info += f" • Target: {target}"
    if status == "Completed" and result:
      info += f"\n🏆 {result}"
    elif live:
      info += f"\nLive — Inn {live[0]}: {live[1]}/{live[2]} ({fmt_overs(live[3])} ov)"
    self.make_label(screen, info, 9, fg=self.COLOR_TEXT_MUTED, justify=tk.CENTER).pack(pady=6)

    card = self.make_card(screen, padx=12, pady=8)
    card.pack(fill=tk.X, padx=16, pady=4)

    self.make_label(card, "🏏  Batting Summary", 10, True, fg=self.COLOR_PRIMARY).pack(anchor="w", pady=(0, 4))
    frame, tree = self.make_table(
        card, ("Player", "R", "B", "4s", "6s", "SR", "Status"), (90, 35, 35, 30, 30, 45, 95), 5
    )
    frame.pack(fill=tk.X)
    for _, name, team, r, b, f4, s6, out, role in batting:
      sr = f"{r * 100 / b:.1f}" if b else "-"
      tree.insert("", tk.END, values=(name, r, b, f4, s6, sr, out))

    self.make_label(card, "🎯  Bowling Summary", 10, True, fg=self.COLOR_PRIMARY).pack(anchor="w", pady=(8, 4))
    frame2, tree2 = self.make_table(
        card, ("Bowler", "Ov", "Runs", "Wkts", "Econ"), (110, 50, 50, 50, 55), 4
    )
    frame2.pack(fill=tk.X)
    for _, name, team, balls, conceded, wk, role in bowling:
      econ = f"{conceded * 6 / balls:.2f}" if balls else "-"
      tree2.insert("", tk.END, values=(name, fmt_overs(balls), conceded, wk, econ))

    action_row = tk.Frame(screen, bg=self.COLOR_BG)
    action_row.pack(fill=tk.X, padx=16, pady=8)

    self.create_styled_button(
        action_row, "📤 Share Scorecard", self.export_shareable_scorecard, "#506b5c", width=14, pady=6
    ).pack(side=tk.LEFT, padx=(0, 4), expand=True, fill=tk.X)

    if status == "Ongoing":
      self.create_styled_button(
          action_row, "Back to Match", self.show_scoring_dashboard, self.COLOR_ACCENT, width=14, pady=6
      ).pack(side=tk.LEFT, expand=True, fill=tk.X)
    else:
      self.create_styled_button(
          action_row, "Main Menu", self.show_main_menu, "#8c9a92", width=14, pady=6
      ).pack(side=tk.LEFT, expand=True, fill=tk.X)

  def export_shareable_scorecard(self):
    with connect() as cur:
      cur.execute(
          "SELECT venue, total_overs, target, result, status, format, team1_name, team2_name FROM matches WHERE id = ?",
          (self.current_match_id,),
      )
      venue, overs, target, result, status, m_fmt, t1_n, t2_n = cur.fetchone()
      cur.execute(
          "SELECT p.name, t.name, s.runs, s.balls_faced, s.fours, s.sixes, s.out_status FROM player_match_stats s JOIN players p ON s.player_id = p.id JOIN teams t ON p.team_id = t.id WHERE s.match_id = ? AND s.batted = 1",
          (self.current_match_id,),
      )
      batting = cur.fetchall()
      cur.execute(
          "SELECT p.name, t.name, s.legal_balls_bowled, s.runs_conceded, s.wickets_taken FROM player_match_stats s JOIN players p ON s.player_id = p.id JOIN teams t ON p.team_id = t.id WHERE s.match_id = ? AND (s.legal_balls_bowled > 0 OR s.runs_conceded > 0)",
          (self.current_match_id,),
      )
      bowling = cur.fetchall()

    lines = [
        "===========================================================",
        "                 CRICKET MATCH SCORECARD                   ",
        "===========================================================",
        f"Match:  {t1_n or 'Team 1'} vs {t2_n or 'Team 2'}",
        f"Venue:  {venue} | Format: {m_fmt} ({overs} Ov)",
        f"Status: {status} | Target: {target or 'N/A'}",
        f"Result: {result or 'In Progress'}",
        "-----------------------------------------------------------",
        "BATTING SUMMARY:",
        f"{'Player':<18} {'R':<5} {'B':<5} {'4s':<4} {'6s':<4} {'Status'}",
        "-" * 59,
    ]
    for n, t, r, b, f4, s6, out in batting:
      lines.append(f"{n:<18} {r:<5} {b:<5} {f4:<4} {s6:<4} {out}")

    lines.extend([
        "-----------------------------------------------------------",
        "BOWLING SUMMARY:",
        f"{'Bowler':<18} {'Overs':<8} {'Runs':<6} {'Wickets':<8}",
        "-" * 59,
    ])
    for n, t, balls, runs, wk in bowling:
      lines.append(f"{n:<18} {fmt_overs(balls):<8} {runs:<6} {wk:<8}")
    lines.append("===========================================================")

    report_text = "\n".join(lines)
    try:
      self.root.clipboard_clear()
      self.root.clipboard_append(report_text)
    except Exception:
      pass

    file_path = filedialog.asksaveasfilename(
        defaultextension=".txt",
        filetypes=[("Text files", "*.txt"), ("All Files", "*.*")],
        initialfile="Match_Scorecard.txt",
    )
    if file_path:
      with open(file_path, "w", encoding="utf-8") as f:
        f.write(report_text)
      messagebox.showinfo("Success", f"Scorecard saved to:\n{file_path}\n\n(Also copied to clipboard!)")

  # ================= SCREEN 6: MOBILE CAREER STATS =================
  def show_career_stats(self):
    screen = self.make_scrollable_screen("Career Statistics")

    card = self.make_card(screen, padx=12, pady=10)
    card.pack(fill=tk.X, padx=16, pady=6)

    with connect() as cur:
      cur.execute("SELECT id, name FROM teams ORDER BY id")
      teams = [("All Teams", 0)] + cur.fetchall()

    row = tk.Frame(card, bg=self.COLOR_CARD)
    row.pack(fill=tk.X, pady=(0, 6))
    self.make_label(row, "Filter Team:").pack(side=tk.LEFT, padx=(0, 4))
    team_cb = ttk.Combobox(row, values=[t[1] for t in teams], state="readonly", font=(FONT, 9), width=18)
    team_cb.pack(side=tk.LEFT)
    team_cb.current(0)

    cols = ("Player", "Role", "M", "Runs", "HS", "Avg", "SR", "Wkts", "Econ")
    widths = (85, 75, 30, 40, 35, 40, 40, 35, 40)
    frame, tree = self.make_table(card, cols, widths, 12)
    frame.pack(fill=tk.X)

    def load_table(event=None):
      tree.delete(*tree.get_children())
      sel_idx = team_cb.current()
      filter_tid = teams[sel_idx][0] if sel_idx > 0 else None

      query = """
          SELECT p.name, COALESCE(p.role, 'All-Rounder'), t.name,
                 COUNT(DISTINCT s.match_id) as matches,
                 SUM(s.batted) as innings,
                 SUM(s.runs) as total_runs,
                 MAX(s.runs) as high_score,
                 SUM(CASE WHEN s.out_status != 'Not Out' AND s.batted = 1 THEN 1 ELSE 0 END) as outs,
                 SUM(s.balls_faced) as balls_faced,
                 SUM(s.fours) as fours,
                 SUM(s.sixes) as sixes,
                 SUM(s.wickets_taken) as wickets,
                 SUM(s.runs_conceded) as runs_conceded,
                 SUM(s.legal_balls_bowled) as balls_bowled
          FROM players p
          JOIN teams t ON p.team_id = t.id
          LEFT JOIN player_match_stats s ON p.id = s.player_id
      """
      params = []
      if filter_tid is not None:
        query += " WHERE t.id = ?"
        params.append(filter_tid)
      query += " GROUP BY p.id, p.name, p.role, t.name ORDER BY total_runs DESC"

      with connect() as cur:
        cur.execute(query, params)
        rows = cur.fetchall()

      for r in rows:
        (name, role, tname, m, inn, runs, hs, outs, b_faced, f4, s6, wk, conceded, balls_b) = r
        m = m or 0
        runs = runs or 0
        hs = hs or 0
        outs = outs or 0
        b_faced = b_faced or 0
        wk = wk or 0
        conceded = conceded or 0
        balls_b = balls_b or 0

        avg = f"{runs / outs:.1f}" if outs > 0 else (str(runs) if runs > 0 else "-")
        sr = f"{runs * 100 / b_faced:.1f}" if b_faced > 0 else "-"
        econ = f"{conceded * 6 / balls_b:.2f}" if balls_b > 0 else "-"

        tree.insert("", tk.END, values=(name, role, m, runs, hs, avg, sr, wk, econ))

    team_cb.bind("<<ComboboxSelected>>", load_table)
    load_table()

  # ================= SCREEN 7: MOBILE CLOUD SYNC =================
  def show_cloud_sync_dialog(self):
    screen = self.make_scrollable_screen("Cloud Backup & Sync")

    card = self.make_card(screen, padx=16, pady=12)
    card.pack(fill=tk.X, padx=20, pady=10)

    self.make_label(card, "☁  JSON Cloud Sync", 11, True, fg=self.COLOR_PRIMARY).pack(anchor="w", pady=(0, 2))
    self.make_label(
        card, "Export all match data into a clean JSON cloud file or restore anytime.", 9, fg=self.COLOR_TEXT_MUTED
    ).pack(anchor="w", pady=(0, 10))

    def export_backup():
      tables = ["teams", "players", "matches", "match_state", "player_match_stats", "delivery_history"]
      dump = {}
      with connect() as cur:
        for t in tables:
          cur.execute(f"SELECT * FROM {t}")
          cols = [c[0] for c in cur.description]
          dump[t] = [dict(zip(cols, row)) for row in cur.fetchall()]

      path = filedialog.asksaveasfilename(
          defaultextension=".json",
          filetypes=[("JSON files", "*.json")],
          initialfile="cricket_cloud_backup.json",
      )
      if path:
        with open(path, "w", encoding="utf-8") as f:
          json.dump(dump, f, indent=2)
        messagebox.showinfo("Cloud Backup", f"Cloud backup exported successfully to:\n{path}")

    def import_backup():
      path = filedialog.askopenfilename(filetypes=[("JSON files", "*.json")])
      if not path:
        return
      with open(path, "r", encoding="utf-8") as f:
        data = json.load(f)

      if not messagebox.askyesno("Confirm Restore", "Restoring cloud backup will overwrite current database. Continue?"):
        return

      with connect() as cur:
        for t in ["delivery_history", "player_match_stats", "match_state", "matches", "players", "teams"]:
          cur.execute(f"DELETE FROM {t}")
        for t, rows in data.items():
          if not rows:
            continue
          cols = list(rows[0].keys())
          placeholders = ", ".join(["?"] * len(cols))
          sql = f"INSERT INTO {t} ({', '.join(cols)}) VALUES ({placeholders})"
          for r in rows:
            cur.execute(sql, [r[c] for c in cols])
      messagebox.showinfo("Cloud Sync", "Database synchronized successfully!")
      self.show_main_menu()

    self.create_styled_button(card, "📤 Export Backup (JSON)", export_backup, self.COLOR_PRIMARY, width=28, pady=8).pack(
        pady=4
    )
    self.create_styled_button(card, "📥 Import / Restore", import_backup, self.COLOR_ACCENT, width=28, pady=8).pack(
        pady=4
    )

  # ================= SCREEN 8: MOBILE PAST MATCHES =================
  def show_past_matches(self):
    screen = self.make_scrollable_screen("Match History")

    card = self.make_card(screen, padx=14, pady=10)
    card.pack(fill=tk.X, padx=16, pady=8)

    with connect() as cur:
      cur.execute("""
          SELECT m.id,
                 COALESCE(m.team1_name, t1.name, 'Team 1'),
                 COALESCE(m.team2_name, t2.name, 'Team 2'),
                 m.venue,
                 m.status,
                 m.result,
                 COALESCE(m.format, 'Custom Overs')
          FROM matches m
          LEFT JOIN teams t1 ON m.team1_id = t1.id
          LEFT JOIN teams t2 ON m.team2_id = t2.id
          ORDER BY m.id DESC
      """)
      self.past_matches_data = cur.fetchall()

    match_frame = tk.Frame(card, bg=self.COLOR_CARD)
    match_frame.pack(fill=tk.X, pady=4)
    match_frame.grid_rowconfigure(0, weight=1)
    match_frame.grid_columnconfigure(0, weight=1)

    m_sb = AutoScrollbar(match_frame, orient="vertical")
    match_listbox = tk.Listbox(
        match_frame,
        font=(FONT, 9),
        height=8,
        bg=self.COLOR_INPUT_BG,
        fg=self.COLOR_TEXT_MAIN,
        bd=1,
        relief="solid",
        highlightbackground=self.COLOR_BORDER,
        highlightthickness=1,
        activestyle="none",
        exportselection=False,
        selectbackground="#d8e2dc",
        selectforeground=self.COLOR_TEXT_MAIN,
        yscrollcommand=m_sb.set,
    )
    m_sb.config(command=match_listbox.yview)
    match_listbox.grid(row=0, column=0, sticky="nsew")
    m_sb.grid(row=0, column=1, sticky="ns")

    total = len(self.past_matches_data)
    for i, m in enumerate(self.past_matches_data):
      outcome = m[5] if m[4] == "Completed" and m[5] else m[4]
      match_listbox.insert(tk.END, f" #{total - i} [{m[6]}] {m[1]} vs {m[2]}\n    • {outcome}")
    if not self.past_matches_data:
      match_listbox.insert(tk.END, "  No matches yet - start one from the main menu!")

    def selected():
      sel = match_listbox.curselection()
      if not sel or not self.past_matches_data:
        messagebox.showerror("Error", "Please select a match!")
        return None
      return self.past_matches_data[sel[0]]

    def reopen_match():
      m = selected()
      if m:
        self.current_match_id = m[0]
        if m[4] == "Ongoing":
          self.show_scoring_dashboard()
        else:
          self.show_scorecard()

    def delete_match():
      m = selected()
      if m and messagebox.askyesno("Confirm Delete", f"Delete match '{m[1]} vs {m[2]}'?"):
        with connect() as cur:
          delete_match_rows(cur, m[0])
        self.show_past_matches()

    btn_row = tk.Frame(card, bg=self.COLOR_CARD)
    btn_row.pack(fill=tk.X, pady=(8, 2))
    self.create_styled_button(btn_row, "Open Match", reopen_match, self.COLOR_ACCENT, width=13, pady=6).pack(
        side=tk.LEFT, padx=(0, 4), expand=True, fill=tk.X
    )
    self.create_styled_button(btn_row, "Delete", delete_match, self.COLOR_DANGER, width=13, pady=6).pack(
        side=tk.LEFT, expand=True, fill=tk.X
    )


# --- RUNNER ---
if __name__ == "__main__":
  init_db()
  seed_test_data()

  print(f"Starting Cricket App (Native Crisp 420x750). DB: {DB_NAME}")
  root = tk.Tk()
  app = CricketApp(root)
  root.mainloop()