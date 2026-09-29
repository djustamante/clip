# clip

Copy text to your clipboard from the command line, **including over SSH**.

`clip` is a single-file Bash script. Pipe something into it, point it at files, or have it run a command, and the text ends up in your clipboard. When you're logged into a remote machine over SSH, it uses the [OSC 52](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html) terminal escape sequence, so the text lands in the clipboard of the computer you're sitting at, not the server's.

```console
$ ls -l | clip
clip: copied 412 bytes, 9 lines -> [c] via osc52/none

$ clip -e 'git diff' -L diff      # copy a diff as a Markdown code block
$ make 2>&1 | clip -t             # watch the output AND copy it
$ clip -H src/a.c src/b.c         # copy multiple files with headers
```

## Features

- **Works over SSH, tmux, and GNU screen** via OSC 52, with automatic multiplexer wrapping
- **Native clipboard support** when you're local: `wl-copy`, `xclip`, `xsel`, `pbcopy`, `clip.exe`, `termux-clipboard-set`
- **Auto-detection**: OSC 52 over SSH, native tool otherwise, OSC 52 as the fallback
- **Flexible input**: stdin, one or more files, or the output of a command
- **Tee mode**: print to the screen and copy at the same time, streaming live, usable mid-pipeline
- **Text tweaks**: first N lines, whitespace trimming, strip trailing newline, Markdown code fences, file headers
- **Any selection**: clipboard, primary, secondary, or cut buffers (OSC 52 targets)
- **Safe by default**: refuses payloads too large for most terminals instead of silently failing
- No dependencies beyond Bash and standard tools (`base64`, `awk`, `sed`, `fold`, `head`, `tail`, `wc`, `mktemp`)

## Installation

```sh
# Download
curl -fsSL -o clip https://raw.githubusercontent.com/<your-username>/<repo>/main/clip

# Install
install -m 755 clip ~/.local/bin/clip
```

Or clone the repo and copy the script wherever you like:

```sh
git clone https://github.com/<your-username>/<repo>.git
install -m 755 <repo>/clip ~/.local/bin/clip
```

Make sure `~/.local/bin` is on your `PATH`.

## Usage

```
clip [options] [FILE ...]
command | clip [options]
```

With no `FILE` (or with `-` as a file), `clip` reads standard input.

### Options

| Option | Description |
|---|---|
| **Input** | |
| `FILE ...` | Copy the contents of one or more files |
| `-e`, `--exec CMD` | Run `CMD` in Bash and copy its output |
| `-C`, `--clear` | Clear the clipboard instead of copying |
| **Output** | |
| `-t`, `--tee` | Also print the input to stdout (streams live) |
| `-q`, `--quiet` | Suppress the status message on stderr |
| **Transform** (applied in this order) | |
| `-l`, `--lines N` | Keep only the first `N` lines |
| `-T`, `--trim` | Strip trailing whitespace and leading/trailing blank lines |
| `-H`, `--header` | Prefix each file with a `==> name <==` header |
| `-M`, `--md`, `--fence` | Wrap the text in a Markdown code fence |
| `-L`, `--lang LANG` | Set the fence language (implies `--fence`) |
| `-n`, `--no-newline` | Strip the trailing newline(s) |
| **Destination** | |
| `-s`, `--selection SEL` | OSC 52 target: `c` clipboard (default), `p` primary, `q` secondary, `s` select, `0`-`7` cut buffers. Combine them: `cp` |
| `-P`, `--primary` | Shortcut for `-s p` |
| `-m`, `--mode MODE` | `auto` (default), `osc52`, or `native` |
| `-p`, `--passthrough W` | OSC 52 wrapping: `auto` (default), `tmux`, `screen`, or `none` |
| `-x`, `--max BYTES` | Size cap for OSC 52 (default `74994`) |
| `-F`, `--force` | Ignore the size cap and send anyway |
| **Misc** | |
| `-h`, `--help` | Show help |
| `-V`, `--version` | Show version |

Short flags can be bundled (`-tn` is `-t -n`). Flags that take a value go on their own (`-s p`).

### Examples

