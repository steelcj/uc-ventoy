# troubleshooting



That confirms it. The hostname has no suffix and the report taken now shows `192.168.1.109`, so the script works. It just ran before DHCP had finished. The wireless card wasn't involved.

**About `ip6` being empty:** that's probably accurate rather than a bug. Many home networks don't hand out IPv6 addresses. Check with:

```sh
ip -6 route show default
```

If that prints nothing, the laptop has no IPv6 route, and an empty `ip6` is the correct answer.

**The fix:** in `shared/bin/raw-node-firstboot`, insert this just before the `dev="$(default_dev -4)"` line:

```sh
# Wait up to 60 seconds for a default route, so the hostname suffix and the
# addresses in the report are not taken before DHCP has finished.
wait_for_route() {
    tries=0
    while [ "$tries" -lt 60 ]; do
        if [ -n "$(ip -4 route show default 2>/dev/null)" ] || [ -n "$(ip -6 route show default 2>/dev/null)" ]; then
            return 0
        fi
        sleep 1
        tries=$((tries + 1))
    done
    return 1
}
wait_for_route || echo "raw-node-firstboot: no default route after 60 seconds" >&2
```

I tested it with a 3-second limit. With a route present it continues immediately. With none, it gives up after the limit, logs one line and carries on, so a node with no network still boots and writes its report. ShellCheck is clean, and it contains no quotes or backslashes, so `bin/build` will still embed it.

**On the test laptop**, you can apply the fix by hand and re-run:

```sh
sudo edit /usr/local/sbin/raw-node-firstboot   # paste the block above
sudo rm /var/lib/raw-node/confirmation.yml
sudo systemctl restart raw-node-firstboot
hostnamectl --static                          # should now be deb13-vt-xxxx
```

For future nodes, make the same change in the repository and run `bin/build --stick` again. Want me to send the updated `raw-node-firstboot` file instead?