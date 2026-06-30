# buf `IMPORT_USED` bug repro

`IMPORT_USED` silently produces no output when `google/protobuf/descriptor.proto` is in the transitive dependency graph.

## Setup

```sh
bun install
```

## Reproduce

```sh
bun run lint:bug    # exits 0 — unused import NOT reported (bug)
bun run lint:valid  # exits 1 — unused import correctly reported
```

## Structure

```
bug/repro/v1/
  opts.proto   extends google.protobuf.FileOptions → pulls descriptor.proto into dep graph
  main.proto   imports opts.proto + unused timestamp.proto → IMPORT_USED silent

valid/repro/v1/
  clean.proto  same unused timestamp.proto, no descriptor.proto → IMPORT_USED works
```

## 🤖 AI potential root cause (take it with a grain of salt, I didn't verify below part)

`protocompile` correctly embeds `UnusedDependency` in the `SourceCodeInfo` buf extension (field `536000000`). However, `resolverForFDS` in `build_image.go` skips adding `buf/descriptor/v1/descriptor.proto` to the resolver when `google/protobuf/descriptor.proto` is present in the compiled output. Without it, `ReparseExtensions` cannot decode the extension, `Has(E_BufSourceCodeInfoExtension)` returns false, and the unused dependency data is lost.

Tested on buf `1.71.0`.
