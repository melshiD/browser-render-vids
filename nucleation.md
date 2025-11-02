Skip to content
Chat history

You said:
I need to build a script that can take an animation that would play in a browser, but render a video of it instead so it doesn't burn down the client's computer (we're talking about animating large triangle-count models and moving them with simple instructions.  I don't have the compute to render the entire models ((at least I don't think I do)) so we need a way to just render an animation tick at at time.  We'll be rendering the 3D with three.js ).  Walk me through, without using specific code examples, the tech and ideas we'll use to build a script that can do this.  It needs to render specified sizes of videos.  We can even supply this feature or server (however it's built) with an interface for me to set values on.  

Help me work out how this will integrate into my webpage (so I can set up an animation on the browser screen and then render a video of it.  If we can render with a transparent background, all the better.) 

If we need to build a dedicated animation editor too, we should do so.

Please give me your answer as an instructional description for a newish software engineer to follow and implement.
ChatGPT said:
Love this problem. You're basically asking: “How do I turn my browser-based Three.js animation into a high-quality offline render pipeline that outputs video (ideally with alpha), without melting the client’s GPU in real time?” Let's design that system end to end.

We’ll cover:

What the pipeline actually is (conceptually)

The parts you need to build

How it ties into your webpage / editor

Special considerations (transparent BG, large meshes, performance)

No code yet — just architecture and responsibilities. You’ll be able to implement from this.

1. The high-level pipeline
Here’s the flow we want:

You (or a content person) set up a scene + animation using Three.js in the browser — lights, camera, models, movement, etc.

Instead of “recording the screen,” you export a “render job,” which is just structured data describing:

the scene (what models, materials, camera, etc.)

the animation timeline (what happens at each frame)

output settings (resolution, frame rate, duration, background mode)

A headless renderer runs that job frame by frame using the same logic as your browser animation loop, but offscreen.

That renderer spits out individual frames (images, ideally PNG if you want alpha).

Those frames get combined into a final video file.

So you’re not capturing live playback. You’re deterministically re-simulating your animation offline and dumping perfect frames.

This is key: determinism. Your animation logic must be “given time t, I can tell you exactly what the scene looks like,” not “every tick, move 0.01 units forward from wherever we are now.” We'll come back to this.

2. Core components you’ll build
You will need five main pieces.

A. Scene Description Format
You need a serializable description of your scene that both the browser and the offline renderer agree on.

This should include, at minimum:

List of assets:

Meshes (including what file they load from: GLTF model, etc.)

Materials (color, roughness, metalness, transparency, emissive, etc.)

Lights (type, position, intensity, color, shadow settings)

Camera (type, FOV, position, rotation)

Background (solid color, environment map, or "transparent")

Animation channels (more below)

This is basically your “project file.” JSON is totally fine.

You already do most of this in Three.js in the browser using code. The difference here is:

You’ll also generate/export a structured version of it (no functions, no closures) so a server can recreate the same scene in a headless context.

Tip: Think of this like saving state from your editor.

B. Animation Timeline / Keyframe Data
This is critical. You cannot just say “play the animation” and hope the server matches what your laptop did.

You need to describe animation in a timeline-based way, so it can be reproduced offline.

There are two main approaches:

Option 1: Keyframed / procedural timeline
You define animation as:

At time 0.0s, mesh “Armature” rotation = X.

At time 1.0s, rotation = Y.

Interpolation is linear / cubic / etc.

Or:

At each frame t, camera position is given by a function you know and can re-run (for example, orbit around target at radius R and angular speed ω). The function must depend only on time t, not accumulated state.

In practice:

You define animation tracks for each property (position, rotation, scale, material opacity, light intensity, etc.).

Each track can be evaluated at any arbitrary time value.

This is almost exactly how GLTF animation works and how Three.js AnimationMixer works conceptually. That’s good, because it means you can likely piggyback on existing animation data.

Advantages:

Small data payload.

Easier to edit in an animation UI.

Easy to scrub and preview live in the browser.

Option 2: Direct per-frame bake
You pre-bake everything up front in the browser:

For frame 0 → object A is here, B is here, camera is here

For frame 1 → ...

etc.

Then you export a giant array of “per-frame snapshots”.

This is heavier data (especially on long animations), but it’s dead simple to replay because the renderer just does:
“Load frame 123 state, render, save image.”

Use this if determinism is hard (physics, ragdoll, morph targets with runtime noise, etc.).

You can even offer both:

“Bake animation for render” button generates per-frame data for submission, so the render server doesn’t need to know your logic at all.

That’s often the most reliable path for v1.

C. Offline / Headless Renderer
This is the workhorse. Its job is:

Load the scene description.

For each frame:

Move everything to the state it should be in for that frame.

Render at the requested resolution.

Save the frame as an image.

Hand those frames to a video encoder.

How it runs:

This can be a Node.js process running Three.js with an offscreen WebGL context (there are libraries to let Three.js render in Node without a visible canvas).

Or it can run in a browser-like environment using something like headless Chromium (Chromium can render WebGL offscreen with GPU, which is better for heavy models than pure software).

Where it runs:

Could be:

your local machine (CLI tool)

a one-off server instance you spin up

a cloud box / container you control

The key: this renderer is not interactive. It never “plays” the scene; it just renders individual known states.

Rendering transparent background:

The renderer needs to support clearing the framebuffer with alpha (instead of a solid color).

You then save PNGs with transparency.

The video step is trickier with alpha (we’ll talk about format later), but PNG frames are fine.

Resolution:

The renderer will create a render target (imagine an offscreen canvas) at whatever size you asked for:

1920×1080

1080×1920

3840×2160
etc.

Then render into that target.

This lets you render 4K output even if the editor viewport is tiny.

This is huge: you are no longer limited by the user’s monitor.

Frame rate:

You choose a frame rate. Example: 30 FPS.

You compute frame count as:

total_frames = duration_seconds × frames_per_second

Then frame i's timestamp is i / frames_per_second.

The renderer steps through each timestamp deterministically.

D. Frame → Video Encoder
Once you have all frames as PNG (or other image format), you need to assemble them into a video.

Two modes:

Mode 1: Normal video (opaque background)
You composite a background color in the renderer (i.e. no alpha).

You encode to H.264 / H.265 / VP9 / AV1, etc.

Output MP4 or WebM, whatever you need.

Mode 2: Transparent video
You have two good options for transparency:

Don’t ship a video, ship an image sequence with alpha (PNG sequence). Many editors accept that.

Use a video codec that supports alpha, like WebM with VP9 or AV1 with alpha channel. Browser support here exists for WebM with alpha in some engines, and AV1 with alpha is emerging (this is changing fast, so you'll want to confirm current support when you implement). You'd effectively encode RGBA instead of RGB.

Practically: for marketing/hero banners on a webpage, people often just want a loopable WebM with transparency. So aim for that if you can.

Implementation detail for you later:

The encoder tool will read the folder of frames in order and mux them into a single file with the correct FPS.

E. Control / Job Submission Interface
You need some way to:

Set render resolution

Set FPS

Set duration / frame range

Set background mode (solid color / transparent)

Hit “Render”

View progress / download result

This interface can be:

A panel in your existing webpage (the animation editor)

OR a tiny internal dashboard (authenticated) that lists submitted render jobs and gives you download links when done.

You’ll send a POST to your render service with:

Scene description (or baked frame data)

Output settings (width, height, fps, etc.)

Render ID

The renderer runs, writes frames, encodes, and stores final output where you can download it.

Because of your “no async background” system requirement for me here, I’ll say this clearly for your implementation: this service is conceptually a batch job runner, even if you just run it manually from the command line. You run it, it finishes, you get your file. There's no magic waiting queue you need to build unless you want to scale.

3. How it integrates into your webpage / editor
Now let's put the above into your day-to-day workflow.

Step 1. Build (or adapt) a browser-side animation editor / previewer
Your webpage needs an editor experience because you want to:

Load the heavy 3D model(s)

Position cameras and lights

Author/preview animation

That editor should do at least this:

Let you scrub time:

There's a timeline slider (0s → N seconds).

When you move it, the scene updates to the correct state for that time.

This proves your animation is deterministic and time-based.

Let you set output options:

Width / height preset (1080x1080, 1920x1080, 1080x1920, etc.)

FPS

Duration

Background mode

Export:

“Export Render Job” button. When you click it, the browser produces one JSON blob that contains:

Scene setup

Animation data (either keyframes or per-frame baked transforms)

Output options

This JSON is what you’ll feed to the headless renderer.

Important UX note:

This editor does NOT have to be pretty-public-facing. It can be ugly/internal. Sliders and numeric inputs are fine.

Treat it like an internal tool.

If you don't already have an animation authoring UI, you will want to build at least the minimal scrubber + transform keyframe panel. That’s your “dedicated animation editor” you mentioned. Even a primitive “record camera path over time” tool is huge.

Step 2. Build the deterministic timeline in the browser
When your in-browser render loop runs today, you probably do something like “delta = clock.getDelta(); obj.position.x += speed * delta;”.

That won't reproduce exactly offline.

You want to move to: “given absolute time t, what is obj.position?”

Two ways to get that behavior:

Author time-based animation tracks and evaluate them by t directly (clean, scalable).

Or, as mentioned earlier, run the animation once in the browser, capture every frame’s transforms into an array, and that array becomes the render job. (Great for v1, especially if you do camera flythroughs, bone animations, etc.)

In other words, your editor must be able to generate either:

time → state rules
or

frame → state snapshots

Once you can do that, the rest of the pipeline becomes pretty mechanical.

Step 3. Send render job to the renderer
From your webpage/editor:

You click “Render.”

The app packages the job JSON and sends it to your render service (or writes it to disk locally if you’re running everything yourself).

The job JSON includes:

resolution target

fps

total duration

background config (solid color or transparent)

asset references (where to load models from)

animation data (keyframes OR baked frames)

Step 4. Renderer runs offline and produces output
The renderer:

Loads assets (models, textures) using the same file paths / URLs you gave it. This means assets must be accessible to it — either bundled, on disk, or fetchable.

Creates a Three.js scene from the scene description.

For frameIndex in [0 .. total_frames-1]:

Compute t = frameIndex / fps.

Apply animation state for time t.

Render using an offscreen WebGL context at width x height.

Save frame as PNG (with alpha if requested).

After all frames are saved, encode them into the requested video format.

It then makes that video available to you (locally as a file or, if you're doing this server-style, via a download link in your dashboard).

4. Special considerations
A. Huge triangle counts / GIANT meshes
Your browser may choke trying to animate millions-of-triangle meshes in real time. The offline renderer also might choke if you try to load them all interactively per frame.

Good news:

In the offline renderer, you are not running live at 60 FPS. It's okay if one frame takes a while to draw.

You can also render at lower FPS for preview, then rerun at higher quality.

But you still want to be smart:

Level of detail (LOD):

In your editor, you can view a lower-poly proxy of the mesh for animation work.

In the render job JSON, you can point to the high-res version instead.

That way you animate a light model but render a heavy one offline.

Material swaps:

You might use simple lambert/standard materials in preview, but higher-quality PBR in final render. That’s just data.

Shadows / post-processing:

Expensive effects (soft shadows, glow, reflections, SSAO, bloom) are allowed in final render even if your preview can't hit 60 FPS.

The render job should include toggles for these “final effects.”

So: design your render job format to support “preview vs final config.” That’s how you get cinematic output.

B. Camera paths
In practice, the single most important animation to keyframe well is the camera.

You’ll want:

Camera position over time

Camera target / lookAt over time

Field of view over time (for dramatic zooms)

This alone can sell motion, even if the model is mostly static.

So your editor should at least let you:

Drop camera keyframes

Interpolate between them

Scrub and see the framing

That’s probably feature #1 of your custom animation editor.

C. Alpha / transparent background videos
To support compositing the 3D over other footage:

Render job needs a flag like "backgroundMode": "transparent".

The offline renderer must clear the WebGL background to transparent and render objects with blending/alpha as usual.

Export frames as PNG with alpha.

When encoding the final video, you'll either:

produce a WebM with alpha from those PNGs, or

just deliver the PNG sequence and let After Effects / Premiere / etc. handle compositing.

One caveat: “transparent MP4” is basically not a thing (MP4/H.264 doesn’t carry alpha). Don't promise that format to stakeholders unless they're okay with non-transparent fallback.

D. Deterministic randomness
If you have anything random-looking (noise, sparkles, jitter), you must lock it down so it’s reproducible. Solutions:

Seed a PRNG manually and drive all “random” with that seed.

Or bake the result during export so it's not random anymore to the renderer.

If you skip this, your video will not match the preview.

E. Audio (if you ever need it)
If there’s music or voiceover timing:

Treat audio as a separate asset with a known duration and maybe timestamps for sync events.

The render pipeline itself does not generate audio.

Instead, after encoding the video frames, you mux in the audio track aligned to time 0.

Not necessary if you're just looping motion graphics silently for web hero sections, but good to plan for.

5. TL;DR implementation plan you can follow
In your current webpage / Three.js scene:

Add a “Timeline Scrubber” UI so you can preview the scene at any time t.

Refactor animation so it’s “state at time t” instead of “increment each frame.”

Or add a “Bake Animation” button that records per-frame transforms over time and stores them.

Add an “Export Render Job” button:

This collects:

Scene setup (camera, meshes, materials, lights, environment/background)

Animation data (keyframes or baked)

Output settings (width, height, fps, duration, background transparent vs solid)

Asset references (paths to GLTFs, textures)

It produces a JSON blob.

Build a headless render service:

Runs Three.js offscreen (Node + offscreen GL or headless Chromium).

Loads the JSON job.

For each frame:

Computes the scene for that frame

Renders at the requested resolution

Saves RGBA PNG

After frame dump:

Stitch PNGs into a video at the requested FPS.

If backgroundMode = transparent, preserve alpha (WebM with alpha or PNG sequence).

Otherwise, produce MP4/WebM.

Add a tiny admin/dashboard page (or CLI usage at first):

So you can hand it a render job and get back the finished video file(s) without doing weird manual steps.

(Optional but smart): Support preview vs final quality:

Allow the job to say: “use highpoly assets and enable bloom” for final export, even if the in-browser editor used the lightweight version.

If you build those pieces, you’ll have:

Repeatable, high-res, offloaded rendering

Deterministic animation in video form

Optional transparent background for compositing

A workflow where the browser is an editor, not the renderer

Which is exactly what you described: ship beautiful motion without cooking someone’s laptop 😌

You said:
So briefly, in one line describe what's being built?
ChatGPT said:
A headless Three.js render pipeline that takes browser-authored 3D animations and exports them as high-quality, resolution-controlled videos (with optional transparency).

You said:
Ok I just realized Cursor (the agent) has a very small chat input character limit.  Please accomodate Cursor's limit with a new usable description/instruction
ChatGPT said:
You're building a headless Three.js video renderer — a system that takes scene + animation data from your browser (or editor), sends it to a Node/headless render service, renders each frame offscreen at any size, and outputs a video or PNG sequence (optionally with transparency).

