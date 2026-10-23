#!/usr/bin/env bash
# Minecraft Java multi-version text UI launcher (Git Bash for Windows)
#
# Run from Git Bash (mintty):
#     ./mclauncher-tui.sh [InstallDir]
#     ./mclauncher-tui.sh --install 1.8.9 [InstallDir]   (non-interactive, for CI)
#
# Keys: Up/Down or W/S move, PgUp/PgDn/Home/End jump, Enter select, Esc back.
#
# Layout (shared like the official launcher, same as the PowerShell version):
#     <Root>/libraries, <Root>/assets            shared by all versions
#     <Root>/versions/<id>/<id>.json/.jar/natives
#     <Root>/instances/<id>                      per-version game dir (worlds, options)
#
# Requirements: Git for Windows (bash, curl >= 7.66, sha1sum, md5sum, find).
# jq is used for JSON; if it is not on PATH it is downloaded to <Root>/tools/jq.exe.
# unzip is used if present, otherwise a tiny PowerShell helper is used for JAR files.

set -o pipefail
shopt -u patsub_replacement 2>/dev/null

# ------------------------------------------------------------------ paths / platform
INSTALL_DIR=''
HEADLESS_VERSION=''
while (( $# )); do
    case $1 in
        --install) HEADLESS_VERSION=${2:-}; shift; [[ $# -gt 0 ]] && shift ;;
        *)         INSTALL_DIR=$1; shift ;;
    esac
done
if [[ -z $INSTALL_DIR ]]; then
    HERE=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
    INSTALL_DIR="$HERE/Minecraft"
fi
HAVE_CYGPATH=''
command -v cygpath >/dev/null 2>&1 && HAVE_CYGPATH=1
to_posix() { if [[ -n $HAVE_CYGPATH ]]; then cygpath -u "$1"; else printf '%s' "$1"; fi; }
to_mixed() { if [[ -n $HAVE_CYGPATH ]]; then cygpath -m "$1"; else printf '%s' "$1"; fi; }

INSTALL_DIR=$(to_posix "$INSTALL_DIR")
mkdir -p "$INSTALL_DIR" || { echo "Cannot create $INSTALL_DIR"; exit 1; }
ROOT=$(cd "$INSTALL_DIR" && pwd)
ROOTM=$(to_mixed "$ROOT")      # C:/... form, safe to hand to Java / curl config files

LIB_ROOT=$ROOT/libraries;        LIB_ROOTM=$ROOTM/libraries
ASSET_ROOT=$ROOT/assets;         ASSET_ROOTM=$ROOTM/assets
AIDX_ROOT=$ASSET_ROOT/indexes
AOBJ_ROOT=$ASSET_ROOT/objects;   AOBJ_ROOTM=$ASSET_ROOTM/objects
VERSIONS=$ROOT/versions
INST_ROOT=$ROOT/instances
TOOLS=$ROOT/tools
JQDIR=$TOOLS/jq
CONFIG_PATH=$ROOT/launcher.json
MANIFEST_FILE=$ROOT/version_manifest_v2.json
MANIFEST_URL='https://piston-meta.mojang.com/mc/game/version_manifest_v2.json'
MANIFEST_URL2='https://launchermeta.mojang.com/mc/game/version_manifest_v2.json'
UA='Mozilla/5.0 (Windows NT 10.0; Win64; x64) MCLauncher-TUI/2.0'
curlx() { curl -A "$UA" "$@"; }
JQ_URL='https://github.com/jqlang/jq/releases/download/jq-1.7.1/jq-windows-amd64.exe'

case "${PROCESSOR_ARCHITEW6432:-${PROCESSOR_ARCHITECTURE:-AMD64}}" in
    ARM64)        ARCH=arm64 ;;
    AMD64|x86_64) ARCH=x86_64 ;;
    *)            ARCH=x86 ;;
esac
BITS=64; [[ $ARCH == x86 ]] && { BITS=32; JQ_URL='https://github.com/jqlang/jq/releases/download/jq-1.7.1/jq-windows-i386.exe'; }
OSVER=$(uname -s | sed -E 's/^[^-]*-([0-9]+\.[0-9]+)-?([0-9]*).*/\1.\2/')

MANIFEST_OK=0
JAVA_SCANNED=''
JAVA_PATHS=(); JAVA_MAJORS=()
TMPD=$(mktemp -d)

# ------------------------------------------------------------------ colors / console helpers
ESC=$'\e'
C_RST="${ESC}[0m";  C_RED="${ESC}[91m";   C_GREEN="${ESC}[92m"; C_YELLOW="${ESC}[93m"
C_CYAN="${ESC}[96m"; C_DCYAN="${ESC}[36m"; C_GRAY="${ESC}[37m";  C_DGRAY="${ESC}[90m"
C_HL="${ESC}[30;46m"
HLINE="+$(printf '%58s' '' | tr ' ' '-')+"

clear_screen()   { printf '\033[2J\033[3J\033[H'; }
cursor_visible() { if [[ $1 == 1 ]]; then printf '\033[?25h'; else printf '\033[?25l'; fi; }
window_rows()    { local h; h=$(tput lines 2>/dev/null) || h=30; [[ $h =~ ^[0-9]+$ ]] || h=30; printf '%s' "$h"; }

cleanup() { cursor_visible 1; rm -rf "$TMPD"; }
trap cleanup EXIT
trap 'exit 130' INT TERM

die() { printf '\n  %sError: %s%s\n' "$C_RED" "$*" "$C_RST"; exit 1; }

# ------------------------------------------------------------------ jq bootstrap + jq programs
ensure_jq() {
    if [[ -f $TOOLS/jq.exe ]]; then JQ=$TOOLS/jq.exe; return 0; fi
    if command -v jq >/dev/null 2>&1; then JQ=$(command -v jq); return 0; fi
    echo "jq not found - downloading it to $TOOLS ..."
    mkdir -p "$TOOLS"
    if curlx -fL --retry 3 -o "$TOOLS/jq.exe.download" "$JQ_URL" && mv -f "$TOOLS/jq.exe.download" "$TOOLS/jq.exe"; then
        JQ=$TOOLS/jq.exe; return 0
    fi
    echo "Could not download jq. Install it (winget install jqlang.jq) or put jq.exe in $TOOLS"
    return 1
}
# Wrapper: strips CR from jq.exe output so read/mapfile never see stray \r
jqx() { "$JQ" "$@" | tr -d '\r'; }

