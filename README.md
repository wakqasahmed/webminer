# Webminer

An experimental vector-accelerated CPU miner for the Webcash electronic payment network.

Webminer is tested and known to work on recent versions of macOS and Linux.  It is written in a platform-independent style and is likely to work on other operating systems as well.  If you need to make modifications to get it to run in your environment, please consider submitting a pull request on the GitHub repo: https://github.com/maaku/webminer

# Performance

Unlike the official Webcash miner which is written in unoptimized Python, webminer is written in hand-tuned C++ and uses x86 vector SIMD extensions to accelerate SHA256 calculations.  On this author's Intel CPU the official Python miner is able to achieve approximately 15.5 khps.  A single-threaded webminer instance on the same machine is able to mine approximately 22.65 Mhps, for a total speedup of over 1460x.

Further optimizations are possible, and will be implemented in time.

# Compiling

To compile webminer, a C++ compiler and a few standard developer tools are required.  On Ubuntu you can install the required dependencies with the following command:

```
sudo apt-get install build-essential autoconf automake libtool
```

You will also need Google's bazel build tool, which is not available from the default Ubuntu package repositories.  Instructions for installing bazel are available at the official bazel website: https://bazel.build

On macOS the build environment can be prepared by installing XCode (with the command-line developer tools), then fetching the required dependencies with homebrew:

```
brew install autoconf automake libtool bazel
```

To build webminer, open a shell and navigate to directory containing the source code and execute this command:

```
bazel build -c opt webminer
```

Bazel will handle the details of fetching and compiling all the necessary dependencies.

# Running

The final built executable will be available at `bazel-bin/webminer`.  Simply run this executable and it will begin mining webcash.

Mining progress will be output to stdout (along with lots of other info).  The claim codes for mined webcash will be stored in a webminer-formatted wallet file named `default_wallet.db` in the current directory.  The recovery key is stored in the file `default_wallet.bak`, and this "master secret" can be used to recover the wallet contents using the official Python wallet as well.  The base name of these files can be changed with the `--walletfile=basename` command line option.

If webcash is successfully generated and confirmed on the server but there is an error adding the webcash to the wallet, the claim codes will be output to a dedicated file named 'webcash.log' in the current directory.  If webcash is successfully generated but there is an error communicating to the server, the relevant information (including both the proof-of-work solution and the claim code) are output to a file named 'orphans.log' in the current directory.  The names of both files can be changed with the `--webcashlog=filename` and `--orphanlog=filename` command line options.

Webminer will automatically spawn mining threads equal to the number of execution units on the machine in which it is running.  To control precisely the number of mining threads, use the `--workers=N` option.

WARNING: Do *NOT* execute webminer with with `bazel run`!  Webminer will generate files to store the claim codes for any webcash generated, and these files will be destroyed along with the temporary sandbox created by `bazel run`.

# Wallet

Webcash claim codes generated by mining need to be inserted into a wallet and replaced quickly, in case the mining submissions are ever released as part of an audit.  This is done automatically by webminer, and the replaced webcash are stored in webminer's internal wallet (`default_wallet.db` in the current directory).

If there is an error storing replacing the webcash or storing it in the wallet, the claim codes will be output to a plain text file which can be inserted into any webcash wallet using the official webcash wallet tool:

### Without cleaning the log file
```
cat webcash.log | xargs -n 25 webcash insertmany
```

### With cleaning the log file
```
cat webcash.log | xargs -n 25 webcash insertmany | >> webcash_claim.log && echo -n > webcash.log
```

Or something similar along those lines.

WARNING: You *must* claim the generated webcash output to `webcash.log` in a wallet quickly after generating them, or else you risk forfeiting the funds if/when your mining report is released.  Mining reports are not treated as sensitive data, so this can happen at any time!

# Webcash server (EXPERIMENTAL)

This repository also contains a reimplementation of the webcash server daemon.  To run the webcash server you need a few extra build and runtime dependencies in addition to all the webminer dependencies mentioned above.  On macOS you have two options: either install the full PostgreSQL (e.g. via `brew install postgresql`), or inst just the needed client library and perform some manual environment setup yourself:

```
# macOS
brew install libpq
brew link --force libpq
sudo ln -s /opt/homebrew/lib/libpq.5.dylib /usr/local/lib
```

On Linux it is sufficient to just install the client libraries, but due to a bug in the reproducable build system some other build dependencies are required:

```
# ubuntu/debian
sudo apt-get install libpq-dev libssl-dev uuid-dev
```

Note: If you previously tried to build `webcashd` or its dependencies (e.g. for the server benchmarks or tests), you will need to clean the bazel build cache with `bazel clean` after installing the above build dependencies.  This is because the `drogon` web server detects PostgreSQL support during build time, and the installation of the missing dependencies which should trigger a rebuild isn't picked up by the bazel build system.

In addition you need a PostgreSQL instance for the server to talk to.  To make setup easier, a docker-compose configuration is provided that sets up PostgreSQL configured the way we need it.  Simply install [Docker Desktop](https://docker.com/get-started/) and run the following from the base directory of the repository:

```
docker compose -f stack.yml up
```

(If you run into a permissions problem, you may need to add your user to the `docker` group first.)

This command will not terminate; the `docker-compose` command will continue to run showing you the database logs until you use <kbd>Ctrl</kbd>+<kbd>C</kbd> to stop the container.

Once the docker container is up and running (you'll see a log output that says `listening on IPv4 address "0.0.0.0", port 5432`), you can open another terminal window to run the webcash server and/or tests.

To run the tests:

```
bazel test -c opt $(bazel query //...)
```

To run the webcash server:

```
bazel build -c opt webcashd
bazel-bin/webcashd
```

# License

This repository and its source code is distributed under the terms of the Mozilla Public License 2.0.  See MPL-2.0.txt.
