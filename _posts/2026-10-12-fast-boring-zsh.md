---
title: "A Fast, Boring zsh: Ditching Oh My Zsh for a Deferred, Pinned Setup"
date: 2026-10-12
categories:
  - sre
  - tooling
tags:
  - zsh
  - shell
  - dotfiles
  - performance
  - kubernetes
  - supply-chain
---

*A fast, modular zsh setup without Oh My Zsh, 53ms to first prompt*

**Code:** [github.com/manubalasree/dotzsh](https://github.com/manubalasree/dotzsh)

## Why your shell should open before you blink

A terminal tab is the most-opened program in an SRE's day, so every extra 100 ms at startup is paid hundreds of times a week. Framework-style zsh setups load completion, plugins and the theme before the first prompt, whether you need them yet or not.

This post builds the opposite: a small, modular zsh config that reaches a prompt in 53 ms on my Mac (81 ms with the same config loaded eagerly) and loads the heavy parts after the prompt is up. It also pins every plugin to a known commit and shows the kube context in the prompt.

The core trick uses [zsh-defer](https://github.com/romkatv/zsh-defer).

## Where the time actually goes

Two things account for about 93% of the time `zprof` measures: `compinit` and the syntax-highlighting plugin's setup. I found that by profiling the same config with every plugin loaded eagerly:

```bash
ZSH_PROFILE=1 zsh -i -c exit
```

| Function | Time (ms) | Share of profiled time |
| --- | --- | --- |
| `compinit` | 13.3 | 69% |
| `_zsh_highlight_load_highlighters` | 3.4 | 18% |
| `_zsh_highlight__function_callable_p` | 1.2 | 6% |
| Everything else (hooks, version checks, autosuggestions init) | 1.4 | 7% |

Profile first: the cost is concentrated in a couple of places, so that is where deferral pays off. (`zprof` counts only time inside functions, about 19 ms here, so it shows where the time goes, not the whole 81 ms.)

## The design: open the doors first, set the room later

Think of a restaurant opening for the evening. The slow version locks the doors until every chair is placed and every candle lit; the fast version opens as soon as there is a table and a menu, and finishes the room while the first guest sits down.

The `.zshrc` follows the same split:

1. **Instant phase.** Only what the first prompt needs: shell options, history, aliases and the prompt itself. These are cheap `setopt` calls and string assignments.
2. **Deferred phase.** Everything you only need once you start typing: completion, autosuggestions, syntax highlighting. [zsh-defer](https://github.com/romkatv/zsh-defer) queues these and runs them when zsh is idle, waiting for input.

The trade-off is a brief window after the prompt appears when completion and highlighting are not active yet. In my test everything was loaded within 0.6 s.

## The layout and the `.zshrc`

The `.zshrc` stays tiny and only decides the order. Each concern lives in its own small file, so a change shows up as a clean diff.

```text
~/.zshrc                      # two phases, nothing else
~/.zsh_configs/
├── core/                     # instant phase
│   ├── options.zsh
│   └── history.zsh
├── deferred/                 # deferred phase
│   └── completion.zsh
├── themes/
│   └── minimal.zsh-theme
├── plugins/                  # fetched by install.sh, pinned by plugins.lock
│   ├── zsh-defer/
│   ├── zsh-autosuggestions/
│   ├── zsh-completions/
│   └── zsh-syntax-highlighting/
└── aliases.zsh
```

The heart of the `.zshrc` is these lines:

```zsh
# Phase 1: instant
for f in "$ZSH_CONFIGS"/core/*.zsh; do
  source "$f"
done
source "$ZSH_CONFIGS/aliases.zsh"
source "$ZSH_CONFIGS/themes/$ZSH_THEME.zsh-theme"

# Phase 2: deferred (runs once the prompt is up and zsh is idle)
# Order matters: completion first, syntax-highlighting last.
zsh-defer source "$ZSH_CONFIGS/deferred/completion.zsh"
zsh-defer source "$ZSH_CONFIGS/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh"
zsh-defer source "$ZSH_CONFIGS/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh"
```

Syntax highlighting is queued last on purpose: its install notes say to source it at the end of the `.zshrc`, because it wraps zle widgets that other plugins define.

## Caching completion and showing the kube context

### 1. Completion, cached

Deferring `compinit` helps, but the bigger win is skipping its full security audit on most shells. The config runs the full version at most once every 24 hours and trusts the cached dump (`-C`) the rest of the time, like inspecting a building daily and trusting the certificate on the wall in between. The cost: a completion installed mid-day appears after the next full run (delete the dump file to force it).

```zsh
setopt local_options extended_glob
if [[ -n $dump(#qN.mh+24) || ! -s $dump ]]; then
  compinit -d "$dump"        # full: scan fpath + audit, at most daily
else
  compinit -C -d "$dump"     # fast: reuse the dump
fi
```

### 2. kubectl completion, generated once

`kubectl completion zsh` spawns a process and prints a large script, so running it in every shell is wasteful. The config caches the output and regenerates it only when the kubectl binary is newer than the cache. (There was no kubectl in my test environment, so I only syntax-checked this block; try it before trusting it.)

```zsh
if [[ ! -s $kc || $kc -ot ${commands[kubectl]} ]]; then
  kubectl completion zsh > "$kc" 2>/dev/null
fi
source "$kc"
compdef k=kubectl     # so the `k` alias completes too
```

### 3. The kube context in the prompt, with no subprocess

If you switch clusters with kubectx, it is easy to run a command against the wrong one, so the prompt shows the active context. It reads it from the kubeconfig in pure zsh instead of spawning `kubectl config current-context` before every prompt:

```zsh
while IFS= read -r line; do
  if [[ $line == current-context:* ]]; then
    ctx="${line#current-context:}"
    ctx="${ctx//[\" ]/}"      # strip quotes and spaces
    break
  fi
done < "$cfg"
```

kubectx rewrites that one `current-context:` line, so the prompt follows every switch. I tested it against a sample kubeconfig and got `repo (main) ⎈ rancher-lab-local ❯`. Limits: it reads only the first file in `$KUBECONFIG` and cannot see a one-off `kubectl --context` flag. Set `PROMPT_KUBE=0` to turn it off.

## Treat your shell like production

Your shell startup runs third-party code with your privileges, in every terminal you open, so treat it like any other dependency.

**Pin the plugins.** `plugins.lock` records each plugin's exact commit, and `install.sh` checks out those commits and nothing newer. Updating becomes a deliberate change: bump the SHA, read the upstream diff, commit.

**Fail safely.** If `zsh-defer` is missing (a fresh machine, a failed clone), the `.zshrc` prints one warning and loads everything eagerly instead of erroring in every new tab. I tested this by deleting the plugin directory.

**Keep secrets out of the repo.** Tokens and per-machine settings go in `~/.zshrc.local`, which the `.zshrc` sources if present and you never commit. Git-ignore the plugins directory too; `install.sh` recreates it from the lock file.

If you want to go one step further, treat startup time as an SLI: pick a budget, such as 50 ms to first prompt on your slowest machine, and check it from a script whenever you change the config.

## The numbers

Time to first prompt on my Mac, in milliseconds, median of 15 runs:

| Setup | Median (ms) | Min / max (ms) |
| --- | --- | --- |
| Same config, everything loaded eagerly | 81 | 74 / 90 |
| Final config, plugins deferred | 53 | 47 / 69 |

The harness starts `zsh -i` in a pseudo-terminal, stops the clock at the first prompt, then waits 0.6 s and checks that autosuggestions, syntax highlighting and completion (1,973 entries) all loaded. They did.

**Read the ratio, not the digits.** Deferring cut time to first prompt by about a third (81 to 53 ms), so the prompt appears roughly 1.5 times sooner. This config defers only three pieces, so a heavier setup has more to gain.

## Try it yourself

From the project folder:

```bash
./install.sh     # copies the config to ~/.zsh_configs and installs ~/.zshrc
exec zsh         # start a fresh shell
```

`install.sh` backs up any existing `~/.zshrc`, then fetches the four plugins at the commits listed in `plugins.lock`: [zsh-defer](https://github.com/romkatv/zsh-defer), [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions), [zsh-completions](https://github.com/zsh-users/zsh-completions) and [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting).

Then measure your own machine:

```bash
ZSH_PROFILE=1 zsh -i -c exit                 # where the time goes
python3 bench/ptybench.py ~ 15 probe         # time to first prompt
```

If your numbers disagree with mine, trust yours. The harness expects the minimal theme's `❯` prompt; for any other prompt pass `--glyph ''`.

Full source on GitHub: [manubalasree/dotzsh](https://github.com/manubalasree/dotzsh)
