
ARM SiP Service
===============

This document enumerates and describes the ARM SiP (Silicon Provider) services.

SiP services are non-standard, platform-specific services offered by the silicon
implementer or platform provider. They are accessed via. `SMC` ("SMC calls")
instruction executed from Exception Levels below EL3. SMC calls for SiP
services:

* Follow [SMC Calling Convention][SMCCC];
* Use SMC function IDs that fall in the SiP range, which are `0xc2000000` -
  `0xc200ffff` for 64-bit calls, and `0x82000000` - `0x8200ffff` for 32-bit
  calls.

The ARM SiP implementation offers the following services:

* Performance Measurement Framework (PMF)
* Execution State Switching service

Source definitions for ARM SiP service are located in the `arm_sip_svc.h` header
file.

Performance Measurement Framework (PMF)
---------------------------------------

The [Performance Measurement Framework](./firmware-design.md#13--performance-measurement-framework)
allows callers to retrieve timestamps captured at various paths in ARM Trusted
Firmware execution. It's described in detail in [Firmware Design document][Firmware Design].

Execution State Switching service
---------------------------------

Execution State Switching service provides a mechanism for a non-secure lower
Exception Level (either EL2, or NS EL1 if EL2 isn't implemented) to request to
switch its execution state (a.k.a. Register Width), either from AArch64 to
AArch32, or from AArch32 to AArch64, for the calling CPU. This service is only
available when ARM Trusted Firmware is built for AArch64 (i.e. when build option
`ARCH` is set to `aarch64`).

### `ARM_SIP_SVC_EXE_STATE_SWITCH`

    Arguments:
        uint32_t Function ID
        uint32_t PC hi
        uint32_t PC lo
        uint32_t Cookie hi
        uint32_t Cookie lo

    Return:
        uint32_t

The function ID parameter must be `0x82000020`. It uniquely identifies the
Execution State Switching service being requested.

The parameters _PC hi_ and _PC lo_ defines upper and lower words, respectively,
of the entry point (physical address) at which execution should start, after
Execution State has been switched. When calling from AArch64, _PC hi_ must be 0.

When execution starts at the supplied entry point after Execution State has been
switched, the parameters _Cookie hi_ and _Cookie lo_ are passed in CPU registers
0 and 1, respectively. When calling from AArch64, _Cookie hi_ must be 0.

This call can only be made on the primary CPU, before any secondaries were
brought up with `CPU_ON` PSCI call. Otherwise, the call will always fail.

The effect of switching execution state is as if the Exception Level were
entered for the first time, following power on. This means CPU registers that
have a defined reset value by the Architecture will assume that value. Other
registers should not be expected to hold their values before the call was made.
CPU endianness, however, is preserved from the previous execution state. Note
that this switches the execution state of the calling CPU only. This is not a
substitute for PSCI `SYSTEM_RESET`.

The service may return the following error codes:

  - `STATE_SW_E_PARAM`: If any of the parameters were deemed invalid for
    a specific request.
  - `STATE_SW_E_DENIED`: If the call is not successful, or when ARM Trusted
    Firmware is built for AArch32.

If the call is successful, the caller wouldn't observe the SMC returning.
Instead, execution starts at the supplied entry point, with the CPU registers 0
and 1 populated with the supplied _Cookie hi_ and _Cookie lo_ values,
respectively.

- - - - - - - - - - - - - - - - - - - - - - - - - -

[Firmware Design]: ./firmware-design.md
[SMCCC]: http://infocenter.arm.com/help/topic/com.arm.doc.den0028a/index.html "SMC Calling Convention PDD (ARM DEN 0028A)"
