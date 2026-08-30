+++
date = '2026-08-28T11:30:00-06:00'
draft = false
title = 'Omarchy: Any User Process Can Escalate to Root'
+++

**A security issue in Omarchy's default Docker configuration meant that
essentially every program running in the user's desktop session could escalate
to root without a password, `sudo`, or a privilege prompt.**

If you use [Omarchy](https://omarchy.org), the most important takeaway is
simple: **update to 4.0.1.**

<iframe width="600" height="380" src="https://www.youtube.com/embed/_ZbhaoJt7tI" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

I reported this issue privately through the project's responsible-disclosure
process. The underlying configuration has since been patched, so I'm publishing
the details now to explain what the issue is and let users know to update their
systems.

## The Issue

Omarchy configured its default user as a member of the Linux `docker` group.

That allows users to run commands such as:

```bash
docker run ...
```

without typing `sudo`.

On arch the Docker daemon runs as root and listens on:

```text
/var/run/docker.sock
```

Members of the `docker` group can communicate with that socket. Docker itself
explicitly warns that the `docker` group grants root-level privileges to the
user.

A process with access to the Docker socket can ask the root-owned Docker daemon
to launch a container as root, mount arbitrary portions of the host filesystem
into it, operate on those files as root, and run code as root.

On affected Omarchy systems, this means that the default user and all processes
launched in that user session have access to root.

## Proof of Concept

On a fresh affected Omarchy installation try reading `/etc/shadow`:

```console
$ cat /etc/shadow
cat: /etc/shadow: Permission denied
```

Now observe the group memberships for your user:

```console
$ id
uid=1000(tester) gid=1000(tester) groups=1000(tester),967(docker),992(input),998(wheel)
```

Now read the protected file with docker acting as root:

```console
$ docker run --rm -v /:/hostroot alpine cat /hostroot/etc/shadow
root:$6$...
bin:!*:...
daemon:!*:...
...
```

The command is launched by an ordinary user process, but the actual filesystem
access is performed through a daemon running as root.

## Scope

Linux supplementary groups are inherited by child processes so this affects the
entire user session.

Walking the process tree below the user's `systemd --user` instance showed the
Docker group present on essentially every normal process in the session.

This means nearly every process where untrusted code could run, could obtain
root, including:

- AI coding agents and agent harnesses
- web browsers
- editors and IDEs
- npm scripts
- random development tools
- background processes

In other words, **a compromise of a normal user application could immediately
become a full machine compromise.**

## Security Defaults

There is another important aspect of this configuration. It was opt-out, not
opt-in. A user did not have to actually use Docker. The security tradeoff was
made for them, applied to the default account, and the tradeoff was not
explained to the user.

Security-sensitive defaults matter precisely because many users reasonably
assume that the operating system defaults to secure and will inform or prompt
them to opt-in to less secure settings.

## Misleading Documentation

Omarchy did mention the Docker group in its development-tools documentation:

> Omarchy installs everything needed to run [docker] well. This includes [...]
> the user group changes needed for you to run Docker as the normal user and
> not as root.

The security implication is almost the opposite of what a typical reader might
infer from "not as root". A user reading that description could reasonably
conclude that Omarchy had configured Docker in some kind of rootless mode. It
had not.

## Impacted Versions

This affects versions prior to 4.0.1. I tested it on the latest 3.x iso (3.8.4)
and it was also impacted.

## Timeline

The timeline of commits from the introduction to resolution of this issue:

- [June 1, 2025 — Docker group membership introduced `25799ee`](https://github.com/basecamp/omarchy/commit/25799ee91f54c35e6d340df3aae8ac2b21fae0a4)
- [June 2, 2025 — Docker group addition temporarily disabled `c5ee230`](https://github.com/basecamp/omarchy/commit/c5ee230dafe665d78b65b116fb23cae3ead6b174)
- [June 17, 2025 — Docker group membership re-enabled `fdd2aaf`](https://github.com/basecamp/omarchy/commit/fdd2aaf4d1f84647ac08de04384313a0c681afdb)
- [August 24, 2026 — Docker group membership removed from the default configuration `b5ded31`](https://github.com/basecamp/omarchy/commit/b5ded31e2f9a86a15442e838f0e548346b91f375)

## The Broader Context

As AI is increasingly producing high severity CVEs against core infrastructure,
security needs to be top of mind for all developers, but especially authors of
distributions targeted at developers. Lately there have been innumerable
reports of developer machines being compromised and their access used to
contaminate the software supply chain or exploit production systems. Developers
are high-value targets because of the level of access they are often granted.
Developer machines typically disable security guardrails for convenience, store
credentials in plain-text dotfiles, and accumulate access to systems. This
**must** change.

I'm sure this was just an oversight by DHH not knowing the implications of
adding the docker group. No distribution is going to make perfect decisions
when it comes to security. I was amazed by the speed of response to this issue
being reported which is a healthy sign.

That said, this isn't the first time I have ran into security issues with
Omarchy and frankly I do not trust the decision making process as it stands to
ensure the level of security I expect out of my distro. I hope that changes at
some point because there is a lot to like about Omarchy.

## Podman

If you are a user of Docker on Linux and don't want to be forced into granting
root (even with sudo) to run containers, then I highly recommend you try out
[Podman](https://podman.io/). Podman is daemonless. Your containers run as
normal child processes in their own user namespaces and don't require any sort
of root access. I have been running Podman for many months now and it has
entirely replaced all of my Docker workflows. I highly recommend giving it a
try.

## References

- [Docker: Manage Docker as a non-root user](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user)
- [Omarchy Docker documentation](https://omarchy.org/manual/development-tools/#docker)
