# `msp430f5529-quickstart`

> A template for building applications for TI MSP430 microcontrollers.

This project is developed and maintained by the [MSP430 team][team].

## Dependencies

- Rust nightly-2022-01-24 or a newer toolchain. _Only nightly compilers work
  for now._

  The [`rust-toolchain.toml`](./rust-toolchain.toml) file makes sure this step
  is done for you. This file tells [`rustup`](https://rustup.rs/) to download
  and use nightly compiler for this crate, as well as download the compiler
  source to build `libcore`.

  You can manually set up a nightly compiler as the default for all MSP430
  projects by running:

  ``` console
  $ rustup default nightly
  $ rustup component add rust-src
  ```

  Alternatively, you can manually set up overrides on a project basis using:

  ```console
  $ rustup override set --path /path/to/crate nightly-YYYY-MM-DD`.
  $ rustup component add --toolchain nightly-YYYY-MM-DD rust-src
  ```

- The `cargo generate` subcommand ([Installation instructions](https://github.com/ashleygwilliams/cargo-generate#installation)).

- TI's [MSP430 GCC Compiler](http://www.ti.com/tool/MSP430-GCC-OPENSOURCE),
  version 8.3.0 or greater. `msp430-elf-gcc` should be visible on the path.

The following dependencies are required in order to generate a peripheral access crate (see *Using this template* below)

- [`svd2rust`](https://github.com/rust-embedded/svd2rust), version 0.20.0 or
  a newer commit.

- [`msp430_svd`](https://github.com/pftbest/msp430_svd), version 0.3.0 or a
  newer commit. Cloning the repo and following the directions in README.md is
  sufficient.

## Using this template

0. Before instantiating a project, you need to know some characteristics about
   your specific MSP430 device that will be used to fill in the template. In
   particular:

   - What is the exact part number of your device? Which MSP430 family does
     your part belong to?

   - Does your part have an already-existing [peripheral access crate](https://rust-embedded.github.io/book/start/registers.html) (PAC)?

   - How much flash memory, RAM, and how many interrupt vectors does your
     device have?

   - Where does flash memory, RAM, and interrupt vectors begin in the address
     space?

   `msp430f5529` is used for the examples in this crate. Answering the first
   question above:

   - `msp430f5529` belongs to the [`MSP430x5xx` family](https://www.ti.com/lit/ug/slau208q/slau208q.pdf).

   The linked family manual and [datasheet](https://www.ti.com/lit/ds/slas590p/slas590p.pdf)
   can be used to answer the remaining questions:

   - The `msp430f5529` PAC exists already and can be found on [crates.io](https://crates.io/crates/msp430f5529).

   - `msp430f5529` has 128kB of flash memory, 8kB of RAM, and 64 vectors.

   - Flash memory begins at address 0x4400. RAM begins at address 0x2200.
     The interrupt vectors begin at 0xFF80.

1. If your particular device does not have a PAC crate, you will need to
   generate one using `svd2rust` and possibly [publish](https://doc.rust-lang.org/cargo/reference/publishing.html)
   the crate to https://crates.io. To generate an SVD file, follow the directions
   in the `msp430_svd` [README.md](https://github.com/pftbest/msp430_svd#msp430_svd)
   for your device.

   In some cases like [`msp430fr2355`](https://github.com/YuhanLiin/msp430fr2355-quickstart/issues/4#issuecomment-569178043),
   TI's linker files are not consistent with datasheets on where interrupt
   vectors begin and how many interrupt vectors there are for a given device.
   In case of a conflict, _[examine](https://github.com/YuhanLiin/msp430fr2355-quickstart#issuecomment-569320608)
   the linker script to determine where to start the `VECTORS` section in
   `memory.x`._ Copies of many linker scripts are kept in the
   [`msp430_svd`](https://github.com/pftbest/msp430_svd/tree/master/msp430-gcc-support-files)
   repository.

2. Instantiate the template and follow the prompts.

   ``` console
   $ cargo generate --git https://github.com/cr1901/msp430f5529-quickstart
   Project Name: app
   Creating project called `app`...
   Done! New project created /tmp/app

   $ cd app
   ```

3. If not targeting `msp430f5529`, replace the PAC entry for `msp430f5529` in
   `Cargo.toml` with that of your device, e.g.:

   ``` toml
   # [dependencies.msp430f5529]
   # version = "0.3.0"
   # features = ["rt"]
   [dependencies.msp430fr2355]
   version = "0.4.0"
   features = ["rt"]
   ```

4. Populate the `memory.x` file with address space layout information of your
   device. _Note: As mentioned above, in case of a conflict between the
   datasheet and linker script on where interrupt vectors begin, use the
   linker script!_

   ``` console
   $ cat memory.x
   MEMORY
   {
     /* These values are correct for the msp430f5529 device. You will need to
        modify these values if using a different device. Room must be reserved
        for interrupt vectors plus reset vector and the end of the first 64kB
        of address space. */
     RAM : ORIGIN = 0x2200, LENGTH = 0x2000
     ROM : ORIGIN = 0x4400, LENGTH = 0xBB80
     VECTORS : ORIGIN = 0xFF80, LENGTH = 0x80
   }
   ```

5. Build the template application or one of the examples. Some examples
   (such as `timer` or `temp-hal`) may not compile due to size
   constraints when building using the `dev` profile (the default). Pass the
   `--release` option to `cargo build` in these cases.

   ``` console
   $ cargo build
   $ cargo build --examples
   ```

   Note that due to [`.cargo/config`](.cargo/config) and [`rust-toolchain.toml`](./rust-toolchain.toml),
   the above is shorthand for:

   ``` console
   $ cargo +nightly build --target=msp430-none-elf -Zbuild-std=core
   $ cargo +nightly build --target=msp430-none-elf -Zbuild-std=core --examples
   ```

   You may wish to experiment with other commented options in `.cargo/config`.

6. Once you have an ELF binary built, flash it to your microcontroller. Use [`mspdebug`](https://github.com/dlbeer/mspdebug) to launch a debug session and `msp430-elf-gdb` with the linked gdb script. For the msp430f5529 and the MSP-EXP430G2 launchpad board this looks like the following:

   In one terminal session
   ```console
   $ mspdebug -C mspdebug.cfg rf2500
   ```

   In another terminal session
   ```console
   $ msp430-elf-gdb -x mspdebug.gdb target/msp430-none-elf/debug/app
   ```

   This will flash your Rust code to the microcontroller and open a gdb debugging session to step through it.

# License

Licensed under either of

- Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or
  http://www.apache.org/licenses/LICENSE-2.0)

- MIT license ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)
  at your option.

## Contribution

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as above, without any additional terms or conditions.

[team]: https://github.com/rust-embedded/wg#the-msp430-team
