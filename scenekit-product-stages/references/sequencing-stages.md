# Sequencing Stages: Directing a Multi-Scene Reel

One stage is a performance; several stages cut together are a FILM, and the
film has failure modes none of the single-stage recipes cover. This file is
the direction handbook for chaining stages into one continuous, optionally
interactive sequence - a promo reel, an onboarding tour, a demo loop. Every
rule below was paid for.

## Contents

- One master clock, beats as data
- The clock must ride wall time
- Cuts land on motion, after the exit clears
- Pre-mount the next stage
- A stage may outlive its own cut
- Interactive beats: re-basing the timeline on the user
- Scripted triggers must survive fingers
- A recording lead
- Verifying a sequence

## One master clock, beats as data

Do not schedule a sequence as a pile of `DispatchQueue.asyncAfter` calls.
The moment the timeline must respond to anything - a user interaction, a
seek, a restart - scattered timers cannot be re-aimed, only leaked.

Model the script as data: a sorted array of `(time, action)` beats, an
action enum, a cursor, and one 30 Hz tick that fires everything due:

```swift
struct ReelBeat { let time: Double; let action: ReelAction }

while nextBeat < beats.count, beats[nextBeat].time <= reelTime {
    // Advance BEFORE performing: an action that re-bases the timeline
    // repoints the cursor, and a post-increment would stomp it and skip
    // the freshly placed first beat.
    let action = beats[nextBeat].action
    nextBeat += 1
    perform(action)
}
```

Actions mutate host state (scene switches, trigger tokens, bubble text);
the stages react through their normal trigger-diff plumbing. Restart and
seek become: rebuild the array, repoint the cursor, bump the remount key.

## The clock must ride wall time

Three clock implementations, two of them broken:

- `Timer.publish(...)` stored as an instance property of a SwiftUI view is
  RECREATED on every render and silently stalls. Store the publisher
  `static`.
- A `.task` sleep loop on the MainActor STARVES under SceneKit load; in
  practice the reel clock fell to a third of wall speed. The symptom is
  subtle and damning: every baked animation plays at true speed (Core
  Animation does not consult your clock), while the beats between them
  drift late - the gaps between scenes stretch on screen while each scene
  in isolation looks perfect. If reviewers say "the pauses feel long" but
  your beat table says they are short, suspect the clock first.
- The working shape: a static `Timer.publish` on the `.common` run-loop
  mode plus wall-clock
  deltas - `reelTime += clamp(now - lastTick, 0, 0.5)`. The clamp exists
  only for app suspension; a tight clamp (0.1) re-creates the starvation
  bug by refusing to let the clock catch up after a stalled tick.

## Cuts land on motion, after the exit clears

Cut between scenes the way an editor cuts film: on motion, at the end of a
beat, never on a resting frame. Two hard rules:

- The cut must come AFTER the outgoing actor's exit animation has left the
  visible frame. Measure the exit's real duration and give it margin; a cut
  timed "0.1s after the exit starts" unmounts the view with the actor still
  half on screen, and it blinks out of existence - the single ugliest
  artifact a sequence can produce. When the exit ends offscreen (an ease
  tail past the frame edge), the cut may trim that invisible remainder
  freely.
- Match energy across the cut: an actor exiting left pairs with the next
  entering from the right or below; a spinning exit hands its rotation to a
  rolling entrance. The eye reads matched motion as one continuous take.

If the outgoing actor's exit is powered by a mechanism that also brings a
successor (a pager turn), and the successor must never be seen, do not
race the cut against the arrival - timing gaps get eaten by frame jitter.
Give the stage an explicit suppress flag that plays the exit and skips the
install. Structural impossibility beats a tuned window every time.

## Pre-mount the next stage

A freshly mounted `SCNView` compiles Metal pipelines; its first beat fires
0.3-0.8s late, unpredictably (see first-frame-and-warmup.md). Inside a
sequence that jitter is fatal: downstream beats are timed against the bake
that just started late, and the next cut clips the exit.

The sequencing cure: mount the NEXT scene's stage while the current one
plays, with its actor parked offstage (fully outside the frustum, already
in its entrance heading). The stage is transparent, the actor invisible -
the viewer sees nothing, the shaders compile in the shadow of the current
scene, and when the entrance trigger lands the bake starts on that exact
frame.

