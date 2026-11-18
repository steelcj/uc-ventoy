# Create a 512 checksum

Use the SHA-512 command for your platform. All of these produce the same hex hash:

bash

```bash
# Linux
sha512sum ubuntu.iso > SHA512SUMS
sha512sum -c SHA512SUMS          # verify later

# macOS
shasum -a 512 ubuntu.iso > SHA512SUMS
shasum -a 512 -c SHA512SUMS
```

powershell

```powershell
# Windows (PowerShell)
Get-FileHash .\ubuntu.iso -Algorithm SHA512
```

To checksum every ISO in a version folder at once on Linux, run `sha512sum *.iso > SHA512SUMS` inside the folder. The macOS command writes the same format, so `sha512sum -c` and `shasum -a 512 -c` can read each other’s files. PowerShell prints a table instead, so compare its hash by eye or with a short script.

In Python (any OS), using the standard library:

python

```python
import hashlib
h = hashlib.sha512()
with open("ubuntu.iso", "rb") as f:
    for block in iter(lambda: f.read(4 * 1024 * 1024), b""):
        h.update(block)
print(f"{h.hexdigest()}  ubuntu.iso")
```

The current `iso_lock.py` doesn’t read a `SHA512SUMS` file. Its parser only accepts 64-character SHA-256 hashes, so a 128-character SHA-512 line is ignored and the image counts as unlisted under `--verify`. Debian publishes both kinds of sums, but Ubuntu only publishes `SHA256SUMS`. If you want `--verify` to read SHA-512 files as well, that’s a small change.