```sh
# Pipe anything in
cat ~/.ssh/id_ed25519.pub | clip -n      # -n drops the trailing newline
journalctl -u nginx --since today | clip

# Files
clip notes.txt
clip -H a.py b.py                        # with ==> filename <== headers
clip -T -l 20 build.log                  # first 20 lines, tidied up

# Run a command and copy its output
clip -e 'df -h'
clip -e 'git diff HEAD~1' -L diff        # paste straight into a GitHub comment

# See it and copy it
make 2>&1 | clip -t
tail -f app.log | clip -t | grep ERROR   # tee is live, so streams keep flowing

# Pick where it goes
clip -P < snippet.txt                    # primary selection
clip -s cp < snippet.txt                 # clipboard AND primary
clip -m native < snippet.txt             # force the local clipboard tool
clip -m osc52 < snippet.txt              # force OSC 52

# Wipe the clipboard
clip -C
```

## How it works

Terminals that support OSC 52 accept an escape sequence of the form:

```
ESC ] 52 ; <selection> ; <base64 data> BEL
```

and set the local clipboard to the decoded data. Because the sequence travels through your SSH session's output stream like any other text, it reaches your local terminal emulator no matter how many hops away the program is running.

`clip` writes the sequence to `/dev/tty` (falling back to stderr), so it never pollutes your pipeline's stdout and `--tee` works correctly.

In `auto` mode:

1. If `SSH_CONNECTION`, `SSH_TTY`, or `SSH_CLIENT` is set, use OSC 52.
2. Otherwise, use the first native tool available (Wayland, then X11, then macOS, WSL, Termux).
3. If no native tool works, fall back to OSC 52.

## Terminal support

OSC 52 support varies by terminal, and some require a setting to be turned on.

| Terminal | Notes |
|---|---|
| kitty, WezTerm, foot, Windows Terminal | Generally work out of the box |
| Alacritty | Works; controlled by its `osc52` setting |
| iTerm2 | Enable *Settings > General > Selection > Applications in terminal may access clipboard* |
| xterm | May need `allowWindowOps` enabled |
| GNOME Terminal and other older VTE-based terminals | Generally not supported |

Check your terminal's documentation if nothing happens. Support and defaults change between versions.

### tmux

Add to `~/.tmux.conf`:

```tmux
set -g set-clipboard on
set -g allow-passthrough on
```

`clip` wraps the sequence in tmux's DCS passthrough automatically when `$TMUX` is set. Reload with `tmux source-file ~/.tmux.conf`.

### GNU screen

`clip` splits the sequence into chunks for screen automatically when `$STY` is set. This path is best-effort. Override with `-p none`, `-p tmux`, or `-p screen` if auto-detection guesses wrong.

## Limitations

- **Size limit.** Many terminals silently drop OSC 52 payloads above roughly 100,000 base64 characters. `clip` therefore refuses inputs over 74,994 bytes by default. Use `-F` to try anyway or `-x` to change the cap.
- **Write-only.** `clip` doesn't read the clipboard back. Most terminals block OSC 52 reads for security reasons.
- **Text only.** It's designed for text. Binary data isn't a goal.
- **Clearing over OSC 52** sends an empty payload, which some terminals ignore.

## Troubleshooting

**Nothing lands in my clipboard over SSH.**
Check that your local terminal supports OSC 52 and that it's enabled (see above). Try a direct test from the remote shell:

```sh
printf '\033]52;c;%s\a' "$(printf 'hello' | base64)"
```

If that doesn't work, it's the terminal, not `clip`.

**It works outside tmux but not inside.**
Make sure `allow-passthrough on` is set and that your tmux is version 3.3 or newer.

**"payload is N bytes; most terminals cap OSC 52..."**
Your input is over the size cap. Trim it (`-l`, `-T`), or try `-F`.

**The status message is noisy.**
Use `-q`.

## Contributing

Issues and pull requests are welcome. Please run `bash -n clip` (and `shellcheck clip` if you have it) before submitting.

## License

MIT. Add a `LICENSE` file to the repo root; GitHub can generate one for you when you create the repository or through *Add file > Create new file > LICENSE*.
