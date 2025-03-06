# Mac and tmux Config

Some configurations I always struggle with when things are misconfigured and suddenly not working as expected (e.g. when installing a new Mac or after a Mac upgrade).

## Mac

### Jump word in iTerm2 with Option + Key

I want to jump words with `Option + ←` and `Option + →`, respectively. Configure iTerm like this:

1. open iTerm2 Settings with `CMD + ,`
1. navigate to `Profiles -> Keys -> General`
1. Make sure Left Option key is set to `Esc+`
1. add the following Keyboard Shortcuts:

    | Keyboard Shortcut       | Action               | Escape Sequence (Esc+) |
    | ----------------------- | -------------------- | ---------------------- |
    | `option + Left` (`⌥←`)  | Send Escape Sequence | `b`                    |
    | `option + Right` (`⌥→`) | Send Escape Sequence | `f`                    |

## tmux

### Resize Pane Shortcut

I want to resize tmux panes with the Shortcut `[tmux Shortcut - Ctrl-b] + Ctrl + ←`, `[tmux Shortcut - Ctrl-b] + Ctrl + →`, `[tmux Shortcut - Ctrl-b] + Ctrl + ↑` and `[tmux Shortcut - Ctrl-b] + Ctrl + ↓`.

By default, this should be configured in the tmux config already. You can verify it with

```sh
tmux list-keys | grep resize
```

Which should give you 

```sh
bind-key -r -T prefix       C-Up                   resize-pane -U
bind-key -r -T prefix       C-Down                 resize-pane -D
bind-key -r -T prefix       C-Left                 resize-pane -L
bind-key -r -T prefix       C-Right                resize-pane -R
```

If they don't exist, add it to your `~/.tmux.conf` and source it with `tmux source-file ~/.tmux.conf`.

To make it work in iTerm2, do the following:

1. open iTerm2 Settings with `CMD + ,`
1. navigate to `Profiles -> Keys -> General`
1. Make sure Left Option key is set to `Esc+`
1. Add the following keyboard shortcuts:

    | Keyboard Shortcut  | Action               | Escape Sequence (Esc+) |
    | ------------------ | -------------------- | ---------------------- |
    | `⌃ + Left` (`^←`)  | Send Escape Sequence | `[1;5D`                |
    | `⌃ + Right` (`^→`) | Send Escape Sequence | `[1;5C`                |
    | `⌃ + Up` (`^↑`)    | Send Escape Sequence | `[1;5A`                |
    | `⌃ + Down` (`^↓`)  | Send Escape Sequence | `[1;5B`                |
