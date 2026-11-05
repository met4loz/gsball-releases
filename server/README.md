# Running a GSBall server on a VPS

This guide sets up a GSBall dedicated server on a rented Linux server (a VPS),
with one installer script. It is also shipped as `GUIDE.md` beside the installed
server.

## What you need

- A VPS running **Ubuntu 22.04 or 24.04, or Debian 12**, on a **64-bit x86
  (amd64)** processor. ARM servers do not work: Steam's server library has no
  ARM version.
- **1 GB of memory** is enough for one server; 2 GB leaves room to spare.
- A way to run commands as root on it (`sudo`, usually over SSH).
- **Your Steam ID**, to make yourself the server's owner (see
  [Finding your Steam ID](#finding-your-steam-id)).

The server does not need a Steam account. It signs in to Steam anonymously, as
game servers do, which it needs to check that every player really is who their
Steam account says.

## Installing

Connect to the VPS and run:

```bash
curl -fsSLO https://github.com/met4loz/gsball-releases/releases/latest/download/install-gsball-server.sh
```

```bash
sudo bash install-gsball-server.sh
```

You can read the script first (`less install-gsball-server.sh`), or see every
command it would run without running any: `bash install-gsball-server.sh --dry-run`.

The installer first **shows its plan and asks before changing anything**:

1. Installs a few packages with apt: `curl`, `ca-certificates`, `unzip`, `tmux`
   and `lib32gcc-s1` (SteamCMD is a 32-bit program). No Java: the server is a
   native program.
2. Creates a user called `gsball` for the server to run as. It has no password
   and cannot log in, and the server runs with no way to gain more rights.
3. Downloads SteamCMD from Valve into `/opt/gsball/steamcmd`. The server needs
   Steam's Linux library from it. Using SteamCMD means accepting the
   [Steam Subscriber Agreement](https://store.steampowered.com/subscriber_agreement/).
4. Installs the GSBall server into `/opt/gsball/server`. The server comes inside
   the installer - it is the only file you download - and the installer checks
   it arrived complete before installing it.
5. Asks the **setup questions**: server name, how many players, a password,
   your Steam ID, who else can be an admin, the ports, whether to list it, and
   the starting map. Press Enter to keep a suggested value.
6. Installs the `gsball-server` service. The server starts right away, and again
   whenever the VPS reboots.
7. Installs the `gsball` command. It, and the copy of the installer it uses for
   `gsball update` and `gsball uninstall`, belong to root (`/usr/local/bin`,
   `/usr/local/lib/gsball`): nothing the server itself could change is ever
   run as root.

It then asks, separately, whether to open the ports in the `ufw` firewall.

## Ports

Players need to reach two **UDP** ports on the VPS:

| Port | What for |
|---|---|
| **54555** (`port`) | players connect to it |
| **27015** (`query-port`) | the room browser asks the server for its name and player count |

The installer can open them in `ufw`. **Many VPS providers have their own
firewall in their web panel** (often called a *security group* or *firewall
rules*). If yours does, allow both UDP ports there as well, or nobody can reach
the server.

## Managing the server

| Command | What it does |
|---|---|
| `gsball help` | lists every command |
| `gsball status` | whether the server is running, and its latest log lines |
| `gsball start` / `gsball stop` / `gsball restart` | start, stop (players are told at once), restart |
| `gsball console` | opens the server console, to type commands into it |
| `gsball send <command>` | runs one console command, e.g. `gsball send players` |
| `gsball logs` | follows the log (Ctrl+C stops following) |
| `gsball config` | edits the settings file |
| `gsball setup` | asks the setup questions again, with your current answers filled in |
| `gsball update` | installs the newest server version, keeping settings and bans |
| `gsball uninstall` | removes the server (it asks first) |
| `gsball guide` | shows this guide |

### The console

`gsball console` opens the server's console. Type `help` there for its commands:

| Command | What it does |
|---|---|
| `status` | the room, its players and its match |
| `players` | everyone connected, with their ids |
| `say <text>` | a message to everyone in the room |
| `start` / `end` / `pause` | match controls |
| `map [name]` | list the maps, or pick the next match's map |
| `lock` / `unlock` | lock or unlock the teams |
| `kick <player> [reason]` | remove a player (`<player>` is an id from `players`, or a name) |
| `ban <player> [reason]` | remove and ban a player |
| `ban steam:<id> [reason]` | ban a Steam account, even if it is not on the server |
| `bans` / `unban <banId>` | list bans, lift one |
| `admin <player>` / `deadmin <player>` | give or take away admin |
| `stop` | shut the server down |

**To leave the console with the server still running, press Ctrl+B, then D.**
Typing `stop`, or pressing Ctrl+C, shuts the server down.

For a single command, `gsball send` is quicker than opening the console:

```bash
gsball send players
```

```bash
gsball send kick 3 spamming
```

## Who is in charge

- **Power admins** are the server's owners: the Steam IDs you give in the setup
  questions (`power-admins` in the settings). They are admins whenever they
  join, **only they can ban players or lift bans**, and nobody in the room can
  kick, ban, mute or demote them.
- **Regular admins** can kick players and run matches, but cannot ban.
- **The console** can do everything.

Bans are on Steam accounts and are kept when the server restarts.

## Finding your Steam ID

Your Steam ID is a 17-digit number starting with `7656`.

1. In Steam, open your profile, then **right-click the page → Copy Page URL**
   (or open your profile in a web browser).
2. If the address looks like `steamcommunity.com/profiles/76561198…`, the number
   is your Steam ID.
3. If it looks like `steamcommunity.com/id/yourname`, paste that address into a
   site such as steamid.io, which shows the number.

Several owners: separate the IDs with commas.

## Changing settings

```bash
gsball config
```

```bash
gsball restart
```

`gsball config` opens the settings file in an editor (save and exit in `nano`:
Ctrl+O, Enter, Ctrl+X). Every setting is explained inside the file.

Or run `gsball setup`, which asks the setup questions again without opening the
file:

- Each question shows the **current** value. Press Enter to keep it.
- To remove the password or the power admins, type `-`.
- It can also ask the match rules: auto-start, time and score limits, golden
  goal and the idle kick.
- Only the settings you answer differently are changed. Everything else in the
  file, including your own edits, stays as it is.
- If the server is running, it offers to restart it. A restart disconnects
  everyone playing.

To change settings **without a restart**:

- `gsball send reload` reads the file again and applies whatever changed. Only
  the ports and the file locations wait for a restart, and it says which.
- `gsball send set score-limit 3` changes one setting until the next restart,
  without touching the file. `gsball send set` lists every setting it can
  change, with its value.

## Updating

```bash
gsball update
```

It downloads the newest installer, shows what it will do and asks first. It stops
the server (disconnecting everyone), installs the newest version, keeps your
settings, bans and logs, and starts it again. To update from an installer you
copied to the VPS yourself: `gsball update install-gsball-server.sh`. **Every player must be on the same version as the server**, so
update it whenever a new game version is released.

## Troubleshooting

Start with `gsball status`: its last log lines usually say what is wrong.

**"Steam could not start"**
The server could not sign in to Steam or load Steam's library.
- Check that the VPS can reach the internet.
- Check that `/opt/gsball/.steam/sdk64/steamclient.so` exists. If it does not,
  run the installer again, which repeats the SteamCMD step.

**"The server config has … problems"**
A setting has a value the server does not accept, and the message names it.
Fix it with `gsball config` (or `gsball setup`), then run `gsball start`.

**"could not listen on UDP …"**
Something else is using that port, often a second server. Pick another `port`
(and `query-port`) with `gsball config`.

**Players cannot find the server in the room browser**
- Both UDP ports must be open: `ufw` on the VPS, **and** your provider's firewall.
- `listed` must be `true`.
- Wait a minute after the server starts.
- Players must be on the same game version as the server.

**Players cannot join**
- Players need Steam running: the server only lets in players whose Steam
  account it can verify.
- The game port (UDP 54555) must be open.

**The server keeps restarting**
It crashed. `gsball logs` shows why each time.

## Removing it

```bash
gsball uninstall
```

It asks before stopping the server, and asks again, separately, before deleting
`/opt/gsball` with your settings and bans. The other packages it installed stay
installed.

## Every setting

Every key is optional. A value that does not parse stops the server with a list
of every problem; a key it does not recognise is reported as a warning, since
that is almost always a typo.

| Key | Default | Meaning |
|---|---|---|
| `room-name` | `GSBall Server` | Shown to players in the room. At most 32 characters. |
| `max-players` | `10` | 2 to 20. |
| `password` | *(blank)* | Blank means anyone may join. At most 24 characters. |
| `port` | `54555` | The UDP port players connect to. |
| `bind-address` | *(blank)* | Blank listens on every network interface. `127.0.0.1` (or `localhost`) means only this machine can connect, whatever the firewall or router allows - use it for testing. `-PserverBind=` / `--bind` override it. |
| `query-port` | `27015` | The UDP port Steam answers server-list queries on. Must differ from `port`; 27015-27020 to be found on a LAN. |
| `listed` | `true` | List the server in Steam's server list and the room browser. Never listed when bound to 127.0.0.1. An unlisted server opens no query port. |
| `admin-policy` | `none` | `none`, `power-admins-only-grant` or `first-to-join` - see Who is in charge. |
| `power-admins` | *(blank)* | Steam account ids (17 digits), comma-separated. |
| `auto-start` | `false` under `first-to-join`, `true` otherwise | Start matches on their own. |
| `auto-start-delay-seconds` | `10` | How long both teams must be ready before a match starts. |
| `auto-start-min-per-team` | `1` | Players each team needs before the countdown begins. |
| `map` | `Normal` | `Small`, `Normal`, `Big A`, `Big B`, `Big - 3DEF`, `Huge`, `Last Man Standing` or `Practice`. |
| `time-limit-minutes` | `5` | `0` for no limit, otherwise 1 to 20. |
| `score-limit` | `5` | `0` for no limit, otherwise 1 to 20. |
| `golden-goal` | `false` | |
| `first-kickoff` | `red` | `red`, `blue` or `random`. |
| `mercy-goals` | `0` | `0` for off, otherwise 2 to 20: the match ends once a team leads by this many goals. |
| `mutators` | `off` | Casual rooms: `off`, or `rule:step` pairs such as `ball-size:big, kick-power:cannon`. Rules: `ball-size`, `ball-weight`, `kick-power`, `player-speed`, `ice-rink`, `slippery-ball`; steps are the names in the game's Match Settings (`normal`, `small`, `tiny`, `big`, `huge`, ...). A rule or step that does not exist refuses the whole file, like every other bad value. Players are told in chat when a match starts with mutators on. |
| `kick-idle-players` | `false` | Kick after `idle-kick-minutes` with no activity. Admins are never kicked. |
| `room-ball` | empty | The ball everyone in the room sees, as an item id such as `ball.default`. Empty or `own`: each player sees their own. |
| `room-pitch` | empty | The pitch theme everyone sees: `pitch.floodlit`, `pitch.sunbaked`, `pitch.court`. Empty or `own`: each player's own. A player can keep their own with a setting; an id their build does not know draws as the default. |
| `persist-bans` | `true` | Save bans on verified Steam accounts across restarts. |
| `ban-file` | `gsball-bans.txt` | Where, relative to the server's working directory. |
| `reset-when-empty` | `true` | Put this file's settings back when the room empties. |
| `console` | `true` | Read operator commands from standard input. |
| `admins` | *(blank)* | Steam account ids made an admin whenever they join, under every admin policy. Unlike power admins, the room can kick and demote them (until they next join). |
| `allowed-players` | *(blank)* | Only these Steam account ids may join. Power admins and `admins` always may. Blank lets anyone in. |
| `max-per-team` | `0` | Players a team may hold, 1 to 10. `0` means no cap. Nobody in the room can go past it, admins included; the console can. Must not be below `auto-start-min-per-team`. |
| `map-rotation` | *(blank)* | Maps played in turn, such as `Normal, Big A, Huge`. The next one is picked after each match that finishes (score, time, golden goal, mercy); an admin's End does not rotate. Blank keeps the map. Practice cannot be in it: its match never ends. |
| `welcome-message` | *(blank)* | Said privately to every player as they join. At most 240 characters. |
| `persist-mutes` | `true` | Keep server mutes, each on a Steam account, across rejoins and restarts, in `mute-file`. |
| `mute-file` | `gsball-mutes.txt` | Where, relative to the server's working directory. Must not be the ban file. |
| `pause-seconds` | `60` | A paused match resumes on its own after this many seconds, 10 to 600. |
| `idle-badge-minutes` | `5` | Minutes with no activity before a player is marked idle, 1 to 120. |
| `idle-kick-minutes` | `10` | Minutes before an idle player is kicked (with `kick-idle-players`). Must be more than `idle-badge-minutes`. The game's Room Settings caption still says 10 minutes. |
| `waiting-list` | `true` | When the room is full, newcomers wait in line for a seat instead of being turned away. |
| `waiting-list-size` | `10` | How many may wait, 1 to 20. |
| `chat-filter` | `true` | Block card and phone numbers and mask swearing in chat. |
| `auto-admin` | `0` | The team size, 1 to 5, for a room the server runs itself (see A room that runs itself). `0` is off. `max-players` must be at least twice it. Only changes with a restart. |
| `auto-admin-max-wins` | `0` | Matches the winners may win in a row before everybody goes back in line. `0` for no limit. |

## A room that runs itself (auto-admin)

With `auto-admin` set to a team size (`auto-admin=3` for 3v3), the server runs the room with
nobody admin-ing it.

- **Growing.** Two players start a 1v1 by themselves. Below the target, teams are filled from the
  front of the line with no captains. More players in a match's first minute restart it bigger;
  after that, the next match is bigger.
- **The line** is the order spectators become captain, longest waiting first. Joining puts you at
  the back. Everybody waiting sees a tab under the scoreboard with their place.
- **The draft.** At the target size two random captains take turns picking from everybody waiting,
  15 seconds a pick. A timeout picks the first in line, except that a captain who has picked
  nobody at all is removed for being away, and the next in line takes over.
- **After a match** the winners stay and the losers go to the back of the line. The first in line
  captains a new team and picks it alone. Golden goal is always on, so no match ends level.
  `auto-admin-max-wins` sends everybody back in line after that many wins in a row.
- **A leave during a full match** pauses it, and the team that lost the player votes in a
  spectator: 30 seconds for each place, and for each team in the order their players left, so a
  later team sees who an earlier one took. A majority takes a player at once; a place nobody
  chose in time gets a random spectator. A replacement keeps their place in line if their team
  loses. A team with nobody left ends the match and the other team stays. With too few spectators
  the match restarts smaller, and the newest player of the bigger team goes to the front of the
  line.
- **Leaving** loses your place: there is no saved spot. A crash reconnect that replaces the old
  connection before it times out is the same session and keeps the place.
- **Admins.** Nobody in the room can move teams, start a match, swap, reset or lock the teams.
  Power admins can still pause, end a match and change the match settings. The console refuses
  `start`, `move`, `swap` and `reset-teams` too, since a team changed behind the server's back
  leaves its draft or vote wrong; `end` still works. Ended by an admin or the console, a match has
  no result and everybody is drafted again.
- On Last Man Standing everybody is seated and a match starts whenever there are enough players;
  newcomers wait for the next one.
- On Practice everybody plays at once, and a newcomer gets a box straight away while it runs: a
  Practice match never ends, so nobody could wait in line for it.
- The config refuses a `max-per-team` below the `auto-admin` size (its teams could never be
  filled), and a `map-rotation` naming Practice (the rotation moves on when a match ends, and
  Practice never does).

## Every console command

Type commands into the server's terminal. A player can be named by the id
`players` shows, or by an exact one-word name.

| Command | Does |
|---|---|
| `help` | Lists the commands. |
| `status` | Room name, players, admin policy, map, and the match and its score. |
| `players` | Everyone connected, with ids, teams, admins, power admins, pings, and `steam:<id>` for verified players. |
| `say <text>` | A message to the whole room. |
| `start` / `end` / `pause` | Match controls. `start` restarts a match that is already running. |
| `map [name]` | Lists the maps, or picks the next match's map. |
| `look [ball\|own] [pitch\|own]` | Reports what everyone sees, or sets it: `look ball.default pitch.court`, `look own own`. Works mid-match; the room is told. |
| `lock` / `unlock` | Locks or unlocks the teams. |
| `kick <player> [reason]` | Removes a player - a power admin too. They are told the reason. |
| `ban <player> [reason]` | Removes and bans a player (not a power admin). |
| `ban steam:<id> [reason]` | Bans a Steam account, whether or not it is here. |
| `bans` / `unban <banId>` | The ban list, marking which bans are saved, and lifting one. |
| `admin <player>` / `deadmin <player>` | Grants or removes regular admin, for this session. To keep someone an admin, list them in `admins`. |
| `move <player> red\|blue\|spec` | Puts a player on a team. Goes past `max-per-team`. |
| `swap` / `reset-teams` | Swaps Red and Blue, or sends everyone to Spectators, like the admin buttons. |
| `mute <player>` / `unmute <player>` | Mutes or unmutes a player in chat. On a verified account the mute stays on the account, so leaving does not lift it. |
| `mute steam:<id>` / `unmute steam:<id>` | The same for an account that is not here. |
| `mutes` | Every muted account. |
| `unban all` | Lifts every ban. |
| `set` | Every setting that can change while the server runs, with its value. |
| `set <key> <value>` | Changes one, such as `set score-limit 3` or `set welcome-message Hello.` Checked with the same rules as the file. The room is told, in the same words an admin's change gets. Lasts until the server restarts: the file is not written. An emptied room goes back to it, not to the file. An empty value means the default (`set password` removes the password). |
| `reload` | Reads the config file again and applies every setting whose value in the file changed (a `set` since counts as a change, so the file wins). Settings that open ports or write files (`port`, `bind-address`, `query-port`, `listed`, `console`, `persist-bans`, `ban-file`, `persist-mutes`, `mute-file`) are named and wait for a restart. A file with a mistake changes nothing. |
| `stop` | Shuts the server down. Ctrl+C does too. |

Changes to `power-admins` and `admins` reach players already here at once. Somebody taken off
`power-admins` loses the protection straight away and keeps admin for the session; somebody taken
off `admins` keeps it until they next join. A change to `allowed-players` applies to the next
people who join: nobody already in the room is removed.

A server started with no terminal (a service manager, `nohup`, a container)
simply has no console: the end of standard input is not taken as `stop`.
