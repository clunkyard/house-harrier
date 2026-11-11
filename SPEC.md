# Vane language specification

Vane is a reactive language for bots whose tactics can be stated as ordered
responses. Every decision evaluates the current sensor observation, then lets
higher-priority rules claim controls before lower-priority rules fill the
remaining controls.

That split is useful when aiming, movement, and emergencies should remain
independent. A close-range evasion rule can take `drive` and `turn`, an
engagement rule can take aim and fire, and a search rule can still supply the
scanner motion.

## First bot

```vane
bot First Vane
modes 2d 3d

rule target priority 100 when contact
  status-text "TRACK"
  status-value range
  aim-heading bearing
  aim-pitch elevation
  scan-heading bearing
  scan-pitch elevation
  fire cooldown == 0 and abs(aim-error) < 3 and battery >= shot-cost-base + shot-cost-per-power + 5
end

rule roam priority 10 when true
  status-text "SEARCH"
  status-value tick
  drive 0.7
  turn 0.25
  scan-heading sweep(8)
  scan-pitch wave(30, 22)
  scan-arc 58
end
```

The same program is included as [minimal.vane](minimal.vane). Run it from the
Arena repository:

```sh
node src/cli.js match --mode 2d --ticks 300 --seed 4101 \
  --bot 'First Vane=node src/dsl/runner.js vane bots/dsl/vane/minimal.vane' \
  --bot dsl-cadence
```

