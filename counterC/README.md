# Install Solana C compiler

```console
./install-solana-c.sh
```

# Build the program

- Go to a program directory

```console
cd counterC
```

```console
make
```

# Test the C program

```console
SBF_OUT_DIR="out" cargo test --manifest-path "Cargo.toml"
```

OR use the helper from the root of this repo to build and test

```console
./test-c.sh counter
```

# Test the unsave Rust C program

A similar result can also be achieved using unsafe rust. The code is in the `counter/src/lib.rs` file. The code is compiled to a shared object file using the following command:

```bash
cargo build-sbf
```

Then you can run the same tests against the rust compiled file:

```bash
SBF_OUT_DIR="../target/deploy" cargo test --manifest-path "Cargo.toml"
```

OR use the helper from the root of this repo to build and test the rust version as well:

```console
./test-rust-mangle-c.sh counter
```

Both the C and Rust version result in 5 CU for increasing a counter. So its a matter of taste which one you want to use.

If you want to see the generated assembly code, you can run the following command:

```bash
./solana-c-sdk/dependencies/platform-tools/llvm/bin/clang \
  -target bpfel \
  -fno-builtin \
  -std=c17 \
  -O2 \
  -S \
  -I ./solana-c-sdk/c/inc \
  -I ./solana-c-sdk/dependencies/platform-tools/llvm/lib/clang/17/include \
  -I ./solana-c-sdk/dependencies/platform-tools/llvm/include \
  -o counter/src/c-assembly-code.s \
  ./counter/src/main.c
```

This is using the `clang`compiler with the `-S`flag which generates assembly code.

If you want to see the generated Assemlby code for the C mangled code you can use `llvm-objdump`:

First install llvm:

```bash
brew install llvm
```

and then decompile the generated `.so` file using `llvm-objdump` and write it into a file:

```bash
llvm-objdump \
--demangle \
--print-imm-hex \
-g --full-leading-addr \
--debug-vars --section-headers \
--symbolize-operands \
--source --disassemble target/deploy/solana_program_rosetta_helloworld.so > counter/src/rust-unsave-c-code-assembly.s
```

To get more info in the dissasebmled file you can use different flags when compiling. For example:

```bash
RUSTFLAGS="-C debuginfo=2 -C opt-level=0" cargo build-sbf
```

of if you cant see some functions even:

```bash
RUSTFLAGS="-C debuginfo=2 -C opt-level=0 -C link-dead-code" cargo build-sbf
```

If you wonder where the strings that you are printing are actually saved. Its in the `.rodata`which stands for read only data.

You can find all strings in an `.so` file using the following command:

```bash
strings target/deploy/solana_program_rosetta_helloworld.so
```

or you can directly print the `.rodata` section using `llvm-objdump`:

```bash
llvm-objdump -s -j .rodata target/deploy/solana_program_rosetta_helloworld.so
```

You can also read the elf file using `readelf`:

```bash
readelf -a target/deploy/solana_program_rosetta_helloworld.so
```

### Input Parameter Serialization

SBF loaders serialize the program input parameters into a byte array that is
then passed to the program's entrypoint, where the program is responsible for
deserializing it on-chain. One of the changes between the deprecated loader and
the current loader is that the input parameters are serialized in a way that
results in various parameters falling on aligned offsets within the aligned byte
array. This allows deserialization implementations to directly reference the
byte array and provide aligned pointers to the program.

The latest loader serializes the program input parameters as follows (all
encoding is little endian):

- 8 bytes unsigned number of accounts
- For each account
  - 1 byte indicating if this is a duplicate account, if not a duplicate then
    the value is 0xff, otherwise the value is the index of the account it is a
    duplicate of.
  - If duplicate: 7 bytes of padding
  - If not duplicate:
    - 1 byte boolean, true if account is a signer
    - 1 byte boolean, true if account is writable
    - 1 byte boolean, true if account is executable
    - 4 bytes of padding
    - 32 bytes of the account public key
    - 32 bytes of the account's owner public key
    - 8 bytes unsigned number of lamports owned by the account
    - 8 bytes unsigned number of bytes of account data
    - x bytes of account data
    - 10k bytes of padding, used for realloc
    - enough padding to align the offset to 8 bytes.
    - 8 bytes rent epoch
- 8 bytes of unsigned number of instruction data
- x bytes of instruction data
- 32 bytes of the program id
