# Patches

Patches applied to the osTicket sources in `/var/www/html` during the image build (see `Dockerfile`), in
alphabetical order, with `patch -p1 --forward --fuzz=0`. The build fails if

- the code around a patch changed upstream (no fuzz allowed; plain line offsets are fine), or
- upstream already contains the change (`--forward`). Then delete that patch.

Each patch starts with a short description, followed by a unified diff against the pristine release
(paths relative to the release's `upload/` directory).

## Creating or updating a patch

```sh
wget -O o.zip https://github.com/osTicket/osTicket/releases/download/v<version>/osTicket-v<version>.zip
unzip o.zip 'upload/*' && cp -a upload a && cp -a upload b
# apply the existing patches to b/ if you are updating one, then edit the file in b/
diff -u a/include/foo.php b/include/foo.php > foo.patch   # then prepend the description
```

Diffing from inside the directory with `--label a/<path> --label b/<path>` gives the same `-p1` paths.
