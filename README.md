# Arcminer

Native Windows-focused C++20 / CUDA GPU cryptocurrency miner project.

CUDA search kernel is used by default on the tested RTX 3060 Ti 
(SM 86) and falls back to the original Arcminer path if unavailable. 
Set `ARCMINER_FUSED=0` in the process environment to disable it, 
or `ARCMINER_FUSED=1` to test it on another GPU. Candidate hashes 
are independently checked on the CPU before submission.

## Algorithms

**Available in Arcminer today: KawPoW for Ravencoin (RVN), experimental
MeowPow for Meowcoin (MEWC), and experimental Etchash for Ethereum Classic
(ETC) mainnet.**
MeowPow CUDA mining for Meowcoin (MEWC) is experimental. Its CPU verifier
matches three Meowcoin Core hashes, the CUDA worker matches those vectors,
and mock pool tests cover share submission and the pool's target updates.
An independently CPU-verified MEWC user share was accepted by Cedric's live
pool at height 2128544 on 2026-10-27. The 2Miners ETC account API reported
three valid shares for Arcminer's `arcminer-test` worker on 2026-10-28, with
zero stale or invalid shares. The remaining algorithms are planned and their
names are not accepted by the current `--algo` option.
Coin tickers in the table are examples from
not claims that Arcminer supports those coins or their pool protocols. Each
coin/network needs its own implementation and verification before release.

| Algorithm | Coin examples in Rigel | Arcminer status |
| --- | --- | --- |
| `abelian` | ABEL | Planned |
| `alephium` | ALPH | Planned |
| `autolykos2` | ERG | CPU/GPU reference hits and read-only 2Miners sessions tested; mining not enabled |
| `etchash` | ETC | Experimental CUDA mining; CPU-verified mainnet shares accepted |
| `ethash` | ETHW, XPB, OCTA | Planned |
| `fishhash` | IRON | Planned |
| `karlsenhashv2` | KLS | Planned |
| `kawpow` | RVN, XNA, QUAI, NEOX | **RVN mainnet available**; other coins planned |
| `meowpow` | MEWC | Experimental CUDA mining; CPU-verified mainnet share accepted |
| `nexapow` | NEXA | Planned |
| `octopus` | CFX | Planned |
| `progpowz` | ZANO | Planned |
| `sha256ton` | GRAM | Planned |
| `sha3x` | XTM | Planned |
| `sha512256d` | RXD | Planned |
| `xelishashv3` | XEL | Planned |
| `zil` | ZIL | Planned |