Add `--serve` to watch the match. To make a source appear in Match and Series
setup, add a `bot.json` beside it as shown in the [DSL
guide](../../../docs/dsls.md#register-your-source).

This bot fires at power 1 once the turret, which turns at 220° per second, is
within 3° of the target and the battery can pay for the shot with 5 to spare.
The new-robot starter,
[src/public/starters/vane/bot.vane](../../../src/public/starters/vane/bot.vane),
also picks shot power by range and keeps a larger reserve for driving.

## Rule arbitration

Rules are sorted by descending numeric priority. Rules with the same priority
keep source order. On each decision tick:

1. Vane evaluates every rule condition in that order.
2. A true rule runs its action lines.
3. The first action to write a field owns that field for the tick.
4. Later rules may write fields that are still unclaimed.

Ownership is per field, not per rule. In the example, `target` owns aim, scan,
fire, and status while contact exists. `roam` still provides movement because
`target` does not write `drive` or `turn`.

Each tick starts with zero movement, no fire request and shot power 1. Aim and
scanner targets start at their current values. Give movement a low-priority
fallback rule if the bot should keep moving when no tactical rule matches.

[harrier.vane](harrier.vane) shows a three-layer tactic: evade at priority
120, engage at 100, and search at 10. [inertia-brake.vane](inertia-brake.vane)
uses the same arbitration with force controls in the 2D inertial ruleset.

[layered-guard.vane](layered-guard.vane) combines boundary steering, a battery
fire veto and independent targeting. The [workshop](../../../docs/dsl-workshop.md#vane-combine-policies-without-copying-targeting)
walks through the controls each rule owns and a real local match.

## File structure

Directives and rules are line-oriented. Keywords are case-insensitive. `#`
starts a comment outside a string.

```text
bot DISPLAY NAME
modes 2d 3d
rulesets classic inertial-2d gravity-flat gravity-hills zero-g

rule NAME priority NUMBER when EXPRESSION
  ACTION EXPRESSION
  ...
end
```

- `bot` sets the spectator name. It is optional.
- `modes` accepts `2d`, `3d`, or both. The default is both.
- `rulesets` lists accepted motion rules. The default is every current
  ruleset.
- A program needs at least one rule.
- A rule name is a label for diagnostics; priorities control execution.

The manifest beside the source must advertise compatible modes and rulesets.
The protocol runner rejects a mismatch during startup.

## Expressions

Vane expressions contain finite numbers, quoted strings, booleans, sensor
names, parentheses, operators, and functions.

| Precedence, high to low | Operators |
| --- | --- |
| Unary | `-`, `not` |
| Multiplication | `*`, `/`, `%` |
| Addition | `+`, `-` |
| Comparison | `=`, `==`, `!=`, `<`, `<=`, `>`, `>=` |
| Boolean | `and`, then `or` |

`and` and `or` short-circuit. Division and remainder by zero are runtime
errors. Strings may use single or double quotes and the escapes `\n`, `\r`,
and `\t`.

| Function | Result |
| --- | --- |
| `min(a, b, ...)`, `max(a, b, ...)` | numeric minimum or maximum |
| `abs(x)` | absolute value |
| `clamp(x, low, high)` | `x` limited to the interval |
| `sin(degrees)`, `cos(degrees)` | trigonometry in degrees |
| `wave(period, amplitude)` | sine wave based on the match tick |
| `sweep(degreesPerTick)` | heading plus a wrapped tick-based sweep |
| `angle(target, current)` | shortest signed angular difference |

## Sensor values

The language receives the same private observation as a protocol bot. It does
not receive unscanned opponent state.

| Area | Values |
| --- | --- |
| Match | `tick`, `time`, `remaining`, `mode3`, `ruleset` |
| Own robot | `health`, `cooldown`, `x`, `y`, `z`, `heading`, `pitch`, `aim`, `aim-pitch` |
| Battery | `battery`, `battery-capacity`, `battery-fraction` |
| Scanner | `scan-heading`, `scan-pitch`, `scan-arc`, `scan-range`, `contact`, `range`, `bearing`, `elevation`, `aim-error` |
| Shots | `power-min`, `power-max`, `shot-speed-base`, `shot-speed-per-power`, `shot-cost-base`, `shot-cost-per-power` |
| Safe zone | `zone-active`, `zone-radius`, `zone-distance`, `zone-bearing`, `zone-elevation` |
| Match mechanics | `storm-active`, `storm-distance`, `storm-bearing`, `storm-heading`, `storm-radius`, `cue-present`, `cue-bearing`, `cue-elevation`, `cue-range`, `cue-error`, `cue-age`, `cue-muzzle`, `pickup-present`, `pickup-bearing`, `pickup-elevation`, `pickup-range`, `hill-active`, `on-hill`, `hill-held`, `hill-bearing`, `hill-elevation`, `hill-range`, `hill-radius`, `hill-time`, `hill-to-win`, `score`, `score-best` |
| Arena | `width`, `height`, `depth`, `terrain-kind` |
| Physics flags | `physics-mode`, `inertial`, `gravity`, `zero-g`, `hills`, `grounded` |
| Motion | `vx`, `vy`, `vz`, `speed`, `yaw-rate`, `pitch-rate`, `roll-rate` |
| Ground and gravity | `altitude`, `terrain-height`, `gravity-x`, `gravity-y`, `gravity-z` |
| Attitude basis | `forward-x/y/z`, `right-x/y/z`, `up-x/y/z` |

`contact` describes whether the scanner returned an echo this tick. `range`,
`bearing`, and `elevation` select the nearest echo. Without an echo, `range`
is a very large number, `bearing` is the robot heading, and `elevation` is
zero. `aim-error` is the signed heading difference from where the turret
points now to that echo, and 0 without one; in 3D, also compare `elevation`
with `aim-pitch`.

`cooldown` is 0 when the gun can fire this tick. `aim` and `aim-pitch` report
where the turret points now, which can lag behind the last request.
`scan-range` is the reach of the current scan arc. A shot costs `shot-cost-base +
shot-cost-per-power * power` battery and flies at `shot-speed-base +
shot-speed-per-power * power`. Before sudden death, `zone-active` is false,
the zone distance and radius are 0, and the zone bearing is the robot heading.
In classic, `speed` is the smoothed speed that aim spread uses. The [DSL
guide](../../../docs/dsls.md#shared-observations-and-actions) lists every
value, and the [protocol guide](../../../docs/protocol.md) defines each one.

The match mechanics' values read as inactive while their mechanic is off or
not yet in play: flags false, ranges and distances a very large number,
bearings the robot heading. `storm-distance` is to the storm's edge and
negative inside it. `cue-*` describe the best passive cue, remembered for 60
ticks where it placed a robot and seen from where the robot is now. The [DSL
guide](../../../docs/dsls.md#shared-observations-and-actions) defines each
one.

## Actions

An action line is an action name followed by an expression. `fire` alone is
short for `fire true`.

| Action | Request |
| --- | --- |
| `drive`, `turn`, `pitch` | Classic forward, yaw, and pitch controls |
| `thrust`, `strafe`, `lift` | Physics linear forces |
| `yaw-torque`, `pitch-torque`, `roll-torque` | Physics angular forces |
| `brake` | Physics braking assist, 0 to 1 |
| `aim-heading`, `aim-pitch` | absolute bearing and elevation the turret turns toward |
| `scan-heading`, `scan-pitch`, `scan-arc` | absolute scanner direction and cone width |
| `fire` | fire this tick if `cooldown` is 0 and the battery can pay |
| `power` | shot power, 0.5 to 3; 1 when no rule writes it |
| `status-text`, `status-value` | spectator text and finite numeric value |

The engine clamps numeric controls to their legal ranges. `brake` takes a
strength: a number is clamped to 0 through 1, and a condition brakes fully
when it holds, so `brake speed > 70` and `brake 0.3` both work. The scanner
turns at once; the turret turns at up to 220° per second. A shot the battery
cannot pay for is refused at no cost. Classic movement actions remain accepted
in physics matches as convenient force aliases. An explicit force action owns
its corresponding axis.

## Work limits and errors

Without a supplied budget, the local Vane interpreter allows 512 operations
per decision. Hosted DSL decisions and browser practice use 1024. The local
CLI with `--resources dsl` defaults to 768; add `--dsl-ops 1024` to use the
hosted operation limit. A supplied `dslOpsPerTurn` replaces the interpreter
default. Expression nodes and action execution consume operations, and the
exact count is reported with each decision.

Parsing and runtime failures include `file:line:column`. Unknown actions or
values, non-finite action results, division by zero, and an exhausted work
budget forfeit the bot without stopping the match.

Vane has no persistent variables. Use [Cadence](../cadence/SPEC.md) when a
tactic needs phases or counters, or [Glyph](../glyph/SPEC.md) when named stack
compositions fit the tactic better.

## Compile to WASM

The build needs Clang, a C compiler and the Wasmtime C API; see the
[WebAssembly guide](../../../docs/sdk/wasm.md). Run from the repository root:

```sh
scripts/build-dsl-wasm.sh vane
cat > build/vane-wasm.json <<'JSON'
{
  "name": "Wasm Vane",
  "language": "vane-wasm",
  "protocol": 2,
  "supportedModes": ["2d", "3d"],
  "resourceModes": ["wasm"],
  "command": ["build/clunkyard-wasm-host", "build/wasm-harrier-vane.wasm"]
}
JSON
node src/cli.js match --resources wasm \
  --bot @build/vane-wasm.json --bot c --ticks 300
```

The compiler lowers parsed rules and first-writer arbitration to a module on
WebAssembly ABI 4: it reads every value through the robot core's `dsl_fill_v1`
import once a turn and sets each control through the core's setters. Wasmtime
meters the resulting instructions and the imports' prices. The interpreter
remains available as `dsl-vane`. Numeric `status-value` works in the module. Text
`status-text` reports a compile error with its source location; run that
program with the interpreter when text is needed.

## Grammar

```text
program      := directive* rule+
directive    := "bot" words
              | "modes" ("2d" | "3d")+
              | "rulesets" ruleset+
rule         := "rule" word "priority" number "when" expression newline
                action* "end"
action       := action-name expression newline
              | "fire" newline
expression   := literal | sensor | function-call | unary | binary | "(" expression ")"
ruleset      := "classic" | "inertial-2d" | "gravity-flat"
              | "gravity-hills" | "zero-g"
```
