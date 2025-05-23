# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## Unreleased

### Added

- Payloads can now use the ELF format (but must still be built for a fixed address)
- New payload runtime functions `startCycleCounter`, `getCycleCounterValue`, `getMonitorAbiVersion`

## 0.6 - 2024-02-16

### Added

- New microbenchmark for FPGA latency

### Changed

- Timer period is now expressed using the `std::chrono::microseconds` data type
- A Xilinx BSP (board support package) is no longer necessary to compile Bmboot
- GoogleTest dependency has been made optional

### Fixed

- `add_bmboot_payload` now correctly adds the linker option `-specs=nosys.specs`

## 0.5 - 2024-01-30

### Added

- New payload runtime functions `getBuiltinTimerFrequency`, `getBuiltinTimerValue`
- Payloads now embed an explicit header. This enables validation of ABI compatibility, load address and program size.

### Fixed

- Better handling of high-traffic console output
- Fixed incorrect frequency of the Generic Timer (`CNTFRQ_EL0`) on ZCU102 Evaluation Kit

## 0.4 - 2023-12-12

### Added

- User of the Manager API can provide an argument to be passed to the payload
- Message Queues to communicate between Linux and bare metal

### Changed

- Console output now includes CPU number and time is indicated in seconds

### Fixed

- Better reliability of detection of running monitor

## 0.3 - 2023-11-06

### Added

- New function overload `toString(DomainIndex)`

### Changed

- The payload runtime function `startPeriodicInterrupt` has been split into `setupPeriodicInterrupt` and
  `startPeriodicInterrupt`

## 0.2 - 2023-10-30

### Added

- New function `disableInterruptHandling`

### Changed

- Interrupt callbacks are now of type `std::function` (backwards-compatible change)
- The payload runtime function `configureAndEnableInterrupt` has been replaced by `setupInterruptHandling` and
  `enableInterruptHandling`

## 0.1 - 2023-10-25

### Added

- First versioned release
