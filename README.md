# hsk-miner

CUDA-capable Cuckoo Cycle miner for Handshake.

This is a node.js module which hooks into [John Tromp][1]'s [many cuckoo cycle
miners][2].

## Building

Tromp's code is not redistributed with this package, so the build process is a
bit complex and involves cloning and preprocessing a bit of code.

``` bash
make
make testnet
```

## Usage

Simply point the miner at an HSK RPC server and you're off.

``` js
$ hsk-miner --rpc-host localhost --rpc-port 13037 --rpc-pass my-password
```

## CLI Usage

Another small utility is available for mining arbitrary cuckoo cycle data. This
is most useful for testing.

``` js
# Mine a single block
$ hsk-mine [header-hex] [target] [backend] -n [nonce] -r [range]
```

## API Usage

``` js
const miner = require('hsk-miner');

// Header without the appended solution.
const hdr = Buffer.alloc(164, 0x00);

if (miner.hasCUDA())
  console.log('Mining with cuda support!');

console.log('CUDA devices:');
console.log(miner.getDevices());

const [sol, nonce, match] = async miner.mineAsync(hdr, {
  backend: 'mean-cuda',
  nonce: 0,
  range: 0xffffffff,
  target: Buffer.alloc(32, 0xff),
  threads: 100,
  trims: 256,
  device: 0
});

if (!sol) {
  console.log('No solution found for nonce range!');
  // At this point we would increment the other 16
  // bytes of the header's nonce and try again.
  return;
}

if (!match) {
  console.log('A solution was found, but it did not meet the target.');
  const hash = miner.blake2b(sol);
  const diff = miner.toDifficulty(hash);
  console.log('Best share: %d (%d).', diff, nonce);
  // Now we should try again by changing
  // the first 16 bytes of the header nonce.
  return;
}

console.log('Solution:');
console.log(miner.toArray(sol));
console.log('Nonce: %d', nonce);
console.log('Match: %s', match);

```

If `match` is not true, this means there was a solution (possibly many), but
they did not pass the target check. The returned solution and nonce will be the
ones which amount to the lowest hash found. This is useful for submitting
shares to a mining pool.

Note that `threads` is a percentage (1-100) of Tromp's default parameters when
used with the `mean-cuda` backend. See `Notes` for more information.

## API

### Functions

- `miner.mine(hdr, options)` - Mine a to-be-solved header (sync).
- `miner.mineAsync(hdr, options)` - Mine a to-be-solved header (async).
- `miner.isRunning(device)` - Test whether a device is currently running.
- `miner.stop(device)` - Stop a running job.
- `miner.stopAll()` - Stop all running jobs.
- `miner.verify(hdr, sol, target?)` - Verify a to-be-solved header (sync).
- `miner.blake2b(data, enc)` - Hash a piece of data with blake2b.
- `miner.isCuda(backend)` - Test whether a backend is a CUDA backend.
- `miner.getEdgeBits()` - Get number of edge bits (compile time flag).
- `miner.getProofSize()` - Get proof size (compile time flag).
- `miner.getEase()` - Get ease (compile time flag).
- `miner.getEasiness()` - Get easiness (compile time flag).
- `miner.getNetwork()` - Get network (compile time flag).
- `miner.getBackends()` - Get available backends.
- `miner.hasBackend(name)` - Test whether a backend is supported.
- `miner.hasCUDA()` - Test whether CUDA support was built.
- `miner.hasDevice()` - Test whether a CUDA device is available.
- `miner.getDeviceCount()` - Get count of CUDA devices.
- `miner.getDevices()` - Get CUDA devices. Returns an array of objects.
- `miner.getCPUCount()` - Get count of CPUs.
- `miner.getCPUs()` - Get CPUs. Returns an array of objects.
- `miner.getBackend()` - Get suggested backend based on the machine's hardware.

## Utilities

- `miner.toArray(buf)` - Convert buffer solution to a uint32 array.
- `miner.toBuffer(arr)` - Convert uint32 array solution to a buffer.
- `miner.toBits(target)` - Convert a big endian target to a mantissa.
- `miner.toTarget(bits)` - Convert mantissa to big endian target.
- `miner.toDouble(target)` - Convert a big endian target to a double.
- `miner.toDifficulty(target)` - Convert target/hash to a difficulty/share.

### Constants

- `miner.EDGE_BITS` - Edge bits (compile time flag).
- `miner.PROOF_SIZE` - Proof size (compile time flag).
- `miner.EASE` - Ease (compile time flag).
- `miner.EASINESS` - Easiness (compile time flag).
- `miner.NETWORK` - Network (compile time flag).
- `miner.BACKENDS` - Available backends.
- `miner.HAS_CUDA` - Whether CUDA support was built.
- `miner.BACKEND` - Default backend.
- `miner.TARGET` - Default target.
- `miner.HDR_SIZE` - Handshake to-be-solved header size (164).
- `miner.NONCE_SIZE` - Total size of nonce (20).
- `miner.NONCE_START` - Start of nonce position (144).
- `miner.NONCE_END` - End of extra nonce position (160).

## Options

- `backend` - Name of the desired backend.
- `nonce` - 32 bit nonce, during mining, this will alter the last 4 bytes of
  the header (serialized as little endian), the front 16 bytes will stay
  unchanged.
- `range` - How many iterations before the miner stops looking.
- `target` - Big-endian target (32 bytes).
- `threads` - Backend-specific, see below.
- `trims` - Backend-specific, see below.

## Backends (so far)

- mean-cuda - Mean miner with CUDA (`mean_miner.cu`).
- lean-cuda - Lean miner with CUDA (`lean_miner.cu`).
- mean - Mean miner in C++ (`mean_miner.cpp`).
- lean - Lean miner in C++ (`lean_miner.cpp`).
- simple - Simple miner, mostly for testing (`simple_miner.cpp`).

## Platform Support

The code has only been tested on Linux. Realistically, linux is the only thing
that should matter for mining, but build support for other OSes is welcome via
pull request.

Supports 64bit only right now.

## Todo

- Stratum support.
- Improve speed of job stoppage.

## Notes

When using one of the CUDA backends, only one job per device is allowed at a
time. This is done because the CUDA miners generally wipe out your GPU's memory
anyway.

The `threads` and `trims` parameters are backend-specific. You may have to dig
into each one to figure out what reasonable values are and how they work.

For CUDA support, CUDA must be installed in either `/opt/cuda` or
`/usr/local/cuda` when running the build scripts.

## Contribution and License Agreement

If you contribute code to this project, you are implicitly allowing your code
to be distributed under the MIT license. You are also implicitly verifying that
all code is your original work. `</legalese>`

## License

- Copyright (c) 2018, Christopher Jeffrey (MIT License).

See LICENSE for more info.

[1]: https://github.com/tromp
[2]: https://github.com/tromp/cuckoo