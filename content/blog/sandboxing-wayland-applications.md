+++
title = "sandboxing Wayland applications"
date = 2026-05-15
updated = 2026-05-15
+++

I like sandboxing Linux applications. Tools like [bubblewrap](https://github.com/containers/bubblewrap) and [Bubblejail](https://github.com/igo95862/bubblejail) make this easy, building on [Linux namespaces](https://en.wikipedia.org/wiki/Linux_namespaces).

This works well for command line applications, but sandboxing graphical applications is harder. A graphical Wayland application needs to access the Wayland socket to handle its windows. The Wayland socket gives access to sensitive data like screen contents. You might want to restrict this. With the [security_context](https://wayland.emersion.fr/protocol/security-context-v1.html) protocol, you can.

Instead of mounting the normal Wayland socket into the sandbox, mount a socket that has the security context applied. You can create such a socket with [way-secure](https://git.sr.ht/~whynothugo/way-secure). This makes Wayland applications work safely.

I mainly wanted to write about the Wayland socket in this post because there isn't much awareness of the bubblewrap plus way-secure approach. The Wayland socket is not everything. The application can show a window, but other desktop application features are missing. This is the same problem that [Flatpak](https://flatpak.org/) solves with tools like [XDG Desktop Portal](https://flatpak.github.io/xdg-desktop-portal/). You can reuse some of the Flatpak tools and approaches. Bubblejail does this to some degree but needs more polish.
