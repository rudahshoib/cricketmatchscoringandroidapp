CRICKET MATCH SCORING APP - USER GUIDE
ABOUT
A cricket scoring app made with Python (Tkinter and SQLite). It looks like a phone app but
runs on a laptop or computer.
REQUIREMENTS
1. A laptop or desktop computer
2. Windows operating system
3. Python 3.8 or newer (download from python.org)
4. Nothing else to install. Tkinter and SQLite come with Python.
To check if Python is installed, open Command Prompt and type: python --version
HOW TO RUN
1. Put finalapp.py in a folder.
2. Open that folder, click on the address bar, type cmd and press Enter.
3. Type: python finalapp.py and press Enter.
4. The app window will open.
A file called cricket_app.db is created automatically in the same folder to save the data. Two
sample teams (Eagles and Falcons) are added so you can test right away.
HOW TO USE
1. Manage Teams & Roster: add a team, then add players with a role (Batsman, Bowler,
All-Rounder or Wicketkeeper). Maximum 11 players per team.
2. Start Live Match: pick two teams, enter the venue and choose the format (T20, ODI, T10,
etc.). The overs fill automatically.
3. Toss: choose who won the toss and whether they bat or bowl. Then select the opening batters
and the bowler.
4. Scoring: tap runs (0 to 6), extras (Wide, No Ball, Bye, Leg Bye) or a wicket type (Bowled,
Caught, LBW, Stumped, Run Out). Use Undo Ball if you make a mistake. The app asks for the
next batter and bowler automatically.
5. Second innings: after the first innings the target is shown and you select the new lineup. The
result is shown at the end.
6. Scorecard: view it anytime and save it as a .txt file with Share Scorecard.
OTHER MENU OPTIONS
- Player Career Statistics: runs, average, strike rate, wickets and economy of every player.
- Past & Saved Matches: open or delete old matches.
- Cloud Backup & Sync: saves all data to a JSON file, or restores it from one (restoring replaces
current data).
QUICK TEST
Start a match between Eagles and Falcons using the Super Over (1 Over) format. Each innings is
only 6 balls, so it finishes fast.
COMMON PROBLEMS
- "python" not recognised: install Python from python.org and tick "Add Python to PATH".
- To reset all data: close the app and delete cricket_app.db.
