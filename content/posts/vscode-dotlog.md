---
title: "dotLOG - Bringing Notepad's Best Kept Secret to VS Code"
date: 2026-01-13
tags:
  - vscode
  - extension
  - open-source
  - productivity
  - tools
categories:
  - Build
thumbnailImage: /images/posts/VSCode-Logo.svg
thumbnailImagePosition: left
summary: "Windows Notepad has had a hidden logging trick since 1992 - type .LOG as the first line of a file and it automatically appends a timestamp every time you open it. I missed it when I moved to VS Code, so I built dotLOG to bring it back."
---

There is a feature in Windows Notepad that has existed since 1992. Most developers have used Notepad at some point in their lives and never once discovered it. It is not hidden behind a menu or a settings toggle. It just works silently, invisibly, if you know the one thing to type.

Open Notepad. Type `.LOG` as the very first line. Save the file. Close it. Now reopen it.

A timestamp has appeared at the bottom. Every time you open that file, a new timestamp is appended automatically. It turns a plain text file into an effortless, chronological log. No app, no database, no configuration. Just a file that remembers when you were last there.

That feature has been sitting in Notepad since Windows 3.1. Over thirty years. Hiding in plain sight.

When I moved my day-to-day notes workflow into VS Code, I lost it. And I missed it more than I expected.

So I built **dotLOG**.

---

## The `.LOG` Trick - A 30-Year-Old Secret

Here is how it works in Notepad, step by step:

1. Open Notepad
2. Type `.LOG` on the very first line (must be the first line, must be uppercase)
3. Save and close the file
4. Reopen the file

You will see something like this appended at the bottom:

```
.LOG

11:42 AM 22/02/2026
```

Close it. Open it again:

```
.LOG

11:42 AM 22/02/2026

11:51 AM 22/02/2026
```

Every time the file is opened, Notepad checks if the first line is `.LOG`, appends the current date and time, and positions your cursor at the end - ready for you to type your next entry. It is append-only by design. Chronological by nature. The structure enforces the habit without requiring any discipline from you.

I have used this pattern for years. Incident notes during an outage. Quick decision records. Meeting scratch pads. Daily work logs. The simplicity is precisely the point - it gets completely out of your way and just captures.

---

## Why It Works So Well

Most note-taking tools ask something of you before you can start. Open the app. Create a new note. Pick a folder. Name it. Choose a template. By the time you have done all of that, the thought you were trying to capture has either changed shape or disappeared entirely.

The `.LOG` pattern asks almost nothing. The file is already open (or takes one click to open). You type. You save. The timestamp is handled. The chronology is guaranteed.

It also ages extraordinarily well. Plain text files from 1992 are still perfectly readable today. They are grep-able, diffable, portable across every operating system, and will outlive every proprietary note-taking app that has ever promised to revolutionise your workflow.

There is something quietly profound about a feature that has survived thirty years of Windows versions completely unchanged. It was right the first time.

---

## The Problem: Nobody Lives in Notepad Anymore

As a developer, I live in VS Code. My terminal is there, my code is there, my git history is there. It made sense to keep my working notes there too - scratch files, investigation notes, architecture thinking. Quick files I open, scribble in, and leave pinned in a tab.

But VS Code does not know about `.LOG`. Open a file that starts with `.LOG` in VS Code and nothing happens. It is just a text file. The one behaviour I wanted most was the one thing the editor did not do.

There are VS Code extensions for journaling. Extensions for daily notes. Extensions for Zettelkasten and second brains and linked thinking graphs. They are all thoughtful tools that ask a lot of you in return.

I did not want any of that. I wanted the Notepad trick. One specific behaviour, applied to one specific file naming pattern, with zero configuration.

I could not find an extension that did it. So I built one.

---

## Introducing dotLOG

**dotLOG** is a VS Code extension that replicates Notepad's `.LOG` behaviour exactly.

Create any file whose name starts with `.LOG` - whether that is literally `.LOG`, or `daily.LOG`, or `incident-2026-02-22.LOG` - and dotLOG will automatically append a timestamp every time you open or save the file, positioning your cursor at the end ready for your next entry.

That is the entire feature. There is no settings panel. There is no configuration file. There are no options to choose. It installs and immediately works, exactly the way Notepad has worked since 1992.

### Installing dotLOG

Search for **dotLOG** in the VS Code Extensions panel (`Ctrl+Shift+X`), or install it from the command line:

