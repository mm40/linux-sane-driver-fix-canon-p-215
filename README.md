
# Canon P-215 scanner driver fix for Linux Sane

## Problem
When scanning documents with Canon P-215 scanner in duplex mode (both sides of a document), all odd pages, starting from the first one, have a white horizontal gap at the bottom, while all the even pages starting from the second one have the same horizontal white gape, but at the top. 

This is shown in a screenshot below, with first two pages scanned using `simple-scan`. White gaps which shouldn't be there are marked with red circles. Page size is set to A4, and the real page size is also A4.

<img width="1454" height="1074" alt="2026-05-15@22h-36m-51s_1454x1074" src="https://github.com/user-attachments/assets/0e012c27-d869-4d09-a384-f472cb8c6917" />

- This happens only when scanning in duplex mode. Scanners like P-215 are intended for this purpose.
- This happens in any program, not just `simple-scan`, including the command line utility `scandf`. Changing programs doesn't help, nor any of the settings in the programs.
- **NOTE**: set the page size manually in `simple-scan`. Auto-sizing pages doesn't work properly, and that's a separate issue from this.

## Intended solution (that doesn't work)
To solve this problem, a configuration file `/etc/sane.d/canon_dr.conf` can have a line `option duplex-offset 300` added below ``# P-215`, for Canon P-215 printer. **BUT THIS DOESN'T WORK**.

So what's the problem? Took me days to figure out, and a single line of code to fix it. It's the driver itself that's the problem.

## Solution that makes the intended solution work

### Fixing [Sane Backends (Drivers)](http://www.sane-project.org/sane-backends.html)

### Quick and dirty way to make it work (before the [patch for the official driver is mainlined](https://gitlab.com/sane-project/backends/-/merge_requests/921)):

1. `git clone https://gitlab.com/sane-project/backends.git`
2. `git clone https://github.com/mm40/linux-sane-driver-fix-canon-p-215.git`
3. `patch backends/backend/canon_dr.c < linux-sane-driver-fix-canon-p-215/sane-canon-p-215.patch`
4. `cd backends`
5. Run `./autogen.sh` until it works. I had to install `autoconf-archive` and `autopoint` for it to work
6. Run `./configure`, but if you get the warning that USB support won't be configured, install `libusb-dev` and run `./configure` again. After all, Canon P-215 IS a USB scanner
7. `make`



<details>
  <summary><b>8. Click here to see "dirty" of the "quick and dirty part"</b></summary>
  
The relevant file is whatever the link `backends/backend/.libs/libsane-canon_dr.so` is pointing to. In my case it's `backends/backend/.libs/libsane-canon_dr.so.1.4.0`. There is a corresponding file on your system that needs to be replaced with it. In case of my distro, it's `/usr/lib/x86_64-linux-gnu/sane/libsane-canon_dr.so.1.2.1`. To find such a file yourself, use `find / -type f -name 'libsane-canon_dr.so.*' 2>/dev/null`.  Replace that file with the newly created `libsane-canon_dr.so.X.Y.Z`:

Adapt these commands to your own system. For me 

9. `sudo mv /usr/lib/x86_64-linux-gnu/sane/libsane-canon_dr.so.1.2.1 /usr/lib/x86_64-linux-gnu/sane/_libsane-canon_dr.so.1.2.1` to create Backup
10. `sudo cp -L backend/.libs/libsane-canon_dr.so /usr/lib/x86_64-linux-gnu/sane/libsane-canon_dr.so.1.2.1` to copy the newly created, working file, over the original buggy one

11. `sudo apt-mark hold sane`: (optional) do this to prevent the sane package from updating, and overriding your fix. This is only temporary until this fix is mainlined
</details>

Right after the step 10 above, and adding `option duplex-offset 300` under `# P-215` in file `/etc/sane.d/canon_dr.conf`, scanning should work perfectly. Restart `simple-scan` and see whether it's the case. Adjusting the `300` values is possible now, that will change the amount of horizontal white space removed.


## Result

No more white horizontal space either above or below.

<img width="1454" height="1074" alt="2026-05-15@23h-05m-10s_1454x1074" src="https://github.com/user-attachments/assets/a15e103a-eba4-409d-9537-7a795b0e2056" />


## Notes
- Any time you change `/etc/sane.d/canon_dr.conf`, restart `simple-scan`.

TODO:
- [ ] Comment on line 1629 in `canon_dr.c` says "all copied from P-215". Does that mean this patch should be applied to all the printers whose code is "copied from P-215"?
