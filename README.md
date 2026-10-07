# go-rattler

Go bindings for selected Rust rattler functionality. `CompareVersions` uses `rattler_conda_version`; `LockEnvironment` counts conda and PyPI packages for one environment/platform in a `pixi.lock` via `rattler_lock`.

One Rust FFI crate exports one C ABI, split by domain under `internal/native/src/`. The private `internal/api` Go package owns cgo; the public Go package holds idiomatic APIs. This follows rattler's Python/JS binding layout and avoids linking multiple Rust static libraries (which can conflict). Add only the upstream crates needed by implemented calls.

## Build from source

Requires Go 1.26+, Rust/Cargo, cgo, and a C compiler. `go build` does **not** run Cargo. From a consuming Go module, build the exact Go dependency's Rust source first, then link its archive into your final executable:

```sh
module=github.com/dbast/go-rattler
go get "$module@vX.Y.Z" # once released; omit if already pinned in go.mod
go mod download "$module"
source_dir="$(go list -m -f '{{.Dir}}' "$module")"
native_dir="$PWD/.go-rattler-native" # writable; git-ignore this directory
CARGO_TARGET_DIR="$native_dir" cargo build --manifest-path "$source_dir/internal/native/Cargo.toml" --release --locked
CGO_ENABLED=1 CGO_LDFLAGS="-L$native_dir/release" go build -o my-program ./cmd/my-program
```

The cgo wrapper embeds `libgo_rattler_native.a` in the consumer executable; no separate Rust library is needed at runtime. This does not imply a fully static OS binary: platform system libraries may still be dynamic. For Windows amd64, use MinGW-w64 for Go cgo, install the `x86_64-pc-windows-gnu` Rust target, set `CARGO_BUILD_TARGET=x86_64-pc-windows-gnu`, and use `$native_dir/x86_64-pc-windows-gnu/release` in `CGO_LDFLAGS`. The wrapper also links the Windows system import libraries required by Rust's static library. A normal `go get` followed by bare `go build` cannot perform the Rust step; native artifacts are not distributed yet. Locally, `make check` builds the archive and tests both languages.

Licensed under BSD-3-Clause, matching rattler. The initial publication target is `github.com/dbast/go-rattler`. The separate-module test uses a local `replace`; it does not prove that a tagged version contains the Rust sources. Before tagging a release, test a fresh consumer against the published module and verify its native source is included.
