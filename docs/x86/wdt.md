# Watchdog Timer

## Watchdog Timers: The Silent Guardian of Embedded Systems

A watchdog timer is a hardware- or firmware-based countdown timer that monitors whether a system is still alive. It is used to detect abnormal system behavior and take corrective action to bring the system back to a working state. In embedded systems — where a stuck CPU can mean a crashed drone, a frozen pacemaker, or a bricked satellite — the watchdog is often the last line of defense between a transient fault and a catastrophic failure.

## How it works

At its core, a watchdog timer is a programmable counter driven by an independent clock source. Once started, it counts down toward a preset timeout value. The firmware running on the main processor must periodically refresh, or "kick," the counter before it expires. As long as the kicks arrive on schedule, the system is presumed healthy. If a kick is missed — because the processor is stuck in an infinite loop, has deadlocked, or has been corrupted by a hardware glitch — the counter reaches zero and the watchdog forces a system reset or jumps to a recovery handler. Crucially, the watchdog's clock is independent of the main system clock, so even a frozen processor cannot prevent the timer from firing.

## Hardware vs. firmware watchdogs

Hardware watchdogs are dedicated circuits — either built into the processor or provided as separate supervisor chips on the board. They are immune to software corruption and typically cannot be disabled once enabled, which is exactly what you want in safety-critical designs. Firmware watchdogs, by contrast, are implemented as software timers within the operating system or scheduler. They are flexible and can monitor individual tasks, but they share fate with the processor they are supposed to be guarding, which limits their usefulness against hard hangs.

## Simple vs. windowed watchdogs
A simple watchdog only enforces an upper bound: kick it before timeout, or be reset. A windowed watchdog enforces both a lower and upper bound — kick too early and you also get reset. This catches a surprisingly nasty failure mode: a runaway loop that happens to include the kick instruction will keep a simple watchdog happy forever. Windowed watchdogs are standard in automotive and other safety-critical systems for exactly this reason.

## Implementation best practices
The most common mistake is kicking the watchdog from a periodic interrupt. This guarantees the dog is fed even if the main application has died — defeating the entire point. A better pattern is to have each critical task set an "I'm alive" flag, and a single supervisor task that kicks the watchdog only when all flags are set, then clears them. This way, a single hung task brings down the system, and the watchdog does its job.
Other guidelines worth following: never disable the watchdog in production builds, even temporarily; choose a timeout long enough to cover your worst-case processing delays but short enough to meet your reliability budget; and always log the reset cause on boot so field failures can be diagnosed later. Most processors distinguish between power-on resets, brown-out resets, and watchdog resets — use that information.

## A famous save

The canonical watchdog success story is the Mars Pathfinder mission in 1997. A timing bug caused the lander's software to hang repeatedly on the Martian surface. Each time, the onboard watchdog triggered a reset, keeping the mission alive long enough for engineers on Earth to diagnose the problem and upload a fix. Without the watchdog, Pathfinder would have been a very expensive paperweight on another planet.

## The bottom line
A watchdog timer is cheap, simple, and one of the highest-leverage reliability mechanisms available to an embedded engineer. It does not prevent bugs — but it ensures that when bugs slip through, the system recovers automatically instead of staying dead. In any design where field reliability matters, the question is not whether to use a watchdog, but how to architect the kick strategy so it actually catches the failures you care about.