SwiftUI mechanics: give the pre-mounted stage one structural `if` slot
whose condition stays true across the cut -

```swift
if scene == .badge || scene == .bell { BellStage3D(...) }
```

The condition value changes but never goes false, so the view's identity
(and its warmed pipelines) survives the scene switch. Disable its hit
testing until its scene is live.

Do NOT pre-mount a stage whose entrance auto-fires on mount; it will
perform to nobody and be mid-scene at its cut. Pre-mounting requires a
trigger-gated entrance and a parked actor.

## A stage may outlive its own cut

Some effects must not die with their scene. Tire smoke that vanishes the
frame the wheel scene unmounts reads as a glitch; smoke that hangs and
dissipates OVER the next scene's entrance reads as craft. Same mechanism
as pre-mounting, mirrored:

```swift
if scene == .wheel || lingeringWheel { WheelStage3D(...) }
```

Keep the outgoing stage mounted (last in the ZStack - on top), let its
particles fade naturally, and clear the flag with a beat once they are
gone. Transparent stages composite; the viewer sees one scene with smoke
drifting over it. Disable hit testing on the lingering view.

## Interactive beats: re-basing the timeline on the user

The strongest sequence beat is one the USER performs (grab the actor, throw
it offstage). The script then cannot run on absolute time - everything
after the interaction anchors to the moment the user finishes:

- Keep a T0. Beats before the interaction live at absolute times; every
  beat after it lives at `t0 + offset`. On the user's release (or commit),
  set `t0 = now (+ settle margin)` and REBUILD the beat array.
- When re-basing, DROP the already-consumed absolute beats instead of
  carrying them into the rebuilt array. A stale intro beat sitting at its
  absolute time re-fires after an early interaction and re-arms state the
  interaction already consumed - the classic shape is a flag like
  `awaitingRelease` coming back to life with nobody left to release, and
  every guarded beat after it silently skipping.
- Provide a fallback beat: if the user never interacts by a deadline, the
  script performs the moment itself and re-bases identically. Postpone the
  fallback while a finger is down (`guard !isHolding`), or it fires into
  the middle of the interaction.
- The clock must not stop at the timeline's end while an interaction is in
  flight: ending the reel freezes the scene under the user's finger and
  the release lands in a dead clock. Gate the stop on "no interaction
  pending".
- Callbacks from the stage (hold began/ended, pose streams, commits) are
  the host's only truth about the finger. Any stage code that STEALS the
  actor from a live gesture must close the contract - emit the release
  callback - or the host's interaction state latches forever.

## Scripted triggers must survive fingers

Full treatment in gestures-and-haptics.md ("Scripted beats versus live
fingers"); the sequencing-side summary: a trigger-diff latches BEFORE the
action runs, so an action that refuses (actor held, actor mid-beat) loses
its beat forever and the sequence dies downstream with no error. Every
scripted beat that can meet a finger gets a bounded retry; every retry
poll carries a generation token so an interaction cancels it.

## A recording lead

A sequence built for screen recording opens on an EMPTY stage for a couple
of seconds: the restart gesture (double-tap) fires, the operator starts
the recording and clears their hand, and only then does the first actor
enter. Implement it as a mount beat - the first scene's stage simply is
not in the tree until the lead expires. Zero changes inside the stage; the
entrance plays on mount as always.

## Verifying a sequence

- Verify CUTS on extracted frames, not by eye: step the recording at 2-4
  fps around every cut and look for the outgoing actor still on screen,
  the incoming actor's first visible frame, and anything that pops.
- Frame-step every handoff between a bake and a rest pose; a one-frame
  yaw or position snap hides at full speed and glares in stills.
- Measure the clock: burn the reel time into a debug label (or log beats)
  and compare against video timestamps. A growing offset is the starving
  clock; a constant offset is the launch cost.
- Expect the operator's hands: on a shared simulator or device, automated
  runs get touched. Interactive beats firing "impossibly" in a hands-free
  recording usually mean someone grabbed the actor mid-run - design the
  script to survive it, then verify that it did.
