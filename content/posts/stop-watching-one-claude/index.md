---
title: "Stop Watching One Claude Code Session. Run Four."
date: 2026-05-09
author: "Alan Mond"
tags: ["claude-code", "tmux", "ai", "productivity", "terminal", "dotfiles"]
draft: false
cover:
  image: "cover.jpg"
  alt: "Run Four. Title card for a post about running four parallel Claude Code agents in a tmux 2x2 grid."
  relative: true
---

### TL;DR

Claude Code is best when you supervise it: read the diff, push back, redirect. But supervising one agent at a time leaves you idle while it thinks. The productivity hack is to run four agents in parallel, each in its own pane, and rotate through approvals as they pause for input. tmux is the right tool for that, except every default tmux config breaks the things that make a Mac terminal a Mac terminal. Mouse scrolling. Drag-to-copy. Click-to-focus. This post is the 40 lines of config that fix all of it. [Gist here.](https://gist.github.com/alan707/1358db6e81072eaafb1a92105a97fc7b)

<video src="demo.mp4" autoplay loop muted playsinline preload="metadata" style="width: 100%; max-width: 1000px; border-radius: 6px; display: block; margin: 1.5rem auto;"></video>

### The supervised parallel-agent workflow

Here's the pattern I keep landing on with Claude Code. Four panes, four shapes of work:

- **Pane 1, drafting:** *"Outline a blog post about the cheapest viable stack for launching a SaaS MVP in 2026."*
- **Pane 2, building:** *"Pick up `bazel-contrib/bazel-docs` issue #397 (Fix broken links surfaced by Mintlify) and start a plan."*
- **Pane 3, researching:** *"Compare low-cost managed Postgres options under $20/month for a side project. Cover Supabase, Neon, Railway, Fly Postgres, and Render. End with a 2-3 sentence recommendation paragraph I can drop into a blog post."*
- **Pane 4, reviewing:** *"Review `bazel-contrib/bazel-docs` PR #438 (Add hero graphic, reorganize landing page). Look for clarity, hierarchy issues, and anything that would weaken the landing page pitch."*

Each one is a separate `claude` session with its own conversation. They're all running. They all pause on approvals. "Can I run this command?" "This file looks risky, ok to edit?" "Here's the plan, sign off?" That's exactly when I want to be looking at them. While one is thinking, I'm reviewing another. While I'm typing context into a third, the fourth is waiting for me. The dead time disappears.

The four shapes (*draft, build, research, review*) cover most of what I do at the keyboard in a day. If your work fits a different mix, swap them in. The point is that supervising one agent at a time wastes you, and supervising four works.

The bottleneck stops being "how fast can the model think" and starts being "how fast can I read and respond." That's the right bottleneck. That's *me actually doing my job*.

The catch: this only works if switching between agents is friction-free. If I have to leave the keyboard, hunt for a window, mash a keybinding, or fight scrollback, it falls apart. So the terminal has to feel native.

### Yeah, but why?

Fair question. Claude Code already parallelizes work in a few ways, and the four-pane setup might look like reinventing what the tool already does:

- **Subagents** (the Agent tool) spawn parallel research and coding tasks inside one conversation.
- **Background tasks** (`run_in_background`) let one Claude fire off a long-running command and keep going.
- **Git worktrees** (`EnterWorktree`) isolate experiments without leaving the session.
- **Session forking** (`claude --fork-session` or `/branch`) branches one conversation at a fork-in-the-road moment so you can explore approach A and approach B from the same shared context.

All real, all useful. A couple of them you'll actually want to *combine* with the four-pane setup. Worktrees plus four panes is a killer combo: each pane gets its own isolated working tree, and four agents can edit overlapping files without conflicts.

But none of them solve the same problem. Subagents, background tasks, and forking all parallelize *the agent* within one task you're driving. The four-pane pattern parallelizes *you* across unrelated tasks you're actively supervising. Four conversations, four contexts, four approval streams I'm bouncing between in real time. Subagents can't help with that part because I'm the bottleneck, and the only way to widen me is to give me four narrow streams to attend instead of one wide one.

(Anthropic's **Managed Agents** is also out there, but it's a different product class: cloud-hosted, headless, fire-and-forget. It's the opposite move. Useful, just orthogonal.)

### What "native" means here

I want all of these to keep working, exactly like they would in vanilla Terminal.app or Ghostty:

- **Mouse wheel** scrolls the visible scrollback of whichever pane I'm pointing at.
- **Drag-select** copies to the system clipboard. `Cmd-V` pastes anywhere.
- **Click on a pane** focuses it.
- **Drag the divider** resizes splits.
- **Shift+Wheel** scrolls the terminal's own scrollback (the one above tmux).
- **24-bit color and italics** render correctly in `vim`, `bat`, `eza`.
- **No weird Escape lag** in vim or fzf.

Default tmux gets approximately none of these right out of the box. The most painful one, and the reason I started caring, is the wheel.

### The wheel paper cut, specifically

Claude Code uses the terminal's *alternate screen*, the same mode `vim`, `less`, and `htop` use. On the alternate screen, your terminal emulator translates the mouse wheel into up-arrow and down-arrow keystrokes. Inside Claude Code, that means wheel-up doesn't scroll the conversation. It cycles through your *prompt history*. Every time you idly scroll while reading, you're rewinding your own typing. Maddening.

With four Claude Code panes open, this happens constantly. You glance at pane 3, scroll up to re-read a tool result, and the prompt input gets clobbered.

The fix is to put tmux between your terminal and Claude Code, and make tmux intercept the wheel before the alternate-screen app sees it. tmux has a copy mode with its own scrollback buffer per pane. We just need to route wheel-up there. The default tmux mouse binding doesn't do this on alt-screen. It forwards arrow keys. So we override it explicitly.

That single override is the unlock for the whole workflow.

### The config

```bash
# Feel native: default prefix (C-b), default shell, no exotic remaps.

# True color + italics. tmux-256color exposes sitm/ritm; RGB advertises 24-bit
# to children so vim/bat/eza render real colors instead of 256-cube approximations.
set -g default-terminal "tmux-256color"
set -as terminal-features ",*256col*:RGB"
set -as terminal-features ",xterm*:RGB"
set -as terminal-overrides ",*256col*:Tc"

# No delay on Escape (vim, fzf, claude all feel snappier).
set -sg escape-time 0

# 1-indexed windows + panes, renumber on close so gaps don't accumulate.
set -g base-index 1
setw -g pane-base-index 1
set -g renumber-windows on

# Title updates from the running process.
setw -g automatic-rename on
set -g set-titles on
set -g set-titles-string "#T"

# Mouse on. Critical part: override the default WheelUp binding so wheel
# enters tmux copy-mode even on the alternate screen (claude code, vim, less).
# Without this, tmux translates wheel-up into up-arrow on alt-screen apps,
# which in claude code scrolls prompt history instead of conversation scrollback.
# Shift+Wheel is intentionally left unbound so the terminal emulator gets it
# and scrolls its own native scrollback buffer.
set -g mouse on
bind -n WheelUpPane   if -F "#{pane_in_mode}" "send-keys -M" "copy-mode -e"
bind -n WheelDownPane if -F "#{pane_in_mode}" "send-keys -M" "send-keys -M"

# Bigger scrollback than the 2000-line default. Claude Code conversations get long.
set -g history-limit 50000

# Clipboard: OSC 52 for SSH + terminals that support it (iTerm2, Ghostty,
# Alacritty, kitty, WezTerm), pbcopy as the local belt-and-suspenders path.
set -g set-clipboard on
bind -T copy-mode    MouseDragEnd1Pane send -X copy-pipe-no-clear "pbcopy"
bind -T copy-mode-vi MouseDragEnd1Pane send -X copy-pipe-no-clear "pbcopy"
bind -T copy-mode    Enter             send -X copy-pipe-and-cancel "pbcopy"
bind -T copy-mode-vi Enter             send -X copy-pipe-and-cancel "pbcopy"

# Auto 2x2 grid + name the session "main". One window, four panes, four claudes.
set-hook -g session-created 'split-window -h ; split-window -v ; select-pane -L ; split-window -v ; select-pane -U ; select-layout tiled ; rename-session main'

# Minimal status bar: session name on the right, nothing else.
set -g status-style "bg=default fg=default"
set -g status-left ""
set -g status-right "#S "
set -g status-right-length 40
set -g window-status-format ""
set -g window-status-current-format ""
set -g status-justify left
```

Drop it in `~/.tmux.conf`, kill any old tmux server with `tmux kill-server`, and run `tmux`. You'll land in a 2x2 grid named `main`. In each pane, run `claude`. You're set up.

### Why each block earns its place

**The 2x2 grid on session start.** This is the whole frame of the workflow. The `session-created` hook fires once per `tmux new`, lays out four equal panes via `select-layout tiled`, and names the session `main`. It does *not* fire on attach, which is what you want. You don't want the layout to redraw every time you `tmux a` back in. Tomorrow morning, `tmux a` drops you right back into the same four conversations.

**The wheel override.** As discussed. With this in place, scrolling in any pane (even when Claude Code is using the alternate screen) enters tmux copy mode and scrolls *that pane's own* buffer. Your prompt history is untouched. You can scroll back to re-read a tool result, then `q` to drop back into the live conversation.

**Mouse on, generally.** Click a pane to focus it. Drag the divider to resize splits. Drag-select to copy. All the little physical-feeling things you'd do in any normal terminal still work, even though there are now four programs sharing one window.

**Clipboard via OSC 52 and pbcopy.** Drag-select in any pane copies straight to your Mac clipboard. Two paths are wired up. `pbcopy` is the local belt-and-suspenders: drag-select fires it directly, so this works in every Mac terminal including stock Terminal.app. OSC 52 is the bonus: it works *over SSH* in iTerm2, Ghostty, Alacritty, kitty, and WezTerm. If one of your four panes is `ssh`'d into a remote box running its own tmux, selecting text on the remote lands in your local clipboard. No `scp`, no manual fiddling. (Terminal.app doesn't speak OSC 52, so the SSH hop only works locally there. Not a problem if you're working on your own machine.)

**Shift+Wheel left unbound.** The terminal emulator gets it. In Ghostty, iTerm2, Alacritty, that means you can shift-scroll *above* tmux to see whatever was on screen before tmux started. Best of both worlds.

**No escape delay.** When you're context-switching between four agents quickly, every 500ms `Esc` lag is a tax. Setting it to 0 has been correct for a decade.

**Window numbering and titles.** Each pane's title auto-updates from the running process, so even with four `claude` instances you can glance at the pane border and see which is which. `automatic-rename` does the work.

**Big scrollback.** Claude Code conversations get long. 50,000 lines is plenty. tmux's default of 2,000 will cut off the middle of a long agent run.

**Minimal status bar.** Default tmux status bars are loaded with time, date, hostname, battery, weather, and CPU graphs. With four panes you need every row of vertical space for the actual agent output. Just the session name on the right, nothing else.

### How a session actually feels

You run `tmux`. Four panes open. You start `claude` in each one and kick off the four prompts above. Then:

- **Bounce focus** by clicking any pane. (Or `Ctrl-b` + arrow if you're keyboard-only.)
- **Read scrollback** by mousing over any pane and wheeling up. `q` exits scrollback.
- **Grab a snippet** by drag-selecting. Hit `Cmd-V` in another pane, in your editor, in Slack, wherever.
- **Approve** by clicking into the relevant pane and typing `y` or your reply. Pane focus follows the click.
- **Close one out** by `Ctrl-b x`. The remaining three reflow with `select-layout tiled` (or rebind it to a key if you do this often).
- **Detach** with `Ctrl-b d`. All four agents keep running.
- **Come back** with `tmux a`. Same four conversations, picked up exactly where you left them.

The whole thing feels less like "I'm using tmux" and more like "I have four terminals open." Which is the point.

### A real handoff between panes

The clipboard isn't just a convenience. It's the connective tissue between the four work shapes. A flow I run all the time:

1. Pane 3 finishes the managed Postgres comparison and prints a recommendation paragraph.
2. I drag-select that paragraph. It's now in my Mac clipboard.
3. I click into pane 1 (the blog outline), paste, and tell that Claude: *"use this comparison as the foundation for the database section of the post."*
4. Pane 1 starts incorporating it. Meanwhile pane 3 is idle, pane 2 is mid-plan and waiting for my approval, pane 4 just finished surfacing three issues with the bazel-docs PR.
5. I approve pane 2's plan, click pane 4 to read its findings, and queue up replies to both.

Research feeds drafting. Review feeds your next PR comment. Building feeds your next test. The 2x2 grid plus a working clipboard turns four parallel agents into one connected workflow.

### One closing thought

The big productivity unlock with AI coding agents isn't that they're fast. It's that they let you parallelize the parts of work that used to be serial. You can be the bottleneck on four threads of progress at once instead of one. But that only pays off if the substrate (the terminal, the window manager, the muscle memory) gets out of the way. tmux is the right primitive for this. It just needs 40 lines of config to stop fighting your terminal and start amplifying it. Run four. Approve fast. Ship more.
