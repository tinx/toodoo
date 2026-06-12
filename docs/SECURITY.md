# Security Considerations

## Input Validation

All external input is validated, including data read from databases, network
connections, command line parameters, environment variables, config files,
etc. We trust nothing, we verify all input.

## Content Types and Encoding

We keep track of the content type of every payload, so that clients (including
web frontends) can safely encode special characters and avoid cross site
scripting attacks. We use parameter binding for SQL queries, too.

## Secret Injection

We do not store secrets (such as keys, passwords, tokens, etc.) in git, nor
in any config files or database. They need to be injected at runtime via
container volume bindings.

## Supply Chain Security

Recently, there have been a number of supply chain attacks in the open source
world. Credential stealers were build into public libraries and packages, and
everyone downloading and using these dependencies had their secrets exposed
to unknown attackers. Another attack vector was introducing prompt injection
instructions into open source projects. In order to avoid these problems we
do two things:

 * if dependencies publish an update, we wait one week before using it,
   in case it is retracted
 * after one week, we will analyze the changes of the old versus the new
   version locally, in order to look for harmful changes, such as
   credential stealers or backdoors

## Privilege Separation

Privilege Separation is a security architecture to split a service into different
processes, where each process is granted only the privileges necessary for its
part. For example, a minimalistic listener could be allowed to only accept
connections and pass file descriptors to another component.

Golang doesn't easily support multiple processes and IPC, but it can be done via
cmd.Exec() and multiple binaries or mode parameters for the same binary.

