I can audit this, but I cannot run either cleanup through a runner with no TTY.

Before proposing DNF cleanup on bootc, the audit must establish:

- Current deployment from `bootc status --format=json`
- DNF version and effective persistence mode
- Whether `/usr` is image-managed/read-only and whether changes survive reboot/update
- Separate persistence of `/etc` and `/var`
- Every effective persistent DNF cache root, its mount identity, size, and whether it belongs to the approved filesystem
- DNF prompt settings, including `assumeyes` and, for DNF 5, `DNF5_FORCE_INTERACTIVE`

Autoremove is unsupported for durable reclamation if package persistence is unknown or transient. For DNF 4, candidates can be audited with `dnf -C list --autoremove`. DNF 5's candidate query may create `~/.cache/libdnf5`, so its list remains unknown unless that diagnostic write is separately approved.

Cache cleanup and autoremove are separate operations. Each must retain its visible transaction prompt, run interactively without `-y` or other automatic-confirmation options, and be verified before proceeding to the next operation. Because this runner has no TTY, no cleanup will run; an interactive terminal is required.