Arcminer also plans **ZelHash** after Etchash and Autolykos v2. It is on
[Arcminer's roadmap](docs/ROADMAP.md), but it is not in Rigel's current
algorithm list. Its current coin/network requirements still need verification.
No release dates are set for planned algorithms.

The Autolykos v2 work validates Ergo's height-dependent table size and a
published CPU PoW hit, checks the supplied user and developer payout addresses,
parses live 2Miners jobs and subscription nonce spaces, and CPU-gates a
mock-tested share request through separate account-bound sessions. A bounded
CUDA diagnostic matches the published Ergo hit, searches short nonce ranges,
and CPU-verifies returned candidates. It also builds bounded GPU table chunks
that match the CPU reference at tested height and index boundaries. A bounded
GPU table window can be grown between batches in interruptible chunks and
used for diagnostic hashes. Its capacity follows an explicit byte budget and
leaves a 256 MiB GPU memory reserve. It has no
full-height dataset or measured mining throughput. A bounded GPU-filtered
search CPU-verifies candidates, and a job-bound nonce cursor reserves distinct
suffixes across consecutive batches. The batch runner measures CUDA kernel
time and routes diagnostic work through the persisted 1% active-time fee
ledger. Two local mock pool sessions accepted an independently CPU-verified
ERG GPU candidate on only the selected account route. The bounded diagnostic
loop also submits shares through each route to local mock pools, polls for
new jobs between GPU batches, reconnects a dropped pool session with new work
and nonce space, and persists measured active time. It is not an
efficient production search. See
[protocol notes](docs/AUTOLYKOS2_PROTOCOL.md).
It does not accept an ERG mining command or submit live shares.

The Etchash implementation uses a separate ETC mainnet CPU verifier. It
checks the ECIP-1099 transition at block 11,700,000: dataset sizing switches
to 60,000-block epochs, and each epoch uses a seed reached by two steps on
the original 30,000-block seed chain.
The verifier checks complete mix and target values against a pinned Apache-2.0
reference, a published Ethash vector, and four ETC mainnet block
vectors at heights 11,699,999, 11,700,000, 11,730,000, and 11,760,000. The ETC
reference's exported symbols are isolated from KawPoW and both CPU verifiers
pass together in one process. A standalone ETC address checker validates
`0x` plus 40 hexadecimal digits, ERC-55 mixed-case checksums, and rejects the
zero payout address. Hex addresses do not identify a chain or prove wallet
ownership. A bounded parser validates three- and four-field ETC eth-proxy
work (including a live 2Miners block height) and its post-fork seed. The
ETC developer account is disclosed in [protocol notes](docs/ETCHASH_PROTOCOL.md).
Use `--check-config --coin etc --pool HOST:PORT --wallet YOUR_ETC_ADDRESS`
for offline validation, or replace `--check-config` with `--probe-pool` for a
read-only eth-proxy login and work check. A mock pool covers login success,
rejection, and invalid seed. `--mine --coin etc` starts explicit foreground
Etchash hashing with a separate user wallet and the disclosed developer session.
A persistent per-account Etchash session now handles job replacement,
reconnection, and CPU-verified share submission in mock pool tests. The
read-only probe uses this session. A CUDA worker matches
the CPU mix and final hashes at ETC heights 11,700,000, 11,760,000, and
25,622,670 on the tested GPU. It generates DAG items from the light cache on
demand for correctness tests. The mining path builds a full GPU DAG and searches
up to 1,048,576 consecutive nonces per batch, filtering the complete target on
the GPU. Tests compare full-DAG candidates, target filtering, and overflow
handling with the independent CPU reference. The pool session verifies each
candidate again on the CPU before submission. A two-minute RTX 3060 Ti run
measured about 13.3 MH/s wall-clock and 15.4 MH/s of GPU kernel time; these are
observations on that device, not promises for other GPUs. Separate user and
developer mock sessions pass fee-routing tests, and the live 2Miners account
reported three valid user shares. See [the ETC pool test](docs/REAL_ETC_POOL_TEST.md).
`--algorithm` is now an alias for `--algo`.

### Meowcoin preparation

Use `--check-config --coin mewc --pool HOST:PORT --wallet YOUR_MEWC_ADDRESS`
to validate a Meowcoin mainnet legacy Base58Check address and show a redacted
configuration summary. `--coin mewc` selects `meowpow` unless `--algo` is
explicitly set. JSON uses `"algorithm": "meowpow"` and
`"network": "meowcoin-mainnet"`. Arcminer's disclosed MEWC developer payout
address is `MGPRFHxNNpy3gXJ1rzrKZKZdggAa6rzisS`; its configured plain-TCP
pool is `meowcoin.cedric-crispin.com:4284`. The user chooses their own pool and
wallet. Meowcoin's published mainnet legacy address versions are 50 (pubkey)
and 122 (script). The address checker does not yet accept Bech32 addresses.
`--probe-pool --coin mewc --pool HOST:PORT --wallet YOUR_MEWC_ADDRESS` validates
one pool job without hashing or submitting a share. To run experimental mining:

```powershell
.\build\windows-vs\Release\arcminer.exe --mine --coin mewc --pool meowcoin.cedric-crispin.com:4284 --wallet YOUR_MEWC_ADDRESS --worker ArcWorker1 --password x,d=0.0128
```

The user wallet must differ from the fixed developer wallet. The password
requests the pool's published minimum difficulty for testing; omit it to use
the default `x`. The password is also accepted as `user.password` in JSON
configuration and is omitted from the redacted configuration summary.

WoolyPooly is also supported for the user session. Its live endpoint was
probed, its MeowPow job validated, and a short foreground run authorized and
hashed on the RTX 3060 Ti. Use
`--pool pool.woolypooly.com:3116 --password x` with the same wallet and worker.
Its observed initial share target was harder than Cedric's minimum; no
WoolyPooly share was found during the short check. The fixed developer session
still uses the disclosed Cedric endpoint.

The [Meowcoin mining page](https://www.mewccrypto.com/mining) identifies
MeowPow as its GPU algorithm. The [Meowpowminer source](https://github.com/Meowcoin-Foundation/meowpowminer)
uses a 7,500-block epoch, six-block program period, 16 registers, six cache
accesses, and nine math operations per round. Its README's 12,000-block epoch
statement conflicts with its source and current Meowcoin Core. Arcminer uses
the [pinned MeowPow CPU reference](docs/THIRD_PARTY.md), corrected for Core's
epoch-110 DAG size transition. The CPU and CUDA outputs match Core at heights 0,
170915, and 825000. The KawPoW GPU path is not used for MEWC. A
read-only probe of the Cedric pool authorized and validated a live job at
height 2128259 on 2026-10-27. Its standard Stratum V1 subscription shape is
covered by a mock test. Mock pool share submission and acceptance pass with
independent CPU verification. A live CPU-verified user share was accepted by
Cedric's pool at height 2128544. WoolyPooly's alternate subscription form is
also mock tested and was validated by a live read only probe.

## Dual mining roadmap

**Arcminer does not support dual or triple mining yet.** The table records the
combinations listed in [Rigel's dual-mining table](https://github.com/rigelminer/rigel#dual-mining)
as future Arcminer targets. A check means *planned for evaluation*, not a
working Arcminer command or a promise that a particular coin pair will remain
mineable. Each combination needs GPU scheduling, independent share verification,
separate pool/account routing, and real-pool testing.

| Primary algorithm | `+alephium` | `+sha256ton` | `+sha3x` | `+sha512256d` |
| --- | :---: | :---: | :---: | :---: |
| `abelian` | ✓ | ✓ | ✓ | ✓ |
| `autolykos2` | ✓ | ✓ | ✓ | ✓ |
| `ethash` | ✓ | ✓ | — | ✓ |
| `etchash` | ✓ | ✓ | — | ✓ |
| `fishhash` | ✓ | ✓ | ✓ | ✓ |
| `karlsenhashv2` | — | ✓ | ✓ | — |
| `octopus` | ✓ | ✓ | ✓ | ✓ |
 
The **1% Arcminer developer share is measured in active hashing time**, not
user rewards; planned algorithms and dual mining will need to retain that
disclosed accounting policy.

## Current state

Arcminer is an **experimental KawPoW miner for one Windows CUDA GPU**, not a
production-ready multi-algorithm miner. RavenMiner accepted a CPU-verified GPU
share on 2026-10-25. In a short RTX 3060 Ti run on 2026-10-26, the fused search
path received 24 accepted user shares and zero rejects; its completed rate was
about 19.5–20.6 MH/s. These are observed short-run results, not a hashrate
promise. The GPU results match three published Ravencoin vectors, and a mock
pool has accepted verified shares through both user and developer sessions.
See [real-pool evidence](docs/REAL_POOL_TEST.md) and the
[performance history](docs/KAWPOW_PERFORMANCE.md) for details.

On the tested SM 8.6 GPU, the default fused kernel comes from pinned GPLv3
kawpowminer source and is compiled by NVRTC at runtime. Arcminer retains its
own period-specialized and generic fallbacks. The Windows ZIP includes matched
NVRTC DLLs. Set `ARCMINER_FUSED=0` to use an original Arcminer path, or
`ARCMINER_FUSED=1` to test the fused path on another GPU architecture.

### Running on another Windows PC

The CUDA Toolkit and Visual Studio are needed only on the **build PC**.
Arcminer links the CUDA runtime statically, so a target PC does not need the
Toolkit or a separate `cudart` DLL. The miner still requires
a supported NVIDIA GPU and an NVIDIA driver compatible with the CUDA version
used for its release. The regular Windows build also uses the Microsoft Visual
C++ runtime DLLs; a target PC needs the matching Visual C++ Redistributable.
The Windows release script below links that runtime statically and packages
the two NVRTC DLLs used by the fast KawPoW path.
Other libraries introduced in later milestones must be packaged or linked
appropriately. A ZIP package can be produced after the Release build:

```powershell
cpack --config build\windows-vs\CPackConfig.cmake -C Release -G ZIP -B build\package
```

The ZIP contains `bin/arcminer.exe`, matching NVRTC and builtins DLLs for the
experimental kernel, complete buildable Arcminer source and tests, the pinned
Apache-2.0 CPU reference source, this README, dependency notes, GNU GPLv3,
the Apache-2.0 license, and the CUDA Toolkit EULA. Extract it and run
`bin\arcminer.exe --help`; start with the command line or a private config
file as described below. No config, fee state, wallet, or test binary is included. This package
has been checked on the build PC after extraction, but not on a clean target
PC with only the driver and Visual C++ Redistributable.

The default Windows ZIP embeds native KawPoW kernels for SM 7.5, 8.6, 8.9,
and 12.0. The build PC's CUDA compiler must support all four targets; set
`ARCMINER_CUDA_ARCHITECTURES` at configure time to select a different set.
Only SM 8.6 has been run and verified on local hardware. The other kernels
compile into the ZIP but still need GPU and driver tests on target machines.
Use `bin\arcminer.exe --list-devices` to check a target GPU's SM version.

 
## Start from one command

From the extracted ZIP's `bin` directory, replace the wallet placeholder with
your own Ravencoin mainnet address:

```powershell
.\arcminer.exe --mine --algo kawpow --pool stratum+tcp://stratum.ravenminer.com:3800 --wallet YOUR_RVN_ADDRESS --worker W1
```

`--mine` is the explicit opt-in to foreground GPU hashing. The user pool and
wallet come from the command line. The separate developer session always uses
`stratum.ravenminer.com:3800`, payout address
`RREmTz9PHsKAKS7QbGh3aU7cZHUeBavMcC`, and worker `arcdev`. Arcminer validates both addresses,
prints the developer session and the 1% active GPU time policy, and stores
the fee ledger in `%LOCALAPPDATA%\ArcMiner\kawpow-ravencoin-mainnet.fee-state`.
The existing `ArcMiner` directory name is retained so earlier fee ledgers
remain available after the display-name change.
Preserve that file across runs. Add `--devices 0` to select a GPU; only one
GPU per process is supported. Press Ctrl+C to request a graceful stop: Arcminer
finishes the in-flight GPU batch, submits any verified shares, checkpoints its
active GPU time, closes both pool sessions, and exits. Closing the terminal
window or force-killing the process does not allow that drain. A command with pool and
wallet but without `--mine` will not start hashing.

The fused kernel runs by default on SM 8.6 for both one-line and config mining.
It compiles each new block period before its first fee-accounted batch. The
original period kernel remains a fallback; the generic kernel remains available
if specialization fails. To use the original generic kernel in PowerShell:

```powershell
$env:ARCMINER_PERIOD_JIT='0'
$env:ARCMINER_FUSED='0'
.\arcminer.exe --mine --algo kawpow --pool stratum+tcp://stratum.ravenminer.com:3800 --wallet YOUR_RVN_ADDRESS --worker W1
```

Remove both variables to restore the SM 8.6 default.

KawPoW, MeowPoW, and Etchash use the same foreground dashboard. ETC sessions
request fresh work every ten minutes while connected. KawPoW and MeowPoW
sessions with no pool messages for twenty minutes reconnect for a fresh job
before common pool idle timeouts; a transient reconnect may briefly pause
hashing. A zero developer share count is possible even while the 1% active-time
allocation is working because shares depend on pool difficulty. The dashboard clears the
console before its first render, redraws when the window changes size, and
fills the visible width and height with a compact layout in smaller windows.
The interactive view uses a separate terminal screen; on exit, Arcminer restores
the prompt and prints a concise final share and active-time summary.
Its panels use continuous Unicode box-drawing borders.
It shows user
and developer pool status, local and GPU-kernel hashrate, accepted/rejected shares,
active GPU time, current account, DAG cache, and available NVIDIA driver power,
temperature, fan, and clock readings. It fits the current terminal width and
highlights accepted shares, errors, and high temperatures. Set `NO_COLOR=1` or
`ARCMINER_COLOR=0` to disable color. NVIDIA telemetry is optional and appears
as `n/a` when unavailable. Consoles smaller than 33 columns by 8 rows show a
resize hint; redirected output uses flushed, timestamped status lines. The local
rate measures completed hashes per wall
second; the GPU-kernel rate excludes pool waits and fee-state checkpoints.
Neither is a pool-reported rate, which depends on accepted shares.
Set `ARCMINER_TIMING=1` before mining to show per-sample fee checkpoint,
submission, and GPU initial/mix/final/filter timing. The extra GPU events are
recorded only in this diagnostic mode.

## JSON configuration alternative

Create a JSON file with your own user pool and valid Ravencoin mainnet payout
address. The developer pool, payout, and worker are built into Arcminer:

```json
{
  "algorithm": "kawpow",
  "network": "ravencoin-mainnet",
  "user": {"host": "your-user-pool.example.com", "port": 3333,
           "wallet": "YOUR_VALID_RVN_ADDRESS", "worker": "rig1"},
  "devices": [0]
}
```

The address string above is a placeholder and will be rejected. A `developer`
field in a JSON config is rejected, even if it repeats the built-in values. Run
`arcminer --check-config --config config.json`. Optional `--algo`, `--pool
HOST:PORT` (or `--pool stratum+tcp://HOST:PORT`), `--wallet`, `--worker` and `--devices 0,1` override user settings
from the file. The checker validates the schema, ports, device indices, and
Ravencoin mainnet Base58Check payout addresses. It prints a redacted summary;
it does not contact pools or start mining. Hostnames must currently be DNS
names or IPv4 addresses. TLS and actual device availability checks belong to
later milestones. Keep config files
private because they contain payout addresses.

To check a live user-pool handshake and one KawPoW job without mining:

```powershell
.\build\windows-vs\Release\arcminer.exe --probe-pool --config config.json
```

This validates the user account and the built-in developer account, connects only the user pool,
checks the job's epoch seed and target syntax, prints the public share target,
then disconnects. It does not
hash, submit shares, or start the developer session.

To start explicit foreground hashing on one CUDA GPU with the JSON user settings,
run:

```powershell
.\build\windows-vs\Release\arcminer.exe --mine --config build\arcminer.local.json
```

For ETC mainnet, use your own ETC payout address and the regular pool:

```powershell
.\build\windows-vs\Release\arcminer.exe --mine --coin etc --pool etc.2miners.com:1010 --wallet YOUR_ETC_ADDRESS
```

The ETC miner builds a full GPU DAG and pauses hashing unless both the user
and developer pool sessions have valid work. Its fee ledger checkpoints every
measured GPU batch. A GPU needs enough free VRAM for the current DAG plus a
256 MiB reserve; the observed DAG was 4,439 MiB on 2026-10-28. ETC uses
plain TCP and one GPU per process.

`--algo`, `--pool`, `--wallet`, and `--worker` may override the user settings.
The developer account cannot be set or overridden from a config file or CLI.
Press Ctrl+C to stop. Mining
creates `FILE.fee-state` beside the config; preserve that file across runs.
The CLI prints the absolute state path and a 1% active GPU time disclosure.
For KawPoW and MeowPow, the fee state records a pending checkpoint before
hashing and settles measured active GPU time in groups about once per second.
It also settles before a pool wait and on clean exit.
The same account mines throughout each checkpoint group; a fee-account switch
may therefore lag by up to a group plus one GPU batch. If the process is
interrupted before a group is settled, restart charges a one-second estimate
and prints the cumulative estimated time. Keep the state file across runs.
An interrupted group may have used less or more than one second of actual GPU
time; exact crash-time GPU duration cannot be recovered from a state file.
The current run loop supports one GPU, plain TCP, and one pool endpoint per
account. A disconnected account reconnects with bounded exponential backoff
(500 ms to 15 seconds) and must receive a fresh job before hashing resumes.
Hashing pauses while either account is unavailable, so pool downtime does not
count toward active GPU hashing time. Ctrl+C is checked between GPU batches;
initial DAG cache preparation checks for Ctrl+C between 16 MiB chunks.

## Networking development state

The Windows network module supports a bounded TCP handshake for one documented
KawPoW Stratum dialect: subscribe, authorize, receive `mining.notify`, and
validate the job fields. It handles `mining.set_difficulty` and validates
`mining.set_target` notifications while using the target carried by each job.
It rejects malformed
or oversized frames. Local mock tests cover success, failed authorization, and
an invalid job. The executable's read-only `--probe-pool` command connected to
HeroMiners on 2026-10-25, authorized the user account and validated a live
KawPoW job at height 4596313, without submitting shares.
Another probe on 2026-10-25 observed a share target implying about 1.07 billion
hashes per accepted share on average. At the later measured ~24,000 H/s, that
would be about 12 hours of active hashing; pool targets and actual outcomes vary.
The same day, a separate RavenMiner static-difficulty 0.001 endpoint supplied a
valid KawPoW job with a target implying about 43 million hashes per share and
accepted one CPU-verified GPU share.
The module can submit a candidate from a job issued by that same session only
after independent CPU verification. A local mock pool test checks its wire
encoding and accepted/rejected responses. A second mock pool test exercises
the full GPU mining loop and both fee routes. It has no TLS or endpoint
failover. The mining-loop mock also checks
reconnects after either account drops before a job and after a share response
is lost.

## KawPoW CPU verification development state

The CPU module validates a job's header, epoch seed, height, and 256-bit target,
then independently checks a candidate nonce and mix digest. Tests compare
three published Ravencoin KawPoW vectors and reject altered mixes, invalid
seeds, and hashes above target. The live GPU worker hands each candidate to
this verifier before submission. RavenMiner's accepted share confirms target
and nonce encoding for that pool's current Stratum dialect.

The CUDA library computes all 64 KawPoW mix rounds and final hashes. Its tests
compare Keccak stages, 512 initial registers, dataset items at two epochs, and
full hashes at three published heights with the independent CPU reference.
The persistent 64 and 256 nonce batch tests check every lane against the CPU reference
and the mock pool checks the exact nonce at an inclusive target boundary, plus
no submission when the target is one step lower.
It keeps the light cache and bounded batch buffers on the GPU across jobs and
generates uncached DAG items on demand. The search kernel distributes each hash
over 16 CUDA threads in 256-thread blocks, stores the indexed mix in shared
memory, and prepares its block-period operation schedule once. Initial and
final Keccak stages run in separate kernels. When the full DAG is cached, a
specialized search kernel omits the on-demand dataset-generation path.
Production mining transfers only compact target matches; an overflow falls
back to a complete batch scan. The diagnostic path still returns every hash.
Production mining gives the GPU a consecutive nonce range, avoiding a large
host nonce buffer and transfer for each batch; diagnostic tests retain the
arbitrary-nonce path.
Production nonce batches adapt toward 250 ms of measured kernel time, up to
2,097,152 nonces, to reduce per-batch fee checkpoint overhead while leaving
more VRAM available alongside the full DAG on 8 GB GPUs. Published vector,
full-DAG CPU/GPU equality, target-boundary and overflow, and mock-pool tests
pass. A short uncontended full-DAG live run measured about 3.87–3.90 MH/s
completed work on an RTX 3060 Ti.
It attempts to cache the full DAG when
VRAM permits and reduces the allocation in 256 MiB steps if needed. On the
RTX 3060 Ti, an epoch 612 run cached all 5,919 MiB. The earlier serial kernel
reported 23,700–25,400 H/s with 256 nonce batches. Cache preparation takes time, uses
substantial VRAM, and is not
charged as active hashing time. A more efficient DAG strategy and parallel mix
layout remain needed for practical speed.

## Roadmap

1. Configuration parser and CLI validation (implemented as an offline checker).
2. Stratum transport and per-algorithm protocol adapters, TLS, reconnect.
3. Working CUDA KawPoW hashing and independently verified share submission.
4. Active hashing-time accounting, persistent fee state, transition-safe jobs.
5. Additional Etchash performance work; Autolykos v2 and ZelHash adapters and test vectors.
6. NVML monitoring, multi-GPU scheduling, performance benchmarking.
7. Windows release packaging and signed binaries.

## Fee policy

Official product target: **1% developer mining time**, explicitly shown.
The scheduler allocates 60 seconds per 6000 active seconds. The miner records
a pending batch before each GPU launch, then replaces it with measured GPU
kernel time in a checksummed, atomically replaced state file. A restart with
an unfinished batch charges a one-second estimate exactly once and reports
the cumulative estimate. The Windows checkpoint retries transient
sharing failures before stopping mining. It opens separate user/developer sessions and
routes each candidate only to the session that issued its job. A mock test
covers restart persistence, the fee transition, and both share routes.
The one-second recovery value is an estimate; exact time for an interrupted
kernel remains unknown, so crash accounting needs further validation before a
production release. Pool downtime pauses hashing rather than skipping a developer interval.
One percent of time does not guarantee one percent of revenue.

## Safety and transparency

No background startup, stealth mode, remote deployment, or hidden mining.
Only run on devices where the owner explicitly authorizes mining.
The project name may conflict with existing Arcminer projects; check before release.
