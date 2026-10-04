# Windows build notes

## Native application

On Windows, use `lake exe lean-fmt` so Lake supplies the DLL search path. The Unix `make install`
instructions in the [repository README](../../README.md#install) do not apply. CI builds, lints,
and runs the unit tier and selected regression tests on Windows. Some integration suites require
Unix tools.

## Manual build issues

Run `lake exe docs` from `docs/manual`. Output is written to `_out/html-multi/` by default.
The unmodified pinned dependencies build successfully on Ubuntu 22.04. Their Windows build has
the two upstream issues recorded below. This repository does not apply compatibility patches.

### Investigated versions

The investigation on 2026-10-04 used Lean `v4.34.0`, Verso `v4.34.0`
(`cad4b633e75ea769b851f12f9ca3b4f0dfcc625f`), and the manifest's MD4Lean revision
`31907cc18f48a95384f99cee5582c00fb39e0f67`.

### DLL exports

`VersoManual.Docstring:dynlib` fails to link on Windows with an undefined
`lean_md4c_markdown_parse` symbol. MD4Lean includes its C wrapper in its shared library, but the
wrapper's Windows compilation omits `-DLEAN_EXPORTING`. In Lean's header, `LEAN_EXPORT` expands
to `__declspec(dllexport)` on Windows only when `LEAN_EXPORTING` is defined.

Adding `-DLEAN_EXPORTING` to the Windows flags of `wrapperOTarget` in MD4Lean's `lakefile.lean`
allowed the DLL to link in a local experiment. That target hashes its C input but not its
compiler flags, so changing only the flags can retain an old object. The experiment invalidated
`.lake/packages/MD4Lean/.lake/build/wrapper/wrapper.o.trace` before rebuilding. Recovery from an
object compiled without the export flag was also verified.

### Embedded search asset paths

After the DLL fix, HTML generation failed with a path such as
`_out\html-multi\-verso-search\../../../static-web/search\domain-mappers.d.ts`.
In `VersoSearch/DomainSearch.lean`, `searchBoxCode` removes a forward-slash source-directory
prefix from filenames produced by `include_bin_dir`. The Windows filenames contain backslashes,
so the prefix is not removed.

Normalizing the filename before removing the prefix allowed HTML generation to complete:

```lean
let name := name.replace "\\" "/"
```

### Experimental validation

With both local changes, Windows generated the manual successfully. Repeated generation and
an alternate output directory also succeeded. All 104 local HTML references resolved, and the
copied search assets matched their source bytes. The experimental source changes were then
removed; these results describe the tested remedies, not an unmodified Windows build.

Serve generated HTML over HTTP for preview. Search and code hover data use requests that browsers
block under `file://`.
