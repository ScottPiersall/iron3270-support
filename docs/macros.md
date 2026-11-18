# Writing Iron 3270 macros

A macro is a small text file that drives a 3270 session for you: it types, presses keys, waits for
the host, and checks what's on the screen. Use macros to automate the things you do every day,
such as logging on, opening a data set in ISPF, submitting a job, or paging through a long listing.

This guide starts with recording and a step-by-step example, then covers the whole language,
common recipes, and what to do when a macro doesn't behave.

- [1. Quick start: record a macro](#1-quick-start-record-a-macro)
- [2. Where macros live and how to run them](#2-where-macros-live-and-how-to-run-them)
- [3. Your first hand-written macro](#3-your-first-hand-written-macro)
- [4. The golden rule: wait for the screen](#4-the-golden-rule-wait-for-the-screen)
- [5. Language reference](#5-language-reference)
- [6. Recipes](#6-recipes)
- [7. Troubleshooting](#7-troubleshooting)
- [8. Safety and limits](#8-safety-and-limits)
- [9. Settings](#9-settings)

## 1. Quick start: record a macro

The quickest way to start is to record what you do, then tidy up the result.

1. Connect to a host and get to the screen where the macro should start.
2. Click the **●** (record) button in the terminal toolbar, or run **Iron 3270: Record Macro** from
   the Command Palette. The OIA (the status line under the screen) shows `REC`.
3. Do the task as usual: type, press Tab, Enter, PF keys and so on.
4. Click the **■** (stop) button that replaced **●**, or run **Stop Macro Recording**. Give the
   macro a name, and it opens in the editor.

A recording of opening a data set in ISPF looks something like this:

```
# Recorded 2026-11-18T09:12:44.120Z
# Review before use: macros replay keystrokes into whatever screen is shown.
type "=2"
key Enter
wait unlock
type "'USER.JCL(MYJOB)'"
key Enter
wait unlock
```

Run it with the toolbar **▶** button, or **Iron 3270: Run Macro…**.

What the recorder does:

- Typing, keys and mouse clicks that move the cursor are recorded.
- After every key that sends the screen to the host (Enter, PF and PA keys, Clear and so on), it
  adds `wait unlock`, so the replay waits for the host to answer.
- **Input to password fields is never recorded.** When you type into a hidden (non-display) field,
  the recorder writes `prompt secret password` at the top and `type "${password}"` in its place.
  Each time the macro runs, it asks you for the password.

A recording is a starting point. It replays exactly the same keystrokes, so it only works when the
host shows exactly the same screens. To make it reliable, add the checks described in
[section 4](#4-the-golden-rule-wait-for-the-screen).

## 2. Where macros live and how to run them

### The macro folder

Macros are plain text files with the extension `.3270macro`, all in one folder.

- Open it with **Iron 3270: Open Macros Folder**. By default it's in the extension's own storage.
- To keep macros somewhere else, such as a folder you back up or share with your team, set
  `iron3270.macros.folder` to that folder's path.
- **Iron 3270: New Macro** creates an empty macro there and opens it.
- The first time the folder is used, four example macros are created: `example-logon`,
  `example-ispf-edit`, `example-submit-jcl` and `example-send-and-receive`. They are written for a
  typical z/OS system, so adjust panel texts and options to match yours.

A macro's name is its file name without `.3270macro`. Names can use letters, digits, spaces, `_`,
`.` and `-`.

When a macro is open in the editor, syntax errors are underlined as you type, with the reason.

### Running a macro

A macro always runs in a terminal session. Choose one of these ways:

| How | Where |
|---|---|
| **▶** in the terminal toolbar, or **Iron 3270: Run Macro…** | Pick from the list of macros. |
| A key, mouse wheel or mouse button | In the keymap, bind a key to `macro:NAME`, for example `"Ctrl+L": "macro:logon"`. |
| A custom toolbar button | Add `{ "label": "Logon", "macro": "logon" }` to `iron3270.macros.toolbar`. |
| Automatically after connecting | In the connection's settings, set **Startup macro**. |

While a macro runs, the OIA shows `MACRO`, and a notification shows its name. Only one macro runs
in a session at a time, but different sessions can each run one.

### Stopping a macro

Click the **■** button that replaces **▶** in the toolbar, click **Cancel** on the notification,
or run **Iron 3270: Stop Running Macro**. The macro stops
before its next statement, and also in the middle of a wait.

## 3. Your first hand-written macro

This example logs on to TSO, starting from the screen where you'd type `LOGON`. Every host is
different, so you'll probably need to change the texts the macro waits for.

Create it with **Iron 3270: New Macro**, name it `logon`, and build it up step by step.

**Step 1: ask for the user ID and password.** Never write a password into a macro. `prompt`
asks for a value each time the macro runs, and `prompt secret` hides what you type and keeps it
out of the log.

```
prompt user "TSO user ID"
prompt secret password "Password"
```

**Step 2: make sure the host is ready, then type the LOGON command.**

```
wait unlock
type "LOGON ${user}"
key Enter
```

`wait unlock` waits until the keyboard is unlocked, so the host is ready for input. `${user}`
inside the quotes is replaced by the value of the `user` variable.

**Step 3: wait for the password panel, and type the password.**

```
wait text "Password" 30
type password
key Enter
```

`wait text "Password" 30` waits until the word *Password* appears anywhere on the screen, for up
to 30 seconds. If it doesn't appear, the macro stops with an error rather than typing your password
into the wrong screen. `type password` (without quotes) types the variable's value.

**Step 4: wait for the result, and check it.**

```
wait unlock 60
if screen contains "READY"
  log "Logged on as ${user}"
elseif screen contains "***"
  key Enter
  wait unlock 30
else
  fail "Logon did not reach READY or a *** prompt"
end
```

The whole macro:

```
# Log on to TSO. The password is asked for every time and never stored.
prompt user "TSO user ID"
prompt secret password "Password"

wait unlock
type "LOGON ${user}"
key Enter

wait text "Password" 30
type password
key Enter

wait unlock 60
if screen contains "READY"
  log "Logged on as ${user}"
elseif screen contains "***"
  key Enter
  wait unlock 30
else
  fail "Logon did not reach READY or a *** prompt"
end
```

Save it, click **▶** in the terminal and choose `logon`. The `log` message appears in the
**Iron 3270** Output panel (**View → Output**, then choose *Iron 3270*).

## 4. The golden rule: wait for the screen

A macro runs much faster than you type. If it presses Enter and immediately types the next
command, the host may not have answered yet, or a different screen may have appeared: a message,
a "***" prompt, or an error panel. The macro then types into the wrong place.

So after every key that goes to the host, **wait for the state you expect**, and **check you're
on the right screen** before typing anything that matters:

| Use | When |
|---|---|
| `wait unlock` | The host has answered and the keyboard is free. Good after every Enter or PF key. |
| `wait text "EDIT"` | You know a word the next screen shows. More reliable than `wait unlock` alone, because some applications unlock the keyboard before the screen is complete. |
| `wait gone "MORE..."` | You're waiting for a message or status text to go away. |
| `if screen contains "ISPF"` … `fail` | Before consequential input, such as deleting, submitting or typing a password, confirm the panel. |
| `wait seconds 2` | Only as a last resort. A fixed delay is either too long or too short. |

A good pattern is: press the key, wait for the text that proves you've arrived, then act.

```
key PF3
wait text "Primary Option Menu" 10
type "=3.4"
key Enter
```

## 5. Language reference

### The basics

- One statement per line. Blank lines are ignored.
- `#` starts a comment that runs to the end of the line (except inside quotes).
- Statement names, key names and words such as `contains` can be written in any case: `key Enter`,
  `KEY ENTER` and `key enter` are the same. **Variable names are case-sensitive.**
- Indentation is optional, but it makes `if` and loops much easier to read.
- Rows and columns count from **1**, top left. The OIA shows the cursor position as `row/column`
  (for example `005/015`), so you can put the cursor where you want it and read the position off.

### Text, numbers and variables

- **Text** goes in double quotes: `"LOGON IBMUSER"`. Inside quotes, `\"` is a quote, `\\` is a
  backslash, and `\n` and `\t` are a newline and a tab.
- **Numbers** are plain digits: `30`, `1.5`. There are no negative numbers.
- **Variables** hold text. Create one with `set` or `prompt`, then use it:
  - on its own, as `user` or `$user`, for example `type user`;
  - inside quotes, as `${user}`, for example `type "LOGON ${user}"`.
- Using an unknown variable on its own stops the macro with *undefined variable*. Inside quotes,
  `${unknown}` becomes empty text.
- `a + b` adds when both sides are numbers (`"2" + 3` is `5`), and otherwise joins the text
  (`"USER." + "JCL"` is `USER.JCL`).
- `screen ROW COL LENGTH` reads text from the screen: `set title = screen 1 2 30`.

Variables are shared with any macro you `call`, in both directions.

### Statements

| Statement | What it does |
|---|---|
| `set NAME = VALUE` | Stores a value. `VALUE` can be text, a number, a variable, `screen R C L`, or a `+` of these. |
| `prompt NAME "question"` | Asks you for a value. **Cancel** ends the whole macro run. Without a question, it asks "Value for NAME". |
| `prompt secret NAME "question"` | Same, but hides what you type and replaces the value with `********` in `log` messages. |
| `type VALUE` | Types at the cursor, like the keyboard (field rules apply: protected fields, numeric fields, insert mode). |
| `key NAME` | Presses a 3270 key. See [Keys](#keys). |
| `move ROW COL` | Moves the cursor. |
| `wait unlock [SECONDS]` | Waits until the keyboard unlocks. |
| `wait text "TEXT" [row N] [SECONDS]` | Waits until the text appears anywhere on the screen, or on row `N`. Ignores upper and lower case. |
| `wait gone "TEXT" [SECONDS]` | Waits until the text is no longer on the screen. Ignores case. |
| `wait seconds N` | Waits a fixed time, 0 to 300 seconds. |
| `if COND` … `elseif COND` … `else` … `end` | Runs the first branch whose condition is true. `then` after the condition is optional. |
| `while COND` … `end` | Repeats while the condition is true. |
| `repeat N` … `end` | Repeats `N` times (0 to 100,000). |
| `break` / `continue` | Leaves the loop, or skips to its next round. Only inside `while` or `repeat`. |
| `call "NAME"` | Runs another macro from the macro folder, then carries on. Up to 16 levels deep. |
| `send "LOCAL" "HOST" [options]` | Uploads a file with IND$FILE. See [File transfer](#file-transfer). |
| `receive "HOST" "LOCAL" [options]` | Downloads a file with IND$FILE. |
| `pause "message"` | Shows the message and waits for you to click **Continue**. Closing it ends the macro run. |
| `log "message"` | Writes a line to the *Iron 3270* Output panel. Secret values are masked. |
| `stop` | Ends the macro run successfully, including any macro that called this one. |
| `fail "message"` | Ends the macro run with an error showing the message. |

The waits stop the macro with a *Timed out* error if the state doesn't arrive in time. Without
`SECONDS`, they use `iron3270.macros.defaultTimeoutSeconds` (30 by default). A timeout can be
1 to 3600 seconds.

`wait text` without `row` also matches text that runs from the end of one row onto the next.

### Conditions

Conditions are used by `if`, `elseif` and `while`.

| Condition | True when |
|---|---|
| `screen contains "TEXT"` | The text is anywhere on the screen (within one row). |
| `row N contains "TEXT"` | The text is on row `N`. |
| `at ROW COL is "TEXT"` | The text starts exactly at that row and column. |
| `locked` | The keyboard is locked (the host is busy, or there's an input error). |
| `connected` | The session is connected. |
| `A == B` / `A != B` | Two values are the same / different. |
| `A contains B` | Value `A` includes `B`, for example `title contains "EDIT"`. |
| `not COND` | The condition is false. |
| `COND and COND` / `COND or COND` | Both are true / either is true. |

Things to know:

- **Conditions are case-sensitive**: `screen contains "Ready"` doesn't match `READY`. (The `wait`
  statements ignore case.)
- `and` and `or` are read strictly **left to right**, and there are no parentheses:
  `a or b and c` means `(a or b) and c`. Split complicated tests into nested `if`s.
- There's no `<` or `>`. To count, use `repeat`, or a counter with `==` / `!=`.
- Each side of a condition is a single value. To compare against a combination, `set` it first:
  `set want = "USER." + suffix`, then `if title contains want`.

### Keys

`key` accepts any of these names:

| Group | Keys |
|---|---|
| Send to the host | `Enter`, `Clear`, `PF1` … `PF24`, `PA1` `PA2` `PA3`, `Attn`, `SysReq` |
| Cursor | `Tab`, `BackTab`, `Home`, `NewLine`, `FieldEnd`, `Up`, `Down`, `Left`, `Right`, `Left2`, `Right2`, `WordLeft`, `WordRight` |
| Editing | `EraseEOF`, `EraseInput`, `Insert`, `Delete`, `BackSpace`, `BackSpaceMove`, `DeleteWord`, `DeleteField`, `Dup`, `FieldMark`, `CursorSelect`, `Reset` |

Local functions such as Copy, Paste, Find or Zoom aren't available in macros.

### File transfer

`send` uploads a local file to the host, and `receive` downloads one, using IND$FILE. The host
must have IND$FILE installed, and the session must be at a TSO READY prompt, a CMS Ready prompt or
a CICS screen that accepts the transaction, as it would for a manual transfer.

```
send "C:/src/job.jcl" "'USER.JCL(JOB)'" ascii crlf recfm F lrecl 80
receive "'USER.LISTING'" "C:/out/listing.txt" ascii crlf
```

Options can be in any order:

| Option | Meaning |
|---|---|
| `ascii` (default) / `binary` | Text with code-page conversion, or bytes as they are. |
| `crlf` (default) / `nocrlf` | Convert line ends to and from host records, or keep them. Binary always keeps them. |
| `noremap` | Don't remap special characters in ASCII mode. |
| `tso` (default) / `vm` / `cics` | The host system type. |
| `append` | Add to the end of the target file instead of replacing it. |
| `replace` | Replace the target file (the default). |
| `recfm F` / `recfm V` / `recfm U` | Record format of a new host data set (send only). |
| `lrecl N` | Record length of a new host data set (send only). |

**Every transfer asks you for approval** in a dialog showing both file names. If you decline, the
macro stops. After a transfer, `transfer_ok` is `1` or `0` and `transfer_message` holds the
host's message. A failed transfer stops the macro with that message.

## 6. Recipes

The panel texts in these recipes come from a typical z/OS system. Check them against your own
screens.

### Refuse to run on the wrong screen

Start any macro that types into a specific panel with a check, so running it from the wrong place
does nothing harmful:

```
wait unlock
if not screen contains "ISPF Primary Option Menu"
  fail "Start this macro from the ISPF Primary Option Menu"
end
```

### Jump to an ISPF option from anywhere in ISPF

```
# Go to Data Set List (3.4) from any ISPF panel with a command line.
wait unlock
key Home
type "=3.4"
key Enter
wait text "Data Set List Utility" 10
```

### Open a data set in the editor

```
prompt dsn "Data set to edit, for example 'USER.JCL(MYJOB)'"
wait unlock
key Home
type "=2"
key Enter
wait text "EDIT" 20
key EraseEOF
type dsn
key Enter
wait unlock 30
```

### Page down until the end

```
# Scroll through a listing until the end, for at most 200 pages.
repeat 200
  if screen contains "BOTTOM OF DATA"
    break
  end
  key PF8
  wait unlock 30
end
```

The `repeat 200` limit protects you from scrolling forever if the end marker never appears.

### Find a message on any row and read it

There's no "find and read" statement, but a counter and `row N contains` do the job. This submits
the job being edited, and logs the line that says it was submitted:

```
wait unlock
key Home
type "SUBMIT"
key Enter
wait text "SUBMITTED" 30
set n = 1
repeat 24
  if row n contains "SUBMITTED"
    set message = screen n 1 80
    log message
    break
  end
  set n = n + 1
end
```

On a screen with more rows (for example model 4, 43 rows), raise `repeat 24` to match.

### Build a macro from smaller ones

Keep common steps in their own macros and `call` them. Variables are shared, so a called macro
can return a value:

```
# go-ispf.3270macro
wait unlock
if not screen contains "ISPF"
  type "ISPF"
  key Enter
  wait text "ISPF Primary Option Menu" 30
end
```

```
# morning.3270macro
call "logon"
call "go-ispf"
key Home
type "=SD.ST"
key Enter
wait unlock 30
```

If any called macro fails, stops, or you cancel a prompt in it, the whole run ends.

### Run a macro when you connect

Open the connection's settings, set **Startup macro** to a macro name (for example `logon`), and
save. The macro runs each time the session connects. Pick the file with **Browse** to use a macro
from another folder.

### One key for a whole task

In the keymap editor (**Iron 3270: Edit Keyboard Map…**), bind any key, wheel direction or mouse button
to `macro:NAME`. Or add toolbar buttons in `settings.json`:

```json
"iron3270.macros.toolbar": [
  { "label": "Logon", "macro": "logon" },
  { "label": "SDSF", "macro": "go-sdsf" },
  { "label": "PF3", "action": "PF3" }
]
```

## 7. Troubleshooting

When a macro stops with an error, the message says which macro and which line, for example:

```text
Macro stopped: logon:9: Timed out after 30000 ms waiting for text "Password"
```

| Message | Likely cause and fix |
|---|---|
| `Timed out … waiting for text "X"` | The screen never showed *X*: the panel text differs on your host, or a different screen appeared. Look at the screen, then change the text or add an `if` for the other screen. |
| `Timed out waiting for "X" to disappear` | The text stayed on the screen. Check whether the host needs a key press, such as Enter at a `***` prompt. |
| `Timed out … waiting for keyboard unlock` | The host is slow, or the keyboard locked because of an input error. Increase the timeout, or add `key Reset` after checking `if locked`. |
| `undefined variable "x"` | The variable was never set, or its name differs in upper/lower case. |
| `unknown key "X"` | Check the spelling against [Keys](#keys). |
| `unknown statement "X"` | A typo, or text without quotes at the start of a line. |
| `missing "end"` | An `if`, `while` or `repeat` has no matching `end`. |
| `expected ==, != or contains` | A comparison is missing its operator, or uses one that doesn't exist, such as `<`. |
| `step limit (100000) exceeded` | A `while` loop never ends. Make sure its condition can become false, or use `repeat` with a limit. |
| `session is not connected` | The session disconnected while the macro was running. |
| `file transfer not approved by the user` | You chose not to allow a `send` or `receive`. |

Tips for finding problems:

- **Add `log` lines** to see how far the macro got and what it read, for example
  `log "title is [${title}]"`. The brackets show leading and trailing spaces.
- **Add `pause` lines** to stop at a point and look at the screen before continuing.
- **Check positions** by putting the cursor where you want it and reading the OIA.
- **Text may be split by a field attribute.** Every field starts with an attribute position that
  shows as a space. If `screen contains "USER ID"` doesn't match, try a shorter piece of text.

## 8. Safety and limits

Macros are deliberately limited, so a macro from someone else can't harm your computer:

- A macro can only type, press keys, read the screen and ask you questions. It can't read or write
  files, start programs, use the network, or change settings.
- The only file access is through `send` and `receive`, and you approve every transfer.
- Passwords are never recorded and never stored in macros. Use `prompt secret`. Secret values are
  masked in `log` messages.
- `iron3270.macros.maxSteps` (100,000 statements by default) stops runaway loops.

A macro replays keystrokes into **whatever screen is showing**, with your authority on the host. A
macro that deletes members or submits jobs will do so wherever it happens to be. Read a macro
before you run it, especially one you didn't write, and use `wait text` and `if screen contains`
before anything consequential.

## 9. Settings

| Setting | Default | Meaning |
|---|---|---|
| `iron3270.macros.folder` | (extension storage) | Folder for `.3270macro` files. |
| `iron3270.macros.maxSteps` | `100000` | Most statements one run may execute. |
| `iron3270.macros.defaultTimeoutSeconds` | `30` | Timeout for `wait` statements that don't give one. |
| `iron3270.macros.toolbar` | `[]` | Custom toolbar buttons: `{ "label", "macro" }` or `{ "label", "action" }`. |