write_jq_files() {
    mkdir -p "$JQDIR"
    # Shared rule evaluation (last matching rule wins). Globals: $arch, $osver
    local RULES_DEF
    RULES_DEF=$(cat <<'EOF'
def rules_ok($feat):
  if . == null or length == 0 then true
  else reduce .[] as $r (false;
    (
      (if $r.os then
         (($r.os.name // "windows") == "windows")
         and (($r.os.arch // $arch) == $arch)
         and (($r.os.version // null) == null or ($osver | test($r.os.version)))
       else true end)
      and
      (if $r.features then ($r.features | to_entries | all(.value == ($feat[.key] // false))) else true end)
    ) as $m
    | if $m then ($r.action == "allow") else . end)
  end;
EOF
)
    # Library work list: url|relpath|sha1|native|onclasspath
    { printf '%s\n' "$RULES_DEF"; cat <<'EOF'
.libraries[]
| select(.rules | rules_ok({}))
| select(((.name // "") | test("natives-windows-arm64$")) == false or $arch == "arm64")
| select(((.name // "") | test("natives-windows-x86$")) == false or $arch == "x86")
| . as $lib
| (.downloads // null) as $dl
| (
    if ($dl != null and $dl.artifact != null and $dl.artifact.path != null) then
      [$dl.artifact.url, $dl.artifact.path, ($dl.artifact.sha1 // ""),
       (if ($lib.name | test(":natives-windows")) then "1" else "0" end), "1"]
    elif ($dl == null and $lib.name != null) then
      ($lib.name | split(":")) as $p
      | if ($p | length) >= 3 then
          (if ($p | length) >= 4 then "-" + $p[3] else "" end) as $suf
          | (($p[0] | gsub("\\."; "/")) + "/" + $p[1] + "/" + $p[2] + "/" + $p[1] + "-" + $p[2] + $suf + ".jar") as $rel
          | (($lib.url // "https://libraries.minecraft.net/") | if endswith("/") then . else . + "/" end) as $base
          | [$base + $rel, $rel, ($lib.sha1 // ""), "0", "1"]
        else empty end
    else empty end
  ),
  (
    if ($lib.natives != null and $lib.natives.windows != null and $dl != null and $dl.classifiers != null) then
      ($lib.natives.windows | sub("\\$\\{arch\\}"; $bits)) as $cn
      | ($dl.classifiers[$cn] // null)
      | select(. != null and .path != null)
      | [.url, .path, (.sha1 // ""), "1", "0"]
    else empty end
  )
| join("|")
EOF
    } > "$JQDIR/work.jq"

    # Launch arguments (jvm or game) after rule filtering; placeholders are expanded in bash
    { printf '%s\n' "$RULES_DEF"; cat <<'EOF'
def feat: {is_quick_play_multiplayer: ($qp == "1")};
.arguments[$sel][]
| if type == "string" then .
  else select(.rules | rules_ok(feat)) | (.value | if type == "array" then .[] else . end)
  end
EOF
    } > "$JQDIR/args.jq"

    # Merge a child profile (Fabric/Forge style, "inheritsFrom") onto its resolved parent: slurp [base, child]
    cat > "$JQDIR/merge.jq" <<'EOF'
.[0] as $b | .[1] as $v
| $b
| .libraries = (($v.libraries // []) + ($b.libraries // []))
| (if $v.mainClass then .mainClass = $v.mainClass else . end)
| (if $v.minecraftArguments then .minecraftArguments = $v.minecraftArguments else . end)
| (if $v.arguments then
     .arguments = {
       jvm:  (($b.arguments.jvm  // []) + ($v.arguments.jvm  // [])),
       game: (($b.arguments.game // []) + ($v.arguments.game // []))
     }
   else . end)
| (if $v.assetIndex  then .assetIndex  = $v.assetIndex  else . end)
| (if $v.javaVersion then .javaVersion = $v.javaVersion else . end)
| .id = $v.id
EOF
    printf '. + {_jarId: $jar}\n' > "$JQDIR/base.jq"

    # PowerShell fallback for JAR handling when unzip is not installed
    cat > "$TOOLS/zip.ps1" <<'EOF'
param([string]$Jar, [string]$Dest, [string]$Mode)
Add-Type -AssemblyName System.IO.Compression.FileSystem
$zip = [IO.Compression.ZipFile]::OpenRead($Jar)
try {
    if ($Mode -eq 'check') {
        if (-not $zip.GetEntry('net/minecraft/client/main/Main.class')) { exit 2 }
    } else {
        foreach ($e in $zip.Entries) {
            if ([string]::IsNullOrWhiteSpace($e.Name) -or $e.Name -notmatch '(?i)\.dll$') { continue }
            $out = Join-Path $Dest $e.Name
            $in = $e.Open()
            try {
                $fs = [IO.File]::Open($out, 'Create', 'Write', 'None')
                try { $in.CopyTo($fs) } finally { $fs.Dispose() }
            } finally { $in.Dispose() }
        }
    }
} finally { $zip.Dispose() }
EOF
}

# ------------------------------------------------------------------ config
load_config() {
    CFG_version=1.8.9; CFG_listMode=release; CFG_username=Player; CFG_memoryMB=2048
    CFG_javaPath=''; CFG_server=''; CFG_extraJvm=''; CFG_sharedGameDir=false
    [[ -f $CONFIG_PATH ]] || return 0
    local k v
    for k in version listMode username memoryMB javaPath server extraJvm sharedGameDir; do
        v=$(jqx -r --arg k "$k" 'if (type == "object" and has($k) and .[$k] != null) then (.[$k] | tostring) else empty end' "$CONFIG_PATH" 2>/dev/null)
        [[ -n $v ]] && printf -v "CFG_$k" '%s' "$v"
    done
    [[ $CFG_memoryMB =~ ^[0-9]+$ ]] || CFG_memoryMB=2048
    [[ $CFG_sharedGameDir == true ]] || CFG_sharedGameDir=false
    [[ -n $CFG_javaPath ]] && CFG_javaPath=$(to_posix "$CFG_javaPath")
}

save_config() {
    mkdir -p "$ROOT"
    MSYS2_ARG_CONV_EXCL='*' jqx -n \
        --arg version "$CFG_version" --arg listMode "$CFG_listMode" --arg username "$CFG_username" \
        --argjson memoryMB "$CFG_memoryMB" --arg javaPath "$CFG_javaPath" --arg server "$CFG_server" \
        --arg extraJvm "$CFG_extraJvm" --argjson shared "$CFG_sharedGameDir" \
        '{version:$version, listMode:$listMode, username:$username, memoryMB:$memoryMB, javaPath:$javaPath, server:$server, extraJvm:$extraJvm, sharedGameDir:$shared}' \
        > "$CONFIG_PATH"
}

game_dir_of() {   # sets GAMEDIR (posix) and GAMEDIRM (mixed)
    if [[ $CFG_sharedGameDir == true ]]; then
        GAMEDIR=$ROOT/game; GAMEDIRM=$ROOTM/game
    else
        GAMEDIR=$INST_ROOT/$1; GAMEDIRM=$ROOTM/instances/$1
    fi
}

# ------------------------------------------------------------------ TUI primitives
wait_key() {
    printf '\n%s%s%s\n' "$C_DGRAY" "${1:-Press any key to continue...}" "$C_RST"
    IFS= read -rsn1 _
}

write_header() {
    clear_screen
    printf '%s%s%s\n' "$C_DCYAN" "$HLINE" "$C_RST"
    printf '%s| %-56s |%s\n' "$C_CYAN" 'Minecraft Launcher (offline)' "$C_RST"
    printf '%s| %-56s |%s\n' "$C_DGRAY" "$1" "$C_RST"
    printf '%s%s%s\n\n' "$C_DCYAN" "$HLINE" "$C_RST"
}

# Sets KEY to: Up Down PgUp PgDn Home End Enter Esc EOF or Other
read_key() {
    local k rest c seq=''
    IFS= read -rsn1 k || { KEY=EOF; return; }
    if [[ $k == $'\e' ]]; then
        if IFS= read -rsn1 -t 0.05 rest && [[ $rest == '[' || $rest == 'O' ]]; then
            while IFS= read -rsn1 -t 0.05 c; do
                seq+=$c
                [[ $c == [A-Za-z~] ]] && break
            done
            case $seq in
                A) KEY=Up ;;  B) KEY=Down ;;
                5~) KEY=PgUp ;; 6~) KEY=PgDn ;;
                H|1~|7~) KEY=Home ;; F|4~|8~) KEY=End ;;
                *) KEY=Other ;;
            esac
        else
            KEY=Esc
        fi
    elif [[ -z $k || $k == $'\r' ]]; then KEY=Enter
    else
        case $k in
            w|W) KEY=Up ;;
            s|S) KEY=Down ;;
            *)   KEY=Other ;;
        esac
    fi
}

# Items in MENU_ITEMS, header lines in MENU_INFO. Result in MENU_SEL (-1 on Esc).
show_menu() {
    local title=$1 sel=${2:-0} top=0 rows n=${#MENU_ITEMS[@]} i end t l scroll
    cursor_visible 0
    while :; do
        rows=$(( $(window_rows) - (14 + ${#MENU_INFO[@]}) )); (( rows < 5 )) && rows=5
        (( sel < top )) && top=$sel
        (( sel >= top + rows )) && top=$(( sel - rows + 1 ))
        write_header "$title"
        for l in "${MENU_INFO[@]}"; do printf '%s  %s%s\n' "$C_GRAY" "$l" "$C_RST"; done
        (( ${#MENU_INFO[@]} > 0 )) && printf '\n'
        scroll=0; (( n > rows )) && scroll=1
        (( scroll )) && printf '%s    ... %d more above%s\n' "$C_DGRAY" "$top" "$C_RST"
        end=$(( top + rows )); (( end > n )) && end=$n
        for (( i = top; i < end; i++ )); do
            t=${MENU_ITEMS[i]}
            (( ${#t} > 52 )) && t="${t:0:49}..."
            if (( i == sel )); then printf '%s  > %-52s%s\n' "$C_HL" "$t" "$C_RST"
            else printf '    %s\n' "$t"; fi
        done
        (( scroll )) && printf '%s    ... %d more below%s\n' "$C_DGRAY" "$(( n - end ))" "$C_RST"
        printf '\n%s  Up/Down: move   Enter: select   Esc: back%s\n' "$C_DGRAY" "$C_RST"
        read_key
        case $KEY in
            Up)    sel=$(( (sel - 1 + n) % n )) ;;
            Down)  sel=$(( (sel + 1) % n )) ;;
            PgUp)  sel=$(( sel - rows )); (( sel < 0 )) && sel=0 ;;
            PgDn)  sel=$(( sel + rows )); (( sel > n - 1 )) && sel=$(( n - 1 )) ;;
            Home)  sel=0 ;;
            End)   sel=$(( n - 1 )) ;;
            Enter) MENU_SEL=$sel; return ;;
            Esc|EOF) MENU_SEL=-1; return ;;
        esac
    done
}

# Result in INPUT (current value if the user just presses Enter)
read_input() {
    local label=$1 current=$2 suffix='' v
    [[ -n $current ]] && suffix=" [$current]"
    cursor_visible 1
    IFS= read -r -p "$label$suffix: " v
    cursor_visible 0
    v="${v#"${v%%[![:space:]]*}"}"; v="${v%"${v##*[![:space:]]}"}"
    if [[ -z $v ]]; then INPUT=$current; else INPUT=$v; fi
}

write_bar() {   # label done total
    local label=$1 fin=$2 total=$3 w=30 pct fill h b
    if (( total > 0 )); then pct=$(( 100 * fin / total )); else pct=100; fi
    fill=$(( w * pct / 100 ))
    printf -v h '%*s' "$fill" ''; h=${h// /#}
    printf -v b '%*s' "$(( w - fill ))" ''; b=${b// /-}
    printf '\r  %-12s [%s%s] %3d%% (%d/%d)    ' "$label" "$h" "$b" "$pct" "$fin" "$total"
}

# ------------------------------------------------------------------ download / zip helpers
download_verified() {   # url dest [sha1]
    local url=$1 dest=$2 sha1=${3:-} tmp actual
    mkdir -p "${dest%/*}"
    if [[ -f $dest ]]; then
        if [[ -z $sha1 ]]; then
            [[ -s $dest ]] && return 0
        else
            actual=$(sha1sum -- "$dest" | cut -d' ' -f1)
            [[ $actual == "${sha1,,}" ]] && return 0
        fi
    fi
    tmp="$dest.download"
    rm -f "$tmp"
    curlx -fsSL --retry 3 -o "$tmp" "$url" || { rm -f "$tmp"; die "Download failed: $url"; }
    if [[ -n $sha1 ]]; then
        actual=$(sha1sum -- "$tmp" | cut -d' ' -f1)
        if [[ $actual != "${sha1,,}" ]]; then rm -f "$tmp"; die "SHA-1 mismatch for $dest"; fi
    fi
    mv -f "$tmp" "$dest"
}

extract_dlls() {   # jar destdir
    if command -v unzip >/dev/null 2>&1; then
        unzip -joq "$1" '*.dll' -d "$2" >/dev/null 2>&1
        return 0
    fi
    powershell.exe -NoProfile -ExecutionPolicy Bypass -File "$(to_mixed "$TOOLS/zip.ps1")" \
        -Jar "$(to_mixed "$1")" -Dest "$(to_mixed "$2")" -Mode extract >/dev/null
}

jar_has_main_class() {
    if command -v unzip >/dev/null 2>&1; then
        unzip -l "$1" 'net/minecraft/client/main/Main.class' >/dev/null 2>&1
    else
        powershell.exe -NoProfile -ExecutionPolicy Bypass -File "$(to_mixed "$TOOLS/zip.ps1")" \
            -Jar "$(to_mixed "$1")" -Dest '' -Mode check >/dev/null 2>&1
    fi
}

# ------------------------------------------------------------------ version metadata
get_manifest() {   # [refresh]
    if [[ $MANIFEST_OK == 1 && ${1:-} != refresh ]]; then return 0; fi
    mkdir -p "$ROOT"
    MANIFEST_ERR=''
    local opt url
    for url in "$MANIFEST_URL" "$MANIFEST_URL2"; do
        for opt in '' '--ssl-no-revoke'; do
            if MANIFEST_ERR=$(curlx -fsSL $opt --max-time 30 -o "$MANIFEST_FILE.tmp" "$url" 2>&1); then
                mv -f "$MANIFEST_FILE.tmp" "$MANIFEST_FILE"
                MANIFEST_ERR=''
                break 2
            fi
        done
    done
    rm -f "$MANIFEST_FILE.tmp"
    MANIFEST_OK=0
    if [[ -f $MANIFEST_FILE ]]; then
        if jqx -e '.versions | length > 0' "$MANIFEST_FILE" >/dev/null 2>&1; then MANIFEST_OK=1
        else MANIFEST_ERR="${MANIFEST_ERR:+$MANIFEST_ERR; }cached manifest at $MANIFEST_FILE is not valid (delete it); jq=$JQ"; fi
    fi
}

# Writes the merged version JSON (inheritsFrom resolved, _jarId added) to $2. Returns 1 + ERRMSG on failure.
resolve_json() {   # id outfile
    local id=$1 out=$2 p="$VERSIONS/$1/$1.json" parent tmp
    if [[ ! -f $p ]]; then ERRMSG="Version JSON for '$id' not found. Install it first."; return 1; fi
    parent=$(jqx -r '.inheritsFrom // empty' "$p") || { ERRMSG="Cannot parse $p"; return 1; }
    if [[ -n $parent ]]; then
        tmp=$(mktemp "$TMPD/res.XXXXXX")
        resolve_json "$parent" "$tmp" || return 1
        jqx -s -f "$JQDIR/merge.jq" "$tmp" "$p" > "$out" || { ERRMSG="Cannot merge $id with $parent"; return 1; }
        rm -f "$tmp"
    else
        jqx --arg jar "$id" -f "$JQDIR/base.jq" "$p" > "$out" || { ERRMSG="Cannot parse $p"; return 1; }
    fi
}

jar_id_of() {   # id -> echoes the id whose jar is used (last ancestor)
    local id=$1 n=0 parent
    while :; do
        [[ -f $VERSIONS/$id/$id.json ]] || return 1
        parent=$(jqx -r '.inheritsFrom // empty' "$VERSIONS/$id/$id.json" 2>/dev/null) || return 1
        [[ -z $parent ]] && break
        id=$parent; n=$(( n + 1 )); (( n > 10 )) && return 1
    done
    printf '%s' "$id"
}

local_version_ids() {
    local d n
    [[ -d $VERSIONS ]] || return 0
    for d in "$VERSIONS"/*/; do
        d=${d%/}; n=${d##*/}
        [[ -f $d/$n.json ]] && printf '%s\n' "$n"
    done
}

test_installed() {
    local id=$1 jid
    [[ -f $VERSIONS/$id/$id.json ]] || return 1
    jid=$(jar_id_of "$id") || return 1
    [[ -f $VERSIONS/$jid/$jid.jar ]] || return 1
    [[ -d $VERSIONS/$id/natives ]] || return 1
    compgen -G "$VERSIONS/$id/natives/*.dll" >/dev/null && return 0
    # No extracted DLLs: fine for 1.19+, where LWJGL loads natives from the classpath JARs.
    # Only versions with classifier-style natives (<= 1.18) need them in the natives folder.
    local tmp; tmp=$(mktemp "$TMPD/ti.XXXXXX")
    resolve_json "$id" "$tmp" || { rm -f "$tmp"; return 1; }
    jqx -e '[.libraries[] | select(.natives.windows != null)] | length == 0' "$tmp" >/dev/null 2>&1
    local rc=$?
    rm -f "$tmp"
    return $rc
}

# Fills ENT_ID / ENT_TYPE / ENT_INST (1/0)
build_entries() {   # mode filter
    local mode=$1 lf=${2,,} id type
    ENT_ID=(); ENT_TYPE=(); ENT_INST=()
    local -A inman=() islocal=()
    local -a locals=()
    mapfile -t locals < <(local_version_ids)
    for id in "${locals[@]}"; do islocal[$id]=1; done
    if (( MANIFEST_OK )); then
        while IFS= read -r id; do inman[$id]=1; done < <(jqx -r '.versions[].id' "$MANIFEST_FILE")
    fi
    for id in "${locals[@]}"; do
        [[ -n ${inman[$id]:-} ]] && continue
        if [[ -z $lf || ${id,,} == *"$lf"* ]]; then
            ENT_ID+=("$id"); ENT_TYPE+=(custom)
            if test_installed "$id"; then ENT_INST+=(1); else ENT_INST+=(0); fi
        fi
    done
    (( MANIFEST_OK )) || return 0
    while IFS='|' read -r id type; do
        [[ $mode == release && $type != release ]] && continue
        [[ $mode == installed && -z ${islocal[$id]:-} ]] && continue
        [[ -n $lf && ${id,,} != *"$lf"* ]] && continue
        ENT_ID+=("$id"); ENT_TYPE+=("$type")
        if [[ -n ${islocal[$id]:-} ]] && test_installed "$id"; then ENT_INST+=(1); else ENT_INST+=(0); fi
    done < <(jqx -r '.versions[] | select(.type == "release" or .type == "snapshot") | select(.releaseTime >= "2013-06-25") | [.id, .type] | join("|")' "$MANIFEST_FILE")
}

# ------------------------------------------------------------------ libraries
library_work() {   # resolved.json -> url|rel|sha1|native|oncp lines
    jqx -r -f "$JQDIR/work.jq" --arg arch "$ARCH" --arg osver "$OSVER" --arg bits "$BITS" "$1"
}

build_classpath() {   # resolved.json jarid -> CLASSPATH (mixed paths, ';' separated)
    local url rel sha native oncp parts=''
    local -A seen=()
    while IFS='|' read -r url rel sha native oncp; do
        [[ $oncp == 1 ]] || continue
        [[ -n ${seen[$rel]:-} ]] && continue
        seen[$rel]=1
        parts+="$LIB_ROOTM/$rel;"
    done < <(library_work "$1")
    CLASSPATH="$parts$ROOTM/versions/$2/$2.jar"
}

# ------------------------------------------------------------------ assets
count_objects() { find "$AOBJ_ROOT" -type f 2>/dev/null | wc -l; }

missing_assets() {   # idxfile -> TODO_H (hashes whose file is absent or has the wrong size)
    TODO_H=()
    local -A have=() seen=()
    local p s h sz
    while IFS='|' read -r p s; do have[$p]=$s; done < <(find "$AOBJ_ROOT" -type f -printf '%P|%s\n' 2>/dev/null)
    while IFS='|' read -r h sz; do
        [[ -n ${seen[$h]:-} ]] && continue
        seen[$h]=1
        [[ ${have["${h:0:2}/$h"]:-} == "$sz" ]] || TODO_H+=("$h")
    done < <(jqx -r '.objects[] | [.hash, (.size | tostring)] | join("|")' "$1")
}

install_assets() {   # resolved.json gamedir
    local resolved=$1 gamedir=$2 ai_id ai_url ai_sha aif attempt total base now fin cpid cfg h
    ai_id=$(jqx -r '.assetIndex.id // empty' "$resolved")
    ai_url=$(jqx -r '.assetIndex.url // empty' "$resolved")
    ai_sha=$(jqx -r '.assetIndex.sha1 // empty' "$resolved")
    [[ -n $ai_id ]] || die 'Version has no asset index.'
    aif=$AIDX_ROOT/$ai_id.json
    download_verified "$ai_url" "$aif" "$ai_sha"

    cfg=$TMPD/assets.curl
    for attempt in 1 2 3; do
        missing_assets "$aif"
        total=${#TODO_H[@]}
        (( total == 0 )) && break
        printf '  %d assets to download (8 parallel connections).\n' "$total"
        {
            printf 'parallel\nparallel-max = 8\ncreate-dirs\nfail\nsilent\nretry = 3\n'
            for h in "${TODO_H[@]}"; do
                printf 'url = "https://resources.download.minecraft.net/%s/%s"\n' "${h:0:2}" "$h"
                printf 'output = "%s/%s/%s"\n' "$AOBJ_ROOTM" "${h:0:2}" "$h"
            done
        } > "$cfg"
        base=$(count_objects)
        curlx -K "$cfg" &
        cpid=$!
        while kill -0 "$cpid" 2>/dev/null; do
            now=$(count_objects); fin=$(( now - base ))
            (( fin > total )) && fin=$total
            (( fin < 0 )) && fin=0
            write_bar Assets "$fin" "$total"
            sleep 1
        done
        wait "$cpid" 2>/dev/null
        write_bar Assets "$total" "$total"; printf '\n'
    done
    missing_assets "$aif"
    (( ${#TODO_H[@]} == 0 )) || die "${#TODO_H[@]} assets failed to download. Run Install / Repair again."

    # Very old versions (1.6 - 1.7.2) need real file names instead of hashed objects
    local virt mapres dest name src target
    virt=$(jqx -r '.virtual // false' "$aif")
    mapres=$(jqx -r '.map_to_resources // false' "$aif")
    if [[ $virt == true || $mapres == true ]]; then
        printf '  %sPreparing legacy asset layout...%s\n' "$C_CYAN" "$C_RST"
        if [[ $virt == true ]]; then dest=$ASSET_ROOT/virtual/$ai_id; else dest=$gamedir/resources; fi
        while IFS='|' read -r name h; do
            src=$AOBJ_ROOT/${h:0:2}/$h
            target=$dest/$name
            if [[ ! -f $target ]]; then mkdir -p "${target%/*}"; cp -- "$src" "$target"; fi
        done < <(jqx -r '.objects | to_entries[] | [.key, .value.hash] | join("|")' "$aif")
    fi
}

# ------------------------------------------------------------------ install
install_version() {
    local id=$1 jp="$VERSIONS/$1/$1.json" entry='' url sha1 parent resolved jurl jsha i=0 total line rel native oncp nat
    mkdir -p "$VERSIONS/$id" "$LIB_ROOT" "$AIDX_ROOT" "$AOBJ_ROOT"
    printf '  %s[%s] fetching metadata...%s\n' "$C_CYAN" "$id" "$C_RST"
    if (( MANIFEST_OK )); then
        entry=$(jqx -r --arg id "$id" '[.versions[] | select(.id == $id)][0] // empty | [.url, .sha1] | join("|")' "$MANIFEST_FILE")
    fi
    if [[ -n $entry ]]; then
        IFS='|' read -r url sha1 <<< "$entry"
        download_verified "$url" "$jp" "$sha1"
    elif [[ ! -f $jp ]]; then
        if (( MANIFEST_OK )); then die "Version '$id' is not in the manifest."; fi
        die "Could not load the version manifest: ${MANIFEST_ERR:-unknown error}"
    fi

    parent=$(jqx -r '.inheritsFrom // empty' "$jp")
    [[ -n $parent ]] && install_version "$parent"
    resolved=$(mktemp "$TMPD/inst.XXXXXX")
    resolve_json "$id" "$resolved" || die "$ERRMSG"

    jurl=$(jqx -r '.downloads.client.url // empty' "$jp")
    jsha=$(jqx -r '.downloads.client.sha1 // empty' "$jp")
    if [[ -n $jurl ]]; then
        printf '  %s[%s] downloading client JAR...%s\n' "$C_CYAN" "$id" "$C_RST"
        download_verified "$jurl" "$VERSIONS/$id/$id.jar" "$jsha"
        jar_has_main_class "$VERSIONS/$id/$id.jar" || die 'client JAR is missing Main.class.'
    fi

    nat=$VERSIONS/$id/natives
    rm -rf "$nat"; mkdir -p "$nat"
    local -a work=()
    mapfile -t work < <(library_work "$resolved")
    total=${#work[@]}
    printf '  %s[%s] downloading libraries and natives...%s\n' "$C_CYAN" "$id" "$C_RST"
    for line in "${work[@]}"; do
        i=$(( i + 1 ))
        IFS='|' read -r url rel sha1 native oncp <<< "$line"
        [[ -n $url ]] && download_verified "$url" "$LIB_ROOT/$rel" "$sha1"
        if [[ $native == 1 && -f $LIB_ROOT/$rel ]]; then extract_dlls "$LIB_ROOT/$rel" "$nat"; fi
        write_bar Libraries "$i" "$total"
    done
    printf '\n'
    local ndll=0 f
    for f in "$nat"/*.dll; do [[ -f $f ]] && ndll=$(( ndll + 1 )); done
    printf '  [%s] natives: %d DLL extracted\n' "$id" "$ndll"

    game_dir_of "$id"
    mkdir -p "$GAMEDIR"
    printf '  %s[%s] checking assets...%s\n' "$C_CYAN" "$id" "$C_RST"
    install_assets "$resolved" "$GAMEDIR"
}

invoke_install() {
    write_header "Install / Repair: $CFG_version"
    get_manifest refresh
    install_version "$CFG_version"
    printf '\n  %sInstall complete.%s\n' "$C_GREEN" "$C_RST"
}

# ------------------------------------------------------------------ java
get_java_major() {   # path -> JMAJOR
    local out re1='version "1\.([0-9]+)' re2='version "([0-9]+)'
    JMAJOR=0
    out=$("$1" -version 2>&1) || true
    if   [[ $out =~ $re1 ]]; then JMAJOR=${BASH_REMATCH[1]}
    elif [[ $out =~ $re2 ]]; then JMAJOR=${BASH_REMATCH[1]}
    fi
}

find_java_candidates() {
    [[ -n $JAVA_SCANNED ]] && return 0
    local -a paths=(); local f base pf pf86 appdata local_app key p
    if [[ -n $CFG_javaPath && -f $CFG_javaPath ]]; then
        paths+=("$CFG_javaPath")
    else
        [[ -n ${JAVA_HOME:-} ]] && paths+=("$(to_posix "$JAVA_HOME")/bin/java.exe")
        while IFS= read -r f; do [[ -n $f ]] && paths+=("$f"); done < <(type -ap java 2>/dev/null)
        pf=$(to_posix "${PROGRAMFILES:-C:\\Program Files}")
        pf86=$(to_posix "${PROGRAMFILES:-C:\\Program Files} (x86)")
        appdata=$(to_posix "${APPDATA:-}")
        local_app=$(to_posix "${LOCALAPPDATA:-}")
        for base in "$pf/Java" "$pf86/Java" "$pf/Eclipse Adoptium" "$pf/Microsoft" "$pf/Zulu" \
                    "$pf/Amazon Corretto" "$pf/BellSoft" "$pf/Semeru"; do
            for f in "$base"/*/bin/java.exe; do [[ -f $f ]] && paths+=("$f"); done
        done
        for f in "$ROOT"/java/bin/java.exe "$ROOT"/java/*/bin/java.exe; do [[ -f $f ]] && paths+=("$f"); done
        for f in "$appdata"/.minecraft/runtime/*/*/*/bin/java.exe \
                 "$local_app"/Packages/Microsoft.4297127D64EC6_8wekyb3d8bbwe/LocalCache/Local/runtime/*/*/*/bin/java.exe; do
            [[ -f $f ]] && paths+=("$f")
        done
    fi
    JAVA_PATHS=(); JAVA_MAJORS=()
    local -A seen=()
    for p in "${paths[@]}"; do
        [[ -z $p ]] && continue
        key=${p,,}; key=${key%.exe}
        [[ -n ${seen[$key]:-} ]] && continue
        seen[$key]=1
        get_java_major "$p"
        if (( JMAJOR > 0 )); then JAVA_PATHS+=("$p"); JAVA_MAJORS+=("$JMAJOR"); fi
    done
    JAVA_SCANNED=1
}

select_java() {   # required major -> SEL_JAVA, SEL_JAVA_MAJOR
    local req=$1 i best=-1 bm=9999 m
    find_java_candidates
    if [[ -n $CFG_javaPath && -f $CFG_javaPath ]]; then
        (( ${#JAVA_PATHS[@]} > 0 )) || return 1
        SEL_JAVA=${JAVA_PATHS[0]}; SEL_JAVA_MAJOR=${JAVA_MAJORS[0]}; return 0
    fi
    for i in "${!JAVA_PATHS[@]}"; do
        m=${JAVA_MAJORS[i]}
        if (( m >= req && m < bm )); then bm=$m; best=$i; fi
    done
    (( best >= 0 )) || return 1
    SEL_JAVA=${JAVA_PATHS[best]}; SEL_JAVA_MAJOR=${JAVA_MAJORS[best]}
}

# ------------------------------------------------------------------ launch
offline_uuid() {   # name -> echoes 32 hex chars (same algorithm as the PowerShell launcher)
    local h b6 b8
    h=$(printf 'OfflinePlayer:%s' "$1" | md5sum | cut -c1-32)
    b6=$(( (0x${h:12:2} & 0x0f) | 0x30 ))
    b8=$(( (0x${h:16:2} & 0x3f) | 0x80 ))
    printf '%s%02x%s%02x%s' "${h:0:12}" "$b6" "${h:14:2}" "$b8" "${h:18}"
}

expand_template() {   # string -> EXPANDED using associative array MAP
    local s=$1 k
    for k in "${!MAP[@]}"; do s=${s//"\${$k}"/"${MAP[$k]}"}; done
    EXPANDED=$s
}

# Reads raw args (one per line) from stdin, expands placeholders into CLEAN.
# Drops empty tokens and flags whose value could not be resolved.
expand_stream() {
    CLEAN=()
    local t
    while IFS= read -r t; do
        expand_template "$t"; t=$EXPANDED
        [[ -z $t ]] && continue
        if [[ $t == *'${'* ]]; then
            if (( ${#CLEAN[@]} > 0 )) && [[ ${CLEAN[-1]} == --* ]]; then unset 'CLEAN[-1]'; fi
            continue
        fi
        CLEAN+=("$t")
    done
}

start_game() {
    local id=$CFG_version resolved req=8 v jid name aif ai_id game_assets virt qp=0 hp
    write_header "Launching $id"
    test_installed "$id" || die "Version $id is not installed. Choose Install / Repair first."
    resolved=$(mktemp "$TMPD/launch.XXXXXX")
    resolve_json "$id" "$resolved" || die "$ERRMSG"

    v=$(jqx -r '.javaVersion.majorVersion // empty' "$resolved")
    [[ -n $v ]] && req=$v
    select_java "$req" || die "No suitable Java found. $id needs Java $req or newer. Install a JDK/JRE $req (e.g. from adoptium.net) or set the Java path in Settings."
    if (( SEL_JAVA_MAJOR < req )); then
        printf '  %sWarning: Java %s is older than the required %s.%s\n' "$C_YELLOW" "$SEL_JAVA_MAJOR" "$req" "$C_RST"
    elif (( req <= 8 && SEL_JAVA_MAJOR > 8 )); then
        printf '  %sNote: using Java %s; this version targets Java 8 and may be less stable.%s\n' "$C_YELLOW" "$SEL_JAVA_MAJOR" "$C_RST"
    fi

    game_dir_of "$id"
    mkdir -p "$GAMEDIR"
    jid=$(jqx -r '._jarId' "$resolved")
    build_classpath "$resolved" "$jid"
    name=$CFG_username
    [[ -n $CFG_server ]] && qp=1

    ai_id=$(jqx -r '.assetIndex.id // empty' "$resolved")
    game_assets=$ASSET_ROOTM
    aif=$AIDX_ROOT/$ai_id.json
    if [[ -f $aif ]]; then
        virt=$(jqx -r '.virtual // false' "$aif")
        [[ $virt == true ]] && game_assets=$ASSET_ROOTM/virtual/$ai_id
    fi

    declare -A MAP=(
        [auth_player_name]=$name [version_name]=$id [game_directory]=$GAMEDIRM
        [assets_root]=$ASSET_ROOTM [assets_index_name]=$ai_id [game_assets]=$game_assets
        [auth_uuid]=$(offline_uuid "$name") [auth_access_token]=0 [auth_session]=0
        [auth_xuid]=0 [clientid]=0 [user_properties]='{}' [user_type]=legacy
        [version_type]=$(jqx -r '.type // "release"' "$resolved")
        [natives_directory]=$ROOTM/versions/$id/natives [launcher_name]=MCLauncher-TUI [launcher_version]=2.0
        [classpath]=$CLASSPATH [classpath_separator]=';' [library_directory]=$LIB_ROOTM
        [quickPlayMultiplayer]=$CFG_server
    )

    local -a jvm=() gamel=() w=()
    local a
    jvm+=(-Xms512M "-Xmx${CFG_memoryMB}M")
    if [[ -n $CFG_extraJvm ]]; then read -ra w <<< "$CFG_extraJvm"; jvm+=("${w[@]}"); fi

    if [[ $(jqx -r 'has("arguments")' "$resolved") == true ]]; then
        local sel_name
        for sel_name in jvm game; do
            jqx -r -f "$JQDIR/args.jq" --arg sel "$sel_name" --arg qp "$qp" \
                --arg arch "$ARCH" --arg osver "$OSVER" "$resolved" > "$TMPD/args.$sel_name" \
                || die "Cannot read $sel_name arguments for $id"
            expand_stream < "$TMPD/args.$sel_name"
            if [[ $sel_name == jvm ]]; then jvm+=("${CLEAN[@]}"); else gamel=("${CLEAN[@]}"); fi
        done
    else
        # Legacy format (before 1.13): LWJGL 2, classpath and natives passed manually
        jvm+=("-Djava.library.path=$ROOTM/versions/$id/natives" "-Dorg.lwjgl.librarypath=$ROOTM/versions/$id/natives" -cp "$CLASSPATH")
        read -ra w <<< "$(jqx -r '.minecraftArguments // ""' "$resolved")"
        expand_stream < <(printf '%s\n' "${w[@]}")
        gamel=("${CLEAN[@]}")
    fi

    if [[ -n $CFG_server ]]; then
        local has_qp=0
        for a in "${gamel[@]}"; do [[ $a == --quickPlayMultiplayer ]] && has_qp=1; done
        if (( ! has_qp )); then
            gamel+=(--server "${CFG_server%%:*}")
            [[ $CFG_server == *:* ]] && gamel+=(--port "${CFG_server#*:}")
        fi
    fi
    jvm+=("$(jqx -r '.mainClass' "$resolved")")

    printf '  Profile : %s (offline)\n' "$name"
    printf '  Java    : %s (Java %s, needs %s+)\n' "$SEL_JAVA" "$SEL_JAVA_MAJOR" "$req"
    printf '  Memory  : %s MB\n' "$CFG_memoryMB"
    printf '  GameDir : %s\n' "$GAMEDIRM"
    [[ -n $CFG_server ]] && printf '  Server  : %s\n' "$CFG_server"
    printf '\n%s  Game output follows. Close the game to return.%s\n\n' "$C_DGRAY" "$C_RST"
    cursor_visible 1
    cd "$GAMEDIR" || die "Cannot enter $GAMEDIR"
    MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL='*' "$SEL_JAVA" "${jvm[@]}" "${gamel[@]}"
    local rc=$?
    if (( rc == 0 )); then hp=$C_GREEN; else hp=$C_YELLOW; fi
    printf '\n  %sGame exited with code %d.%s\n' "$hp" "$rc" "$C_RST"
}

# ------------------------------------------------------------------ screens
select_version() {
    write_header 'Select version'
    printf '%s  Fetching version list...%s\n' "$C_DGRAY" "$C_RST"
    get_manifest refresh
    local filter='' sel=2 idx e label modes=(release snapshot installed) i
    local -A labels=([release]='Releases' [snapshot]='Releases + snapshots' [installed]='Installed only')
    while :; do
        build_entries "$CFG_listMode" "$filter"
        MENU_ITEMS=()
        MENU_ITEMS+=("Filter : ${filter:-(none)}")
        MENU_ITEMS+=("Show   : ${labels[$CFG_listMode]}  (Enter to change)")
        for i in "${!ENT_ID[@]}"; do
            e=''; [[ ${ENT_INST[i]} == 1 ]] && e='[installed]'
            MENU_ITEMS+=("$(printf '%-22s %-9s %s' "${ENT_ID[i]}" "${ENT_TYPE[i]}" "$e")")
        done
        MENU_INFO=("Current: $CFG_version")
        (( MANIFEST_OK )) || MENU_INFO+=('Offline: only locally installed versions are listed.')
        show_menu 'Select version' "$sel"
        idx=$MENU_SEL
        if (( idx == -1 )); then return; fi
        if (( idx == 0 )); then
            read_input 'Filter text (type - to clear)' "$filter"
            if [[ $INPUT == - ]]; then filter=''; else filter=$INPUT; fi
            sel=0
        elif (( idx == 1 )); then
            for i in "${!modes[@]}"; do [[ ${modes[i]} == "$CFG_listMode" ]] && break; done
            CFG_listMode=${modes[(i + 1) % ${#modes[@]}]}
            save_config
            sel=1
        else
            CFG_version=${ENT_ID[idx-2]}
            save_config
            return
        fi
    done
}

show_status() {
    write_header 'Status'
    local id=$CFG_version st='not installed' j i
    test_installed "$id" && st=installed
    game_dir_of "$id"
    printf '  Root         : %s\n' "$ROOTM"
    printf '  Version      : %s (%s)\n' "$id" "$st"
    printf '  Game dir     : %s\n' "$GAMEDIRM"
    printf '  Profile      : %s   UUID %s\n' "$CFG_username" "$(offline_uuid "$CFG_username")"
    printf '  Memory       : %s MB\n' "$CFG_memoryMB"
    printf '  Server       : %s\n' "${CFG_server:-(none)}"
    printf '\n'
    local -a inst=()
    while IFS= read -r j; do [[ -n $j ]] && test_installed "$j" && inst+=("$j"); done < <(local_version_ids)
    if (( ${#inst[@]} )); then local IFS=,; printf '  Installed    : %s\n' "${inst[*]}"; unset IFS
    else printf '  Installed    : (none)\n'; fi
    printf '\n  Java installations found:\n'
    find_java_candidates
    if (( ${#JAVA_PATHS[@]} == 0 )); then printf '    %s(none)%s\n' "$C_YELLOW" "$C_RST"; fi
    for i in "${!JAVA_PATHS[@]}"; do printf '    Java %-3s %s\n' "${JAVA_MAJORS[i]}" "${JAVA_PATHS[i]}"; done
    wait_key
}

show_settings() {
    local sel=0 n v shared_txt
    while :; do
        if [[ $CFG_sharedGameDir == true ]]; then shared_txt='yes (one world folder for all versions)'
        else shared_txt='no (separate folder per version)'; fi
        MENU_ITEMS=(
            "Profile name     : $CFG_username"
            "Memory (MB)      : $CFG_memoryMB"
            "Auto-join server : ${CFG_server:-(none)}"
            "Java path        : ${CFG_javaPath:-(auto-detect per version)}"
            "Extra JVM args   : ${CFG_extraJvm:-(none)}"
            "Shared game dir  : $shared_txt"
            'Back'
        )
        MENU_INFO=("Root: $ROOTM")
        show_menu 'Settings' "$sel"
        sel=$MENU_SEL
        if (( sel == -1 || sel == 6 )); then return; fi
        printf '\n'
        case $sel in
            0)  read_input 'Profile name (3-16 letters/digits/_)' "$CFG_username"
                if [[ $INPUT =~ ^[A-Za-z0-9_]{3,16}$ ]]; then CFG_username=$INPUT
                else printf '  %sInvalid name.%s\n' "$C_RED" "$C_RST"; wait_key; fi ;;
            1)  read_input 'Max memory in MB (512-32768)' "$CFG_memoryMB"
                n=$INPUT
                if [[ $n =~ ^[0-9]+$ ]] && (( n >= 512 && n <= 32768 )); then CFG_memoryMB=$n
                else printf '  %sInvalid value.%s\n' "$C_RED" "$C_RST"; wait_key; fi ;;
            2)  read_input 'Server host[:port] (type - to clear)' "$CFG_server"
                if [[ $INPUT == - ]]; then CFG_server=''; else CFG_server=$INPUT; fi ;;
            3)  read_input 'Full path to java.exe (type - for auto)' "$CFG_javaPath"
                v=$INPUT
                if [[ $v == - ]]; then CFG_javaPath=''
                else
                    v=$(to_posix "$v")
                    if [[ -f $v ]]; then CFG_javaPath=$v
                    else printf '  %sFile not found.%s\n' "$C_RED" "$C_RST"; wait_key; fi
                fi
                JAVA_SCANNED='' ;;
            4)  read_input 'Extra JVM args (type - to clear)' "$CFG_extraJvm"
                if [[ $INPUT == - ]]; then CFG_extraJvm=''; else CFG_extraJvm=$INPUT; fi ;;
            5)  if [[ $CFG_sharedGameDir == true ]]; then CFG_sharedGameDir=false; else CFG_sharedGameDir=true; fi ;;
        esac
        save_config
    done
}

# Runs an action in a subshell so die() only aborts the action, then waits for a key
invoke_safely() {
    ( "$@" )
    wait_key
}

# ------------------------------------------------------------------ main loop
ensure_jq || exit 1
write_jq_files
load_config

if [[ -n $HEADLESS_VERSION ]]; then
    CFG_version=$HEADLESS_VERSION
    save_config
    ( invoke_install )
    rc=$?
    (( rc == 0 )) || exit "$rc"
    test_installed "$CFG_version" || { echo "Install verification failed for $CFG_version"; exit 1; }
    echo "Installed $CFG_version into $ROOTM"
    exit 0
fi

sel=0
quit=0
while (( ! quit )); do
    id=$CFG_version
    state='NOT installed - choose Install / Repair'
    test_installed "$id" && state=installed
    MENU_INFO=("Version : $id ($state)" "Profile : $CFG_username (offline)" "Memory  : $CFG_memoryMB MB")
    MENU_ITEMS=('Play' "Version: $id  (change)" 'Install / Repair selected version' 'Settings' 'Status' 'Quit')
    show_menu 'Main menu' "$sel"
    sel=$MENU_SEL
    case $sel in
        0)  invoke_safely start_game ;;
        1)  select_version ;;
        2)  invoke_safely invoke_install ;;
        3)  show_settings ;;
        4)  show_status ;;
        *)  quit=1 ;;
    esac
done
clear_screen