```bash
code --install-extension cloudfractal.dotlog
```

Or find it on the [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=cloudfractal.dotlog).

### Using it

1. Create a new file and name it anything ending in `.LOG` - for example `work.LOG`
2. Start typing your first entry
3. Save and close the file
4. Reopen it - a timestamp has been appended and your cursor is waiting at the end

From that point on, every time you open the file, a new timestamp appears. Your log grows chronologically, automatically, with no effort beyond typing your actual notes.

---

## How It Works Under the Hood

For those curious about what is happening behind the scenes - dotLOG is a straightforward VS Code extension that hooks into two editor lifecycle events.

**The trigger condition** is simple: when a document is opened or saved, dotLOG checks whether the filename starts with `.LOG` (case-insensitive). If it does not match, dotLOG does nothing and gets out of the way entirely.

**The timestamp insertion** happens at the end of the document. dotLOG appends a blank line followed by the current date and time in a human-readable format, then moves the cursor to the very end - the same behaviour Notepad has always used.

The two VS Code API events involved are `onDidOpenTextDocument` and `onDidSaveTextDocument`. Both are lightweight document events that fire reliably without any polling or background work.

There is intentionally no configuration surface. No settings to expose means no decisions to make, no documentation to maintain, and no surface area for things to go wrong. The extension is around 50 lines of meaningful code. Small enough to read in five minutes, simple enough to trust completely.

The full source is on GitHub at [github.com/ebbypeter/dotLOG](https://github.com/ebbypeter/dotLOG) if you want to dig in.

---

## Real-World Use Cases

Here is how I actually use `.LOG` files day to day:

**Daily work log.** A file called `work.LOG` that stays pinned in my editor. At the start of each day I open it, the timestamp appears, and I jot down what I am planning to work on. By end of day I have a running record of what actually happened. Useful for weekly summaries, timesheets, and the inevitable "wait, when did we do that?" conversations.

**Incident investigation notes.** When something is on fire and I am in the middle of diagnosing it, I open `incident.LOG` and start capturing what I am seeing, what I am trying, and what the results are. The automatic timestamps mean I never have to manually record when I ran a command or observed a behaviour. When the incident is over, I have a timestamped record ready for a postmortem.

**Decision journal.** Architecture decisions, technology choices, tradeoffs considered. Captured in the moment with the timestamp preserving the when alongside the what. Future-me (and future teammates) have context that no Confluence page ever quite manages to capture.

**Meeting scratch pad.** A quick file open before a call, timestamp already waiting, type during the meeting, done. No template, no structure imposed - just whatever I needed to capture, when I captured it.

**Debugging session notes.** What I tried, in what order, what happened. When you are deep in a problem across multiple sessions, having a timestamped record of what you have already eliminated is surprisingly valuable.

---

## Getting Started in 60 Seconds

1. Install dotLOG from the [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=cloudfractal.dotlog)
2. Create a new file - name it `.LOG`, `daily.LOG`, `work.LOG`, or anything that starts with `.LOG`
3. Type something. Save. Close. Reopen.
4. The timestamp is there. You are done.

No configuration. No sign-up. No sync. Just a file on your disk that knows when you were last there.

---

## It Is Open Source

dotLOG is open source under the MIT licence. The repository is at [github.com/ebbypeter/dotLOG](https://github.com/ebbypeter/dotLOG).

If you find a bug, open an issue. If you have an idea - timestamp format options, perhaps, or configurable trigger patterns - pull requests are welcome. Keep in mind that the guiding philosophy is deliberate minimalism: every feature addition should be measured against whether it keeps the essential zero-friction nature of the tool.

---

## Closing Thought

The `.LOG` trick in Notepad is an example of quiet design. One line of text. No UI. No documentation needed. Completely discoverable by accident. Useful enough to survive unchanged for over thirty years.

dotLOG exists because I wanted that same experience inside the editor I actually use every day. If you have ever kept notes in a plain text file and wished they just knew when you wrote them, give it a try.

It takes about thirty seconds to install and nothing to configure.

*That is kind of the whole point.*

---

*dotLOG is available on the [VS Code Marketplace](https://marketplace.visualstudio.com/items?itemName=cloudfractal.dotlog). Source code on [GitHub](https://github.com/ebbypeter/dotLOG). Issues and contributions welcome.*
clear