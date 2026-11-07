bolt provider snap (Ubuntu Core Desktop, Phase 4)
==================================================

The fat base shipped bolt (thunderbolt device management); the thin
desktop variant bakes only the polkit action
(org.freedesktop.bolt.policy). This snap restores the daemon side:
bolt 0.9.10 staged from the resolute archive (byte-identical to the
fat base's binaries), as a strictly-confined system dbus-activated
daemon (org.freedesktop.bolt).

Design (follows the fprintd/accountsservice provider-snap pattern):

- base: plain core26 — the daemon execs no host helpers and links
  only base libraries; no localdev_swap_base dance.
- slots: `bolt-dbus` (builtin dbus interface, system bus,
  org.freedesktop.bolt — activation + name ownership) and `bolt`
  (custom fork interface carrying the daemon recipe: peer-less dbus
  receive rules, /var/lib/boltd store, /run/boltd power state,
  thunderbolt sysfs access, udev/netlink enumeration).
- plugs: `polkit` (action-prefix org.freedesktop.bolt; the action XML
  ships via a `policy` part at meta/polkit — snapd REQUIRES it for a
  polkit plug; it duplicates the variant's baked copy until a variant
  pass removes that — polkitd tolerates the duplicate) and
  `bolt-client` for the bundled boltctl CLI.
- apps: `boltd` daemon (daemon: dbus, daemon-scope: system,
  activates-on bolt-dbus) + `boltctl`.
- layout: /var/lib/boltd -> $SNAP_COMMON (state). /run/boltd cannot be
  layout-bound (bind sources are $SNAP/$SNAP_DATA/$SNAP_COMMON only);
  the daemon creates it itself as root and the slot policy grants
  /run/boltd rwkl.
- 90-bolt.rules stays staged (fat parity) but inert: it only sets
  SYSTEMD_WANTS+=bolt.service (the baked host unit's hotplug trigger,
  superseded by dbus activation). Thunderbolt access is pure sysfs
  (AppArmor path mediation, no device cgroup), so no udev-file
  attribute is needed.

Build: ./go-build (needs an apt-cacher/proxy env from go-build-lib.sh).

Consumers: gnome-shell (thunderbolt status agent + notifications) and
gnome-control-center (Thunderbolt panel) hold the session snap's
`bolt` plug, connected to this snap's slot via gadget static
connections. Interface XMLs ride gnome-shell's gresource — no host
/usr/share/dbus-1/interfaces files needed.
