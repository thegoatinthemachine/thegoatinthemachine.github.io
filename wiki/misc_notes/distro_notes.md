---
---
## distro notes

### macOS

#### Webcam driver failure

If macOS is allowed to run without interruption for an extended period of time,
it is possible that the processes controlling the webcam may get hinky. I am
unclear on why exactly this happens.

Two commands can resolve this, depending on which process failed.

```bash
sudo killall VDCAssistant
sudo killall AppleCameraAssistant
```

### Manjaro

#### Installing a package from AUR (Arch User Repository)

TL;DR: Use pamac

The AUR is outside of pacman's scope, as it is not really a "repository" in the
same way that the arch or manjaro repos are. It instead contains individual git
repos and PKGBUILDS, which describe to pacman how and where to grab (and/or
build) binary packages. the [archwiki details][aw-AUR_helpers] several
AUR helper programs, which either wrap around pacman and provide searching of
the AUR, or similarly wrangle grabbing the git details from AUR and shoving
them into pacman.

pamac is one such tool built by the Manjaro team. It has a GUI program and a
CLI program. These can be configured to use the AUR. It also supports flatpack,
snap, etc, and has multithreading downloads built in.

[aw-AUR_helpers]: https://wiki.archlinux.org/title/AUR_helpers