We can drop privileges when we no longer need them, etc. Some things to consider:

 * use linux-seccomp-bpf to drop privileges and forbid syscalls in future execution
   (see https://github.com/oxzi/syscallset-go)
 * look at a go implementation of landlock, for similar effect
 * fork and exec to run in multiple modes, use a socket pair to pass file descriptors
   after accepting network connections, etc.
 * we can't use chroot() because it required root privileges
 * but maybe we can use setresuid() and setuid() ?
 * Nice talk explaining these things: https://archive.fosdem.org/2025/schedule/event/fosdem-2025-5302-privilege-separation-in-go/

podman can also help here:
  --cap-drop=ALL --cap-add=<only what is needed> --security-opt no-new-privileges
  
## MFA and Session Tokens

## OWASP Top 10

## 12 Factor App

## Rate Limiting

## Health and Readiness Probes

## Logging

## Encryption

## Logging




Is there a way to drop privileges from a program running inside a rootless podman container?         

● Yes — several ways, and they compose. The key mental model for rootless Podman: the container's
  "root" (UID 0) is already mapped to your unprivileged host UID via a user namespace, and the
  container gets a range of UIDs (your /etc/subuid allocation, usually 65536 of them) it can switch
  among. So dropping privileges inside the container is still meaningful as defense-in-depth
  (capability and filesystem isolation), even though container-root isn't host-root.

  Here are the main approaches, from outside-in:

  1. Just don't start as root

  Best option when you can. Either bake it into the image or set it at run time.

  # Containerfile
  RUN useradd -u 1000 app
  USER 1000:1000

  podman run --user 1000:1000 myimage

  The UID/GID you pick must fall within the namespace's mapped range (it does for anything ≤ 65535 by
  default).

  2. Drop from root to a user in an entrypoint

  If the process needs to do some root-only setup (bind a low port, fix permissions) and then drop, use
  a privilege-dropping exec helper in your entrypoint:

  # setpriv (util-linux, no extra dependency)
  exec setpriv --reuid=1000 --regid=1000 --clear-groups --inh-caps=-all myprogram

  # or gosu / su-exec (common in container images)
  exec gosu 1000:1000 myprogram

  setpriv is preferable to su/sudo — it doesn't fork a login shell or PAM stack and won't leave a root
  parent around.

  3. Drop privileges from within the program itself

  The classic syscall sequence, in this exact order (groups before setuid, because once you lose UID 0
  you can't call setgroups):

  setgroups(0, NULL);   // or set the target's supplementary groups
  setgid(target_gid);
  setuid(target_uid);   // do this LAST

  Always check return values — a silently failed setuid is a classic root-retained-by-accident bug.
  After dropping, verify you can't regain UID 0 (e.g. setuid(0) should fail). This requires the process
  to start with CAP_SETUID/CAP_SETGID (it has them if started as container-root unless you dropped
  them).

  4. Drop capabilities (often what you actually want)

  Privilege isn't only UID. Strip Linux capabilities so even container-root can't do much:

  podman run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myimage

  From inside a program, use libcap (cap_set_proc) or setpriv --inh-caps=-all --bounding-set=-all.

  5. Lock it down so privileges can't come back

  podman run --security-opt no-new-privileges --user 1000:1000 ...

  no-new-privileges sets the kernel NO_NEW_PRIVS bit, so setuid binaries / file capabilities can't
  escalate after you've dropped down.

  ---
  Rootless-specific gotchas:
  - You can only switch to UIDs/GIDs mapped into the namespace. Outside your subuid range, setuid fails
  with EINVAL. Check cat /etc/subuid for your allocation, or podman unshare cat /proc/self/uid_map
  from inside. 
  - Files created by the dropped UID appear on the host as host_uid + (container_uid - 1) from your
  subuid base — relevant if you bind-mount volumes.
  - --userns=keep-id maps your host UID straight through, which can simplify volume permissions but
  changes which in-container UIDs are available to drop to.
  
  For a typical web service the canonical recipe is: USER non-root in the image + --cap-drop=ALL 
  --cap-add=<only what's needed> --security-opt no-new-privileges. Reach for the in-program setuid
  dance only when the process genuinely needs a privileged startup phase.

  If you tell me your specific case (needs a low port? bind-mounted volumes? a setup step that needs
  root?), I can give you an exact command.

✻ Brewed for 32s

❯ My specific case is about a Go service in a rootless podman container for privilege separation. The  
  service will accept connections (on a high port, no root access required to open it) and will        
  perform the TLS handshake. Then I want it to create a new local connection the the backend service   
  and relay traffic both ways until the connection terminates from either side or idles out. The       
  purpose is security isolation of the accept() and TLS parts. The container should be restricted as   
  much as possible and leave as little attack surface as possible.                                     
  
● Good news: in your design there's nothing to drop — and for a Go service that's exactly what you
  want, because in-process privilege dropping is a footgun in Go specifically.

  Why you shouldn't drop privileges in-process (Go-specific)

  On Linux, setuid() is a per-thread attribute in the kernel. The glibc wrapper papers over this by
  broadcasting to all threads; Go's runtime issues raw syscalls and historically did not, so
  syscall.Setuid only changed one of the Go scheduler's threads — leaving other threads running as
  root. Go 1.16 fixed this (syscall.Setuid/Setgid now apply process-wide), but the runtime spins up
  threads before your main runs, so any "start as root, then drop" dance is fragile and easy to get
  subtly wrong.

  Since you're on a high port, no capability needed to bind it, just never acquire privilege in the
  first place. That deletes the entire problem.

  Start unprivileged

  # Containerfile — static binary, no base OS at all
  FROM golang:1.23 AS build
  WORKDIR /src
  COPY . .
  RUN CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o /relay ./cmd/relay

  FROM scratch
  COPY --from=build /relay /relay
  USER 1000:1000          # non-root inside the namespace, belt-and-suspenders
  ENTRYPOINT ["/relay"]

  FROM scratch means there is no shell, no libc, no package manager, no setuid binaries — if the
  process is popped, the attacker lands in an empty filesystem with one static binary. (Use
  gcr.io/distroless/static:nonroot instead if you want CA certs / /etc/passwd / timezone data.)

  Lock the container down

  podman run \
    --user 1000:1000 \
    --cap-drop=ALL \                       # a high-port proxy needs zero capabilities
    --security-opt no-new-privileges \
    --read-only \                          # immutable rootfs
    --tmpfs /tmp:rw,noexec,nosuid,size=8m \ # only if you actually need scratch space
    --pids-limit 64 \
    --memory 128m --memory-swap 128m \
    --cpus 1.0 \
    --ulimit nofile=1024:1024 \            # bound FD exhaustion DoS
    -p 8443:8443 \
    myimage

  - --cap-drop=ALL — confirm with --cap-add nothing. You don't even need NET_BIND_SERVICE.
  - --read-only — the relay writes no files. Mount the cert/key read-only (below); add a tmpfs only if
  some library insists on writing.
  - The resource limits matter here: a TLS-terminating front end is a natural DoS target (handshake
  floods, slowloris, connection exhaustion). --pids-limit, --memory, and the FD ulimit cap the blast
  radius.

  Seccomp

  Podman's default seccomp profile is already a strong allowlist (blocks the dangerous ~40+ syscalls)
  and is tuned to work with Go runtimes. Keep it. Writing a custom tighter profile by hand for Go is
  error-prone because the scheduler's syscall set (futex, epoll, mmap, clone, …) is broad and
  version-dependent. If you want to invest, record the real set rather than guessing:

  podman run --annotation io.containers.trace-syscall=of:/tmp/relay.json ...   # oci-seccomp-bpf-hook

  then load it with --security-opt seccomp=/tmp/relay.json. Test under load before trusting it — a
  missing syscall surfaces as a random crash under traffic.

  The honest threat boundary

  Be clear-eyed about what this isolates. The thing you're protecting the rest of the system from is
  the TLS/parse code path running on untrusted bytes. What it does not protect:

  - The TLS private key lives in this container. A full compromise of the front end leaks it — that's
  intrinsic to terminating TLS here. Mount it read-only and treat it as exposed: podman secret or -v 
  key.pem:/key.pem:ro. If that key is too valuable to lose, the only real fix is moving signing to a
  separate process/HSM (Go's crypto/tls supports a Certificate.PrivateKey implementing crypto.Signer,
  so the key could live behind an RPC) — heavier, probably overkill, but worth naming.
  - What you do get: the backend's filesystem, secrets, and credentials are not in this container. A
  popped front end can only do what the relay can do — open a connection to the backend over the one
  channel you give it.

  Make that channel a Unix domain socket, not TCP. Bind-mount a socket from the backend into the
  front-end container and dial it. That way the front end needs no route to the backend's network at
  all — it can't port-scan, can't reach other internal hosts, can't exfiltrate sideways. Combined with
  egress filtering on the inbound network (pasta/slirp), a compromised front end is boxed into "talk to
  exactly one socket."

  Relay implementation notes (Go)

  A few things that bite people writing this loop:

  - Idle timeout: don't rely on a single deadline. Wrap each conn so every successful Read pushes the
  deadline forward (SetReadDeadline(time.Now().Add(idle))); when io.Copy returns a timeout error, tear
  down both sides.
  - Half-close: when one side sends EOF, propagate it with CloseWrite() (*net.TCPConn / *tls.Conn both
  support it) rather than slamming the whole connection, or you'll truncate in-flight data on protocols
  that half-close.
  - Two io.Copy goroutines, a shared cancel, and a hard cap on total connection lifetime in addition to
  the idle timeout.
  - Set tls.Config conservatively: MinVersion: tls.VersionTLS12 (or 1.3), a pinned cipher list if on
  1.2, and a sane Config.Time/handshake timeout via conn.SetDeadline before Handshake() so a stalled
  handshake can't pin a goroutine + FD forever.

  ---1G
  Want me to write the actual relay (cmd/relay) with the idle/half-close/handshake-timeout handling,
  plus the Containerfile and a backend-over-Unix-socket wiring? I can produce a working skeleton you
  can drop in.


