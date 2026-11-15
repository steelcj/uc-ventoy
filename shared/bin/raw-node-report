#!/bin/sh
# raw-node-report: print the minimal raw node confirmation as YAML.
#
# Usage: raw-node-report [username]    (default username: initial)
# Run as root. Every value is measured on this node, not copied from input.
# Exit status: 0 if the user exists and can use sudo, 1 otherwise.
#
# Written without single quotes or backslashes so that it can be embedded
# in a Debian preseed late_command with printf, one argument per line.

set -u
set -f
LC_ALL=C
export LC_ALL

user="${1:-initial}"

# Print the source address the kernel would use to reach a destination.
# ip route get sends no packets. Documentation addresses are used as
# destinations so that no real host is involved.
src_of() {
    # Word splitting of the ip output is intended.
    # shellcheck disable=SC2046
    set -- $(ip "$1" route get "$2" 2>/dev/null)
    while [ "$#" -gt 1 ]; do
        if [ "$1" = "src" ]; then
            echo "$2"
            return 0
        fi
        shift
    done
    return 0
}

host="$(hostname)"
timestamp="$(date --iso-8601=seconds)"
ip4="$(src_of -4 192.0.2.1)"
ip6="$(src_of -6 2001:db8::1)"

if id -u "$user" >/dev/null 2>&1; then
    user_ok=true
else
    user_ok=false
fi

has_sudo=false
if [ "$user_ok" = true ]; then
    if [ "$(id -u)" -eq 0 ] && command -v sudo >/dev/null 2>&1; then
        if sudo -n -l -U "$user" 2>/dev/null | grep -q "may run the following commands"; then
            has_sudo=true
        fi
    else
        for group in $(id -nG "$user"); do
            case "$group" in
                sudo|wheel) has_sudo=true ;;
            esac
        done
    fi
fi

if [ "$user_ok" = true ]; then
    user_value="$user"
else
    user_value=""
fi

echo "$host:"
echo "  timestamp: $timestamp"
echo "  ip: $ip4"
echo "  ip6: $ip6"
echo "  user: $user_value"
echo "  sudo: $has_sudo"

if [ "$user_ok" = true ] && [ "$has_sudo" = true ]; then
    exit 0
fi
exit 1
