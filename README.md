# `statusline`

I want to see:

* Model.
* Effort.
* Context use %.
* Dir.
* Branch, if on git.
* HEAD commit SHA (first seven hex digits), plus an afterisk if dirty.
* Usage limits and their reset times.

## `claude`

Two things. One: create `~/.claude/statusline.sh` and make it executable:

```
#!/usr/bin/env bash
# Claude Code status line. Claude Code pipes session JSON to stdin; stdout
# becomes the status line. Docs: https://code.claude.com/docs/en/statusline
#
# Shows: model | effort | context % | dir | git branch + HEAD sha (+ * if dirty)
#        | 5h and weekly usage limits with reset times
#
#   Fable 5 high │ ▓▓▓░░░░░░░ 34% ctx │ ~/jul2/code_distill_maxims
#     │ ⎇ main 5fb16a6* │ 5h 24% ↻15:00 │ wk 81% ↻Mon 09:00
#
# Segments hide when their data is absent (rate_limits: Pro/Max only, after the
# first API response; effort: only on models that support it; git: only in a repo).
#
# Requires: bash, jq.

LC_ALL=C  # stable number parsing for printf %f and English day names from date

input=$(cat)

# Extract all fields in one jq call, joined by \u001f (unit separator).
# Deliberately NOT tab/@tsv: tab is IFS whitespace, so read would collapse
# empty fields (e.g. missing effort) and shift every later field.
fields=$(jq -r '[
  (.model.display_name // "?"),
  (.effort.level // ""),
  (.context_window.used_percentage // 0),
  (.rate_limits.five_hour.used_percentage // -1),
  (.rate_limits.five_hour.resets_at // 0),
  (.rate_limits.seven_day.used_percentage // -1),
  (.rate_limits.seven_day.resets_at // 0),
  (.workspace.current_dir // .cwd // "")
] | map(tostring) | join("\u001f")' <<< "$input" 2>/dev/null)

if [[ -z $fields ]]; then  # jq missing or stdin not valid JSON
  printf 'Claude\n'
  exit 0
fi

IFS=$'\x1f' read -r model effort ctx_pct rl5 rl5_reset rl7 rl7_reset cwd <<< "$fields"

RESET=$'\e[0m'; DIM=$'\e[2m'; CYAN=$'\e[36m'; BLUE=$'\e[34m'
GREEN=$'\e[32m'; YELLOW=$'\e[33m'; RED=$'\e[31m'; MAGENTA=$'\e[35m'

# Truncate a numeric string to its integer part; empty/garbage becomes 0.
int_part() {
  local n=${1%%.*}
  [[ $n =~ ^-?[0-9]+$ ]] && printf '%s' "$n" || printf '0'
}

# Green under 50%, yellow under 80%, red at 80%+.
pct_color() {
  local p; p=$(int_part "$1")
  if (( p >= 80 )); then printf '%s' "$RED"
  elif (( p >= 50 )); then printf '%s' "$YELLOW"
  else printf '%s' "$GREEN"; fi
}

# Format a unix epoch; tries GNU date, then BSD/macOS date. Empty on failure.
fmt_epoch() {
  date -d "@$1" "+$2" 2>/dev/null || date -r "$1" "+$2" 2>/dev/null
}

segs=()

# --- Model + effort ----------------------------------------------------------
seg="${CYAN}${model}${RESET}"
[[ -n $effort ]] && seg+=" ${MAGENTA}${effort}${RESET}"
segs+=("$seg")

# --- Context usage, 10-cell bar ----------------------------------------------
ctx_int=$(int_part "$ctx_pct")
bar=""
for ((i = 0; i < 10; i++)); do
  (( i < ctx_int / 10 )) && bar+="▓" || bar+="░"
done
segs+=("$(pct_color "$ctx_int")${bar} ${ctx_int}%${RESET} ${DIM}ctx${RESET}")

# --- Directory (with ~ for $HOME) --------------------------------------------
if [[ -n $cwd ]]; then
  segs+=("${DIM}${cwd/#"$HOME"/\~}${RESET}")
fi

# --- Git: branch + 7-digit HEAD sha, * when staged or unstaged changes -------
if [[ -n $cwd ]] && git -C "$cwd" rev-parse --is-inside-work-tree &>/dev/null; then
  branch=$(git -C "$cwd" branch --show-current 2>/dev/null)
  sha=$(git -C "$cwd" rev-parse --short=7 HEAD 2>/dev/null)  # empty in a fresh repo
  dirty=""
  git -C "$cwd" diff --no-ext-diff --quiet 2>/dev/null &&
    git -C "$cwd" diff --no-ext-diff --cached --quiet 2>/dev/null || dirty="*"
  seg="${BLUE}⎇ ${branch:-detached}"
  [[ -n $sha ]] && seg+=" ${sha}"
  seg+="${dirty}${RESET}"
  segs+=("$seg")
fi

# --- Usage limits with reset times -------------------------------------------
if [[ $rl5 != "-1" ]]; then
  p=$(printf '%.0f' "$rl5" 2>/dev/null) || p=$(int_part "$rl5")
  t=$(fmt_epoch "$(int_part "$rl5_reset")" '%H:%M')
  segs+=("$(pct_color "$p")5h ${p}%${RESET}${t:+ ${DIM}↻${t}${RESET}}")
fi
if [[ $rl7 != "-1" ]]; then
  p=$(printf '%.0f' "$rl7" 2>/dev/null) || p=$(int_part "$rl7")
  t=$(fmt_epoch "$(int_part "$rl7_reset")" '%a %H:%M')
  segs+=("$(pct_color "$p")wk ${p}%${RESET}${t:+ ${DIM}↻${t}${RESET}}")
fi

out=""
for s in "${segs[@]}"; do
  [[ -n $out ]] && out+=" ${DIM}│${RESET} "
  out+="$s"
done
printf '%s\n' "$out"
```

And two, add this into `~/.claude/settings.json`:

```
  "statusLine": {
    "type": "command",
    "command": "~/.claude/statusline.sh",
    "refreshInterval": 1
  },
```
