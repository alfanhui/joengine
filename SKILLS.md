# SKILLS: what Claude has learned working on CircuitBreaker

Read this at the start of a session (CLAUDE.md points here). Add to it whenever something costs
time to find out: a hardware fact, a tool quirk, a mistake not to repeat. Keep each entry short
and say *why*. The game lives in `Projects/circuit-breaker` (its own git repo); this file and
CLAUDE.md sit in the jo-engine checkout around it.

## Working with the user

- Pair programming: build in small, explained steps; comment the code for someone learning.
  Pure logic goes in `src/game/` with unit tests first, then the hardware side.
- Never commit unless asked. Pushes go from the user's own terminal (no SSH agent here).
- The user often follows over Remote Control: put screenshots/GIFs in front of them with
  SendUserFile, and spell out any question in the closing message, not only in the wrap-up box.
- Keep `docs/BACKLOG.md` current: new ideas under Next, finished work under Done with its commit
  (or "uncommitted"). Tangents get logged there so nothing is lost.
- Anything that only the real Saturn can confirm goes in `docs/HARDWARE_TESTS.md`: what to do,
  what a failure looks like, the likely cause. When the user brings results back, start there.
- Check a claim before writing it. "Same length as before" was wrong: the old linear turn took
  11 frames, not 10 (10 x 1638 falls short of 0x4000). Let a unit test prove numbers like that.
- Prototypes the user rejects get reverted fully (code, scripts, makefile, test registration),
  with the findings kept in the backlog.

## Building and testing (Windows)

- Build from **PowerShell**: `cmd /c "scripts\compile.bat"` in `Projects/circuit-breaker`.
  mkisofs fails from Git Bash. Filter output for `error|warning|game.iso found`.
- Unit tests: `tests\run_tests.bat` (MSVC with /W4 /WX if no gcc: int-to-short needs a cast).
  A new `src/game/x.c` must be added to: `makefile` SRCS, `tests/run_tests.bat`,
  `tests/run_tests.sh`, and `run_x_tests()` in `tests/test_main.c`.
- See the exact compile flags: `make -n -B src/file.o` (PATH needs `Compiler\WINDOWS\bin`).
  To read generated SH-2 code, run `sh-elf-gcc ... -S` with those flags (objdump.exe doesn't run).
- E2E in Docker (works with the laptop lid shut): `tests\e2e\run_e2e.ps1 -Steps name.steps`.
  - Boot takes ~15-17 s (`wait 17`); presses sent earlier are lost.
  - Keys: `up/down/left/right`, `a b c x y z l r start`; `press b 6` repeats; `hold up 3`;
    `keydown up` ... `keyup up` holds a button while other steps (shots) run.
  - Boot time varies (14.6 s to 17+ s, slower just after Docker restarts): a fixed `wait` before
    the first press loses presses. Start scripts with `wait_boot` (run.sh polls the screen for
    the floor's dark navy; the BIOS's Saturn globe is grey and fooled a looser colour test).
  - The game boots into the opening flight (`BOOT_SCENE`, input.c). Play scripts start
    `wait_boot`, `skip_prologue` (START, A, A, START: the debug camera's scene jump). Prologue
    scripts press B once (hurries the flight to x3) and wait 3.5 s for it to land: the scene
    jump's target depends on whether the flight has landed yet, so it isn't reliable there.
  - Docker Desktop can drop its engine mid-run after a Windows restart ("dockerDesktopLinuxEngine
    ... cannot find the file"): start it again and wait for `docker info` to succeed.
  - Play starts in 1st person. Scripts written before that press Down after the boot wait
    to start from iso; `start_view.steps` checks the starting view.
  - Each `shot` takes ~130 ms, so back-to-back shots in 1/8 slow motion land ~1 game frame apart.
  - The `cpu` debug figure is a one-second average: wait ~2.5 s after a change before a shot.
  - Shots land in `tests/build/e2e` (git-ignored). Crop, zoom, montage and GIFs with the
    container's ImageMagick: `docker run --rm -v "${o}:/out" --entrypoint convert circuit-breaker-e2e ...`.
  - `.sh` and `.steps` files must stay LF (`.gitattributes`).
  - Run `docker ... /out/...` ImageMagick commands from PowerShell: Git Bash rewrites `/out`
    into a Windows path. For a glob (`montage /out/x/*.png ...`) use `--entrypoint sh` and
    `-c "montage ..."` so the container expands it: a list splatted from PowerShell made nothing,
    silently, and the old sheet was read as if it were new. Check the sheet's timestamp.
  - A view of the whole arena from above: break the loop (`arrival.steps`' presses) and shoot
    ~0.7-1.1 s after the third; the flight down passes over it as the walls rise. Pitching the
    debug camera by hand (`hold c`) gave a useless shot. To see a later round:
    `tests\e2e\film_round.ps1 -Round N [-Steps x.steps]` (builds with `BOOT_ROUND` flipped,
    films, makes a sheet, puts it back).
  - `run.sh` reads `BOOT_SCENE` from `src/input.c`, so `skip_prologue` (to play) and `start_intro`
    (to the opening flight) work whichever part the game boots into. A new prologue script starts
    `wait_boot`, then `start_intro`.
- Text printed with `jo_printf` stays on VDP2's text layer until it's written over: when a part
  of the game stops drawing its line (the status line in the prologue), blank it once on the way
  out, or the last message stays up.
- Turn camera GIFs: `tests\e2e\turn_gif.ps1` (check both rows start on the same turn frame).
- Fast boot on the PC: `scripts\run_fast_boot.bat` (YabaSanshiro's emulated BIOS skips the intro).

## Editing files here

- Python run on Windows writes CRLF (text mode). Git normalises the C files on commit
  (`core.autocrlf`), but `.steps` and `.sh` are read straight from the working copy by the E2E
  container: open them with `newline=''`, or strip the CRs after with
  `sed -i 's/\r$//'`. Check with `file`.
- Bash heredocs mangle backslashes, and long `sed` patterns break easily. For multi-line or
  backslash-heavy edits, write a Python script to the scratchpad with the Write tool and run it.
  A Python heredoc in the Bash tool lost `\\` (it became `\`, breaking `'..\\src\\'` and
  `"\\\n"`): use the Write tool for the script, or `chr(92)`, or just the Edit tool.
- Measuring a screenshot: E2E shots capture Mednafen's window: 906x720, the whole 320x224 picture
  at 2.83 px across and 3.21 down per Saturn pixel (TV aspect: pixels are ~1.13x taller than wide,
  so a square of pixels shows as a tall oblong; the radar widens x by 9/8). Before 2026-10-28 shots
  were a 906-wide crop of the desktop: 73 px of border on the left, the right 15 Saturn px missing. Find edges with ImageMagick `-crop 906x1+0+ROW +repage txt:-` and list
  colour changes. A host "probe" (a scratch .c calling src/game, built with cl through a .bat
  that copies run_tests.bat's vcvars lines; no gcc on PATH) shows what the clip code really
  keeps. The trail hole was found by comparing shots built with different values (5/10/20).
- When measurements don't fit one model, solve for the hidden offset: the hole's screen position
  at cuts 20/10/6 (97/132/153 px) fitted x = 160 * 25 / (cut + 20), i.e. a 90 degree view with
  the eye 20 behind the cycle. `initCamera` had SGL's eye at z 5 looking at the origin: 5 SGL
  units = 20 world at 1/4 scale. My first reading ("the view is 125 degrees") was wrong: check a
  theory against every measurement, not one.
- `run_e2e.ps1` needs `powershell -NoProfile -ExecutionPolicy Bypass -File tests\e2e\run_e2e.ps1 -Steps x.steps`
  (scripts are disabled by policy when called with `&`).
- Don't edit files with PowerShell's `Set-Content -Encoding utf8`: Windows PowerShell 5.1 adds a
  byte-order mark (it did to main.c). Flip a `#define` for a test build with `sed -i` in Bash.
- Many files are CRLF. Edit with Python reading `newline=''` and writing the same line ending
  back, or use the Edit tool.
- Debug readouts: a temporary `jo_printf` line marked `// TEMPORARY TEST`, then grep for that
  marker and delete it. On-screen numbers settled questions faster than guessing (e.g. proving
  the curve's drops matched the PC, finding where an interrupt vector pointed).

- The user's local Mednafen (`%USERPROFILE%\.mednafen\mednafen.cfg`) has the Saturn pad remapped
  for a laptop: WASD d-pad, A/B/C = J/K/L, X/Y/Z = U/I/O, L/R = Q/E, Start = Enter (backup:
  `mednafen.cfg.before-remap`). The Docker E2E Mednafen keeps the defaults (numpad), see run.sh.
  Keys are USB scancodes (a = 4 ... z = 29); edit only while Mednafen is closed (it saves on exit).

## Saturn / SGL / jo facts that cost time

- **jo-engine 2026 has no SGL** (circuit-breaker moved on 2026-11-08, branch
  `feature/jo-engine-2026`). Notes below that name SGL are history unless they say otherwise.
  What took finding out:
  - SGL's types are jo's under new names, same memory layout: POINT -> `jo_pos3D_fixed` (`.x`,
    not `[X]`), POLYGON -> `jo_3d_polygon` (`.normal`, `.vertices`), ATTR ->
    `jo_3d_polygon_attribute` (flag/texno/atrb/colno/gstb/dir -> plane/texture_id/draw_mode/
    color/gouraud_table/direction), PDATA's pntbl/nbPoint/pltbl/nbPolygon/attbl ->
    points/point_count/polygons/polygon_count/attributes. Sort codes keep their values
    (SORT_BFR 0 = `JO_3D_SORT_PREVIOUS`, MIN 1, MAX 2, CEN 3 = `JO_3D_SORT_CENTER`);
    sprPolygon/sprLine -> `JO_VDP1_POLYGON`/`JO_VDP1_LINE`; CL_Gouraud ->
    `JO_VDP1_COLOR_CALC_GOURAUD`. The rename was a script (no layout change, so no surprises).
  - slSin/slCos -> `jo_sin_angle`/`jo_cos_angle` (same jo_angle, 4096-step table). There's no
    tan of a jo_angle (`jo_tan` takes whole degrees): `src/utils/angles.h` `angle_tan` =
    sin/cos with `jo_fixed_div`, the SH-2's divider. slRotX/slScale/slPerspective by an angle:
    jo's `__jo_3d_rotate`/`__jo_3d_scale`/`__jo_3d_set_perspective` ("internal", still public in
    v2026.2), wrapped in that same header. slConvert3Dto2D -> `jo_3d_transform_point` then
    x * `jo_3d_get_projection_distance()` / z. slZdspLevel -> `jo_3d_display_level`.
  - **jo uses SCU DMA level 0 every frame** (the VDP1 command table): the floor fog moved to
    level 1. Levels 1 and 2 are free (CD reads use the SH-2's own DMA).
  - jo writes VDP2 registers from a shadow copy in VBlank, only the ones it changed. It has no
    calls for the line colour screen: LCTA and LNCLEN are written directly; CCCTL through
    `jo_enable_screen_transparency(JO_RBG0_SCREEN, 0)`. jo's coefficient table no longer carries
    line colours (no K_LINECOL to undo).
  - VDP1 RAM: textures from 0x10000 (was 0xE1C0), up to 0x7E4F8 (v2026.2; 0x7E6F8 in v2026.1);
    jo's sprite shading tables from 0x7E4F8 (v2026.2: they were at 0x70000), its 3D's at 0x7E700
    and 0x7FF00. Ours at 0x70000-0x75808 still fit. Command tables at 0x20 and 0x8000,
    turn about: LOPR reads 0x1000 more in the second (`perf.c` masks it).
  - `JO_FRAMERATE = 1` now swaps VDP1's buffers every VBlank, finished or not: a frame VDP1
    can't finish loses its last commands (the HUD). `JO_FRAMERATE_DYNAMIC = 1` swaps only once
    drawn, as SGL did; the game steps by `perf_last_frame_vblanks`.
  - jo zeroes its polygon counts (`jo_3d_get_displayed_polygon_count` etc.) at the end of each
    frame, after our callbacks: read them at the frame's end, not the next frame's start.
  - v2026.2: A+B+C+Start and the CD tray opening go back to the BIOS menu by default
    (`jo_core_set_restart_game_callback` / `jo_core_set_cd_tray_open_callback` to change it).
    With `JO_DEBUG`, the master CPU traps reads and writes to the first 4 KB (NULL pointers):
    a crash there is a real bug, not jo.
  - `start.S` clears all of BSS: jo's old quarter-clear bug (byte count from int pointers) is gone.
  - jo gives jo_malloc all free high Work RAM itself; SGL's 0x060C0000 limit is gone. Low Work
    RAM is still unused by jo (main.c adds it).
  - **The CPU cost on foot, and how it was won back** (2026-11-08). After the port the yard
    took cpu 91-107% (52-60 fps); under SGL 66-81%. SGL had shared polygon work with the slave
    SH-2 without being asked; jo does all of it on the master. Measured: jo's draw calls were
    ~40% of a frame (~1000 cycles a polygon), every polygon on jo's fast assembly path (none cut,
    none in parts: `JO_TEST_PATH_COUNTS`), the meshes in high Work RAM. The SCU DSP for vertices
    (`jo_3d_enable_dsp`) changed nothing. What did:
    - `JO_COMPILE_WITH_DUAL_CPU_MODULE = 1`: jo sorts and sends the VDP1 commands on the slave
      (outside our `cpu` figure, but in the frame's time: the late frames went from ~27 to 5);
    - `src/slave_draw.c`: the corridor's floor and boxes handed to the slave as they're built
      (`jo_3d_slave_begin`, `jo_core_exec_on_slave`, `jo_3d_slave_mesh_draw`,
      `jo_core_wait_for_slave`, `jo_3d_slave_end`): cpu 93-106% -> 54-84%. Only meshes with
      arrays of their own, drawn once a frame: the slave reads them while the master goes on.
      jo clears the slave's cache before each job, and both caches write through, so what the
      master wrote before handing a job over is what the slave sees;
    - `JO_FRAMERATE = 1`, not `JO_FRAMERATE_DYNAMIC`: in its manual frame change jo clears each
      frame with a whole-screen polygon, ~4 ms of VDP1, and VDP1 became the limit (48-52 fps at
      busy 96-100%). Mode 1 lets VDP1 erase for free, but a frame VDP1 can't finish is shown
      with its nearest commands missing ("vdp1 over" counts them: 0 in every film of play).
    Now: the yard 60 fps throughout at cpu 40-72%, the doorway 60 (was 30-45), `stairs_perf`'s
    platform lip 60 (30 even under SGL). There the master waits ~9 ms for the slave: the slave
    is the busier CPU.
  - **The VDP2 floor (RBG0) on jo 2026** (2026-11-08: the grid ~10x too big, iso lines at odd
    angles, no sky). jo's maths (vdp2.c `jo_vdp2_set_rotation_matrix`) is right for any matrix,
    but VDP2's registers aren't: matrix entries A-F are 4 x the texels per matrix unit and VDP2
    keeps sign + 3 integer bits (under 8, or they wrap); the position (Mx, My) under 8192 texels;
    and the zoom comes from a table of 128 / address, coarse when the floor is far (big depths in
    matrix units: bands of rows jump, a dashed grid). Under SGL the floor was drawn in the world's
    1/4-size matrix: 32.8, wrapped. Now main.c's floor block undoes the world's matrix and does
    its own: the world at 17/16 (>> view_floor_shift), so 1.93 texels a matrix unit (7.7), and
    the camera's place within 16 tiles (`floor_wrap`). The sky (plane B, `draw(false)`) is drawn
    before `jo_3d_camera_look_at`, as in jo's plane sample: after it, the camera's -z made it
    wholly transparent.
  - **Single-sided polygons (`JO_3D_SINGLE_PLANE`) face by their winding on screen in jo 2026**
    (clockwise = front, 3d.c), not by their normal as in SGL. The cycle's cube had its outsides
    hidden: its faces are now listed the other way round (main.c `load_player_model`). Checked
    with the top face red and the bottom green, seen from iso.
  - **A mesh with an 8-bit texture takes the current sprite palette** (`jo_3d_set_mesh_polygon_texture`
    sets `color` from `jo_sprite_set_palette`'s), not the texture's own: the cube came out near
    black on jo 2026's bank 0. It's given WALL1.TGA's palette (`plane1_palette_id`). WALL1.TGA is
    navy and purple with white outlines: the grey cube of the SGL builds was another palette by
    accident.
  - **A `JO_3D_SORT_PREVIOUS` chain needs its first polygon drawn.** jo skips a polygon that's off the
    screen before giving it a key, and the chain then takes the key of whatever came before it in
    that CPU's list (on the slave: near 0, the nearest there is). The glass floor's anchor, a dot at
    its deepest point, sat on the edge the floor's cut to (frustum.h's sides are a little wider than
    the screen): facing across the corridor the whole floor was drawn over the grey boxes and the
    map. It's now straight ahead at that depth (level.c `level_ahead`; floor_chain.steps films it).
    Not jo's clipped-part rule (after a polygon cut into parts, `previous_polygon_z` becomes its
    nearest corner): changing that in 3d.c made no difference here.
  - **Arena wall textures need jo's texture tiles near the camera**: jo cuts a polygon that goes past
    its guard band (512 px; SGL allowed 1023, which `wall_clip` was tuned to), and VDP1 stretches
    the whole texture over the cut shape: a band of horizontal lines atop the near wall and a step
    in its foot. `jo_3d_set_texture_subdivision_near_only` on the three wall textures (level.c)
    cuts those polygons into textured tiles instead. edge_1st.steps' `edge_1st_along` shows it.
  - **Mednafen open on game.iso makes every build fail**: make's clean can't delete it ("Error 1",
    not ": error"), the old disc stays, and films ran on it unnoticed. run_e2e.ps1 and
    film_round.ps1 now stop unless game.iso is newer than the build's start.
  - **The walker's traps are found faster on the PC than in films**: random walks over the real
    corridor, boxes and city (`corridor_walker_area`), checking every so often that 8 headings
    still lead somewhere. It found landings that leave you inside a wall's 40-unit margin, where
    every step had counted as into the wall (fixed: `wall_in_way` in walker.c). No gcc on this
    machine or in the e2e container: build such a probe with MSVC (vcvars64, as run_tests.bat).
  - **run_e2e.ps1's rebuild stopped on a compiler warning**: under `$ErrorActionPreference = "Stop"`
    the warnings on a native command's stderr raise. Join them inside cmd (`compile.bat 2>&1`).
  - **A film chained after a build with `&&` didn't run**: the PowerShell build step's exit code
    stopped the chain, and the old shots looked like "no change" twice. Check the shots' times.
  - **Pushing from this machine**: no SSH agent, so `git push` to the SSH remotes fails. `gh` is
    signed in: push one branch over HTTPS without changing any git config with
    `git -c credential.helper= -c credential.helper='!gh auth git-credential' push https://github.com/alfanhui/joengine.git <branch>`.
    Upstream PRs go from a worktree on `upstream/master` (CLAUDE.md, "give back upstream").
  - **jo's profiler** (`JO_COMPILE_WITH_PROFILER_MODULE = 1`, `jo_profile_get(JO_PROFILE_3D_...)`)
    uses the same FRC as `perf.c`, which zeroes it every frame: its clip figure came out as
    garbage. Timing jo's draw calls with our own `perf_part_add` agreed with its vertex and
    polygon figures. `JO_TEST_PATH_COUNTS` (in `_CCFLAGS`, then a clean build) fills
    `__jo_3d_path_counts[8][4]`. jo's profile.c calls `jo_core_exec_on_slave` without the dual
    CPU module (an implicit declaration warning): fixed upstream in PR #106.

- VDP1 maps a texture onto a 4-corner quad linearly, not in perspective: a circle texture on a
  trapezoid comes out a dome (the user: "a bent bicycle tyre"). For perspective, cut it into
  bands, each on its own projected quad. Each jo sprite call costs ~700 SH-2 cycles (~0.15% of a
  frame): 40 calls for one HUD widget was 6% CPU. Count sprite calls like polygons.

- Draw order: VDP1 has no depth buffer; SGL sorts every polygon by one depth (jo's setters write
  SORT_CEN: the centre). Long pieces next to small objects sort wrongly. `SORT_BFR` (low 2 bits
  of ATTR.sort, 0) puts a polygon at the previous one's depth: an anchor quad first, then the
  object's faces with BFR, sorts a whole object as one. Then faces must be `Single_Plane` with
  real normals (SGL culls back faces by the normal; static_mesh used to give every quad 0,1,0).

- SH-2: no divide instruction, no FPU. See CLAUDE.md "avoid division". Shifts are free only by
  1, 2, 8 or 16 bits. Measure on hardware; the PC hides the cost.
- SH-2 (Hitachi SH7604, ~28.6 MHz) parts worth knowing. Two of them, each with its own set:
  - Multiplier (MAC): 32x32 -> 64 multiply and multiply-accumulate in a few cycles. Fast, so
    "x1024 then shift" fixed point is cheap.
  - Division unit (DIVU): memory-mapped, ~39 cycles. GCC never uses it (it calls `__sdivsi3`).
  - Cache: 4 KB, 4-way, 16-byte lines, instructions and data together. Can be split as 2 KB cache
    + 2 KB fast on-chip RAM: a scratchpad for a hot loop's tables (unused so far).
    Address + `0x20000000` reads around the cache: how one CPU sees the other's writes.
  - Its own 2-channel DMA (separate from the SCU's), free-running timer (the main CPU wakes the
    second through its input capture), watchdog timer, 16-level interrupt controller, sleep mode.
- **Which chip limits the frame**: debug line 7 (`vdp1 over busy cmd v`, `game/vdp1_gauge.h`)
  reads VDP1's own status registers. `busy` near 100 means VDP1 was still drawing when the CPU
  finished: cut pixels, not CPU. At 60 fps with `over 0` it just means VDP1 is the slower chip but
  fits. Use it before guessing from "low cpu, low fps".
  But `busy` high with `cpu` over ~85% means both chips are short, and it can't say which more:
  the city's view down the corridor (30 fps, busy 90, cpu 95%) went to 60 with any one part
  taken away, each removing CPU work and VDP1 pixels both. Settle it with one-thing-off builds
  (below), not the gauge. (2026-11-04: an exact cut at the screen's edge, VDP1-only, changed
  nothing there: the "diagonal fill lines past the guard band" theory was wrong for it.)
- **Culling can cost more than it saves.** The first occlusion cull of the arena's walls
  (`game/occluder.h`) took 48 polygons off and the frame got *slower* (cpu 95 to 101%): ~400 box
  tests a frame, each 3 walls x 5 planes. Fixed by testing the biggest thing first (a whole side,
  block or run: one test), finding a run's hidden ends by halving (hidden-ness only falls as a
  box grows), and stopping each test at its first failed plane (the wall's own plane first).
  Time a new cull on its own (borrow an unused `perf.h` part, e.g. `bd`) before believing it.
- **Only test the planes that can hide the thing.** A portal's sill can't hide the floor it's the
  edge of (or anything standing on it, eye above it); a jamb on the side wall you're not beyond
  can't hide anything inside the room. Tested anyway, they send pieces to the clipper for nothing.
  Prove such a claim with a unit test of the geometry (`test_frustum.c`), not by eye.
- **Detail past where it shows, through a portal, is the cheapest cut**: from the city, the
  doorway's floor tiles past their outlines (1000) went to the far glass: ~5% CPU, the view to 60.
  Culling more exactly gave ~1-3% each.
- **The part figures include interrupts.** Each part is timed by the FRC across whatever runs
  meanwhile, so SGL's vblank interrupt landing in a part counts in it (and the "worst frame in a
  second" picks exactly those frames): `fl126` with `pk 106`, and `fl` fell to 22 just by timing a
  loop inside it. Trust `cpu`, `late` and one-thing-off builds over a part's peak.
- **A film that reads the same digits as the last one didn't run.** A failed build leaves no
  game.iso, and cropping the old shots gives the last film's numbers again. Make the script stop
  when the build line has no `game.iso found`.
- Judge "it looks the same without X" side by side, at full size: the arena's walls looked
  absent from the doorway view, but its top rises above the corridor's walls (dark on dark).
- The `cpu` figure stops at `draw_3d`'s end: jo's own work after it (sorting and sending the
  commands; SGL's slSynch before 2026-11-08) isn't in it. jo 2026 transforms and cuts each mesh
  during `jo_3d_mesh_draw`, so that is in it. A
  frame at ~85% average / ~100% peak already misses the vblank; aim for ~75%.
- The debug line's part figures (`ar co sd tp fl ...`) can read more than the `cpu` peak (`fl126`
  with `pk 107`; a temporary check showed the FRC isn't reset mid-frame, cause not found). Read
  them as relative, and trust one-thing-off builds: a temporary `int tt_exp` switch skipping one
  draw call each, flipped with `sed` by a scratch script that builds, films and crops (four
  builds in ~8 min in the background).
- E2E turns don't land the same run to run, and turning depends on the pauses: one `hold right
  2.2` turns less than eleven `hold right 0.2` with shots between (the turn eases in each press).
  To return to a view another film reached, copy its presses *and* shots, then nudge.
  `doorway_perf.steps` does so, then sweeps left in 0.1 s steps with a reading at each.
- Performance hunting: the debug readout's "cpu" is our callbacks' time (game logic plus handing
  polygons to SGL; not VDP1 drawing), so a rise with flat polygon counts is CPU work. Anything
  looping over every trail segment runs several times a frame (each rider's collision, each bot
  query, radar, pieces, occlusion): keep the per-segment work to comparisons. Measure fast with
  `stress_cpu.steps` (X fills a 64-corner trail at once) and confirm with `busy_play.steps` (2 min
  of turns); a one-line early `return` in a suspect function, built and measured, beats guessing.
- The second SH-2 is off in our build (`JO_COMPILE_WITH_DUAL_CPU_MODULE = 0`). Turned on, jo runs
  `jo_core_add_slave_callback` functions alongside the main callbacks and waits for them before
  drawing (`jo_engine/core.c`). It isn't 2x: both CPUs share one bus. SGL may already use it
  internally (its polygon count is written from there: the flaky `displayed` readout).
- VDP1 vertices must stay within -1024..1023 or they wrap: keep near wall/trail corners in front
  of the camera (`wall_clip`). SGL projects points behind the camera and distorts the quad.
- VDP1 textures: width a multiple of 8, height at most 255. No perspective correction inside a
  quad, so long quads near the camera bend: split them into short pieces.
- Thin texture lines minify badly far away: identical pieces drop the same texture rows and
  look fine, but any 1-pixel height difference between neighbours (a curve, a wall at an angle)
  drops different rows and the lines jump. Fix with a coarser texture for far LOD pieces.
- jo gives every 8-bit VDP1 sprite the first TGA palette, so wall textures are 24-bit.
- Disc file names are 8.3 (`WALL_BASE.TGA` failed; `WALLBASE.TGA` works).
- SGL stops drawing polygons a little under 8192 units deep. The world is handed to SGL at
  1/4 scale (`VIEW_WORLD_SCALE`), and `slZdspLevel(7)` keeps near pieces from being dropped.
- (SGL) jo's `JO_VDP2_LINE_COLOR_TABLE` sat at the start of VRAM B1, on SGL's text font: the
  line colour table is allocated with the floor's cells (`JO_VDP2_RAM_CELL_RBG0`) instead.
- VDP2 line colour in additive mode can only brighten: it can fog to a glow, never to black.
  Ratio mode (`CC_RATE`) with a black line colour does fog to black; ratio 31 = black, 0 = floor.
  The ratio is one register (CCRR, 0x25F8010C) for the whole screen.
- jo's 3D sky (`jo_background_3d_plane_b_*`) isn't a separate screen: it's RBG0's rotation
  parameter B, the floor's own screen. Anything done to RBG0 (fog, line colour) reaches it;
  per-row effects must leave the rows above the horizon alone.
- VDP1 Gouraud shading (`src/object_fog_vdp1.c`): ATTR.atrb |= CL_Gouraud, ATTR.gstb = table
  address / 8. Tables are 8 bytes (4 RGB555 colours, one per Vertices[0..3]) from VDP1 RAM
  0x70000 (gstb 0xE000); jo's sprite shading uses the first ones. Each channel adds
  (value - 16): 16 = no change, so it can only dim by 16 of 31 steps, never to black.
  VDP1 draws a frame behind the CPU: per-frame tables need two sets, alternating.
  jo's textures may grow up to 0x7E6F8 (jo 2026), through the table area: check `jo_sprite_usage_percent`.
- SGL reads a mesh's attributes during `jo_3d_mesh_draw` (slPutPolygon): shared ATTR arrays
  can be changed between draws in a frame (the iso cutaway and the wall fog rely on it).
- SGL drops polygons and sprites nearer than its screen distance / 2^slZdspLevel (level 7:
  /128). Screen distance grows as the FOV narrows: 1.25 SGL units at 90 degrees, 2.2 in iso
  (60). A HUD sprite at depth 2 vanished in iso; 4 works (the world's nearest is 5).
- `DEGtoANG()` is a float macro: fine with constants, but with a variable it's software float
  at runtime. Degrees to ANGLE: x 182 (65536 / 360 = 182.04). jo's headers don't declare
  slSin/slCos/slTan (gone with SGL: `src/utils/angles.h`).
- Screen-space rectangles (HUD): `jo_sprite_draw_4p_fixed(sprite, corners, depth, true)`
  takes pixels from the screen centre, no perspective. A solid 8x8 texture from
  `jo_sprite_add` stretches to any size (`src/hud.c`).
- Per-row VDP2 register changes: use **SCU DMA started by HBlank** (`src/floor_fade_vdp2.c`),
  not an HBlank interrupt. The interrupt hung the game when unmasked at start-up, and unmasked
  later it fired only ~2 times a frame (SGL rewrites the SCU mask). Facts that cost hours:
  - The DMA fetches RAM 4 bytes at a time. Move whole 4-byte entries (count 4); a 2-byte
    transfer leaves the read address part-way through a word and the rows drift.
  - HBlank also fires on the blank rows, so the table needs padding at the front and a restart
    every VBlank from the live row counter.
  - The row counter latches when **EXTEN (0x25F80002)** is read, not TVSTAT. VCNT isn't doubled
    in non-interlace; during vsync it jumps so it reaches 511 just before row 0 (so HBlanks left
    = 512 - VCNT there, 263 - VCNT before vsync). Source: Mednafen's `ss/vdp2.cpp`.
  - The picture is 224 rows in Mednafen although jo's `JO_TV_HEIGHT` is 240.
  - Mednafen's source (libretro-mirrors/mednafen-git, `src/ss/`) answers "how does the hardware
    really do X" faster than trial builds; read it before the third guess, not after the tenth.
- Debugging approach that worked: shrink to the smallest visible test (a fixed register value,
  16-row test bands, numbered DMA data written to VRAM and counted on screen) rather than
  staring at the real effect.
- The jo build's interrupt vector table (VBR) is the BIOS one at 0x06000000; `SYS_SETSINT`
  writes there. GCC's `__attribute__((interrupt_handler))` makes correct SH-2 handlers.
- jo's `jo_sound.sample_rate` is not Hz: SGL passes it to the SCSP as its raw pitch word (OCT in
  bits 14-11, two's complement; FNS in bits 9-0; rate = 44100 x 2^OCT x (1 + FNS/1024)). 0 is
  44100 (why Shenmue's sound worked: it never set it); 22050 is `0xF << 11`. Passing 22050
  played 20x too slow: a ping became a long rumble heard "after" the press.
- To hear what the game plays without speakers: an E2E steps file with a `record_sound` line
  records `/out/sound.wav` (Mednafen `-soundrecord`, SDL's dummy audio). The WAV header's sizes
  are left 0 when Mednafen is killed: read the data from byte 44 (stereo, 16-bit, 48 kHz).
  Mednafen won't overwrite an old recording; run.sh deletes it first.
- VDP1 Gouraud shading costs per pixel even with a table that changes nothing. On the huge
  wall pieces next to the camera it took 60 fps to ~45 while the CPU sat at 15%: low CPU with
  a low frame rate means VDP1 is the bottleneck. Set `CL_Gouraud` per quad only when a corner
  is actually fogged (`object_fog_set_quad`). To tell what VDP1 is choking on, switch one
  feature off at a time (`WALL_TRAIL_FOG`) and compare the fps readout.
  SEGA's manual doesn't list Gouraud as slow (only half-transparency and shadow). The cost may
  have been the CPU writing the tables into VDP1 RAM while VDP1 drew, which stops drawing. It's
  open in BACKLOG ("From SEGA's manuals").
- **Share Gouraud tables; don't write them per quad.** A wall or trail piece's corners are only
  two fog levels (a column's top and bottom are equally far), in the order a a b b or a b b a.
  With 16 levels that's 512 tables, 4 KB of VDP1 RAM, written once at start-up
  (`depth_fog_shared_table`). Only faces cut by the view (on foot) still need their own: 0-1 a
  frame instead of one per fogged quad. Look for a small set of values before writing per
  object into VDP1 RAM. Mednafen showed no fps change either way: it may not model VDP1 being
  stopped by CPU writes, so check this kind of change on the Saturn.
- Don't refill one SGL mesh (`slPutPolygon`'s POINT/POLYGON/ATTR arrays) for a second draw in
  the same frame: SGL can still be reading it. A two-trail test looked fine by luck; the
  prologue's 21 strokes through one mesh came out with spikes and stray diagonals now and
  then. Put every trail in one mesh and draw it once (`trail_mesh_begin` / `_add` / `_end`).
- Fixed-point sums with a camera's height in them overflow 32 bits sooner than you'd think:
  `floor_fade_rows` did eye x focal x 1000 (3804 x 597000 = 2.27 billion for the spectator view)
  and wrapped negative, so that view had no floor fog at all, unnoticed for days. Unit-test every
  camera's numbers, not just the 1st person one; divide to a ratio first, then multiply.
- VDP2 rolls the floor fine (`slRotZ` before `slRotX` in the world matrix), but the floor fog is
  one strength per screen row, level: with the horizon on screen, a rolled floor rises out of
  the fog's band at one corner. Lean the camera only once the horizon is off the screen.
- Through a narrow FOV (30 degrees, the spectator camera), geometry well to the side of the
  view projects over 1000 pixels from the centre, past VDP1's coordinate range, and shows as
  wedges and spikes. Cut what's drawn to a box round what's on screen. Trail walls with no
  thickness are edge-on (hairlines) when the camera looks along them: text written with
  trails needs the camera turned off the grid's axes (30 degrees worked).

## SEGA's own VDP1 and VDP2 manuals: the hardware as designed

`Manuals/` (next to this file) holds SEGA's VDP1 User's Manual (ST-013-R3), its supplement on VDP1
version 0, and the VDP2 User's Manual (ST-058-R2). Treat them as the ground truth; Mednafen's
source is the second opinion on timing. To search them: `pdftotext -layout x.pdf x.txt`
(Git Bash has it in /mingw64/bin), then read one chapter at a time (the VDP2 text is ~900 KB).
Registers below are SH-2 addresses: VDP1 RAM 0x25C00000, its frame buffer 0x25C80000, its
registers 0x25D00000; VDP2 VRAM 0x25E00000, colour RAM 0x25F00000, registers 0x25F80000.

### VDP1: what a polygon costs

- **The budget is pixels, not polygons.** VDP1 writes at most one pixel per clock (~27-28.6 MHz),
  so ~450,000 a frame at 60 fps: about six 320x224 screens before any overhead. Textures add
  VRAM reads; each command is a 32-byte fetch.
- **Half-transparency and VDP1 shadow are 6x slower** per pixel: they read the frame buffer
  before writing. The manual names only those two as slow. Gouraud and half-luminance don't read
  it.
- A polygon or distorted sprite is filled as a stack of lines:
  - each line runs from the A-D edge to the B-C edge;
  - there are as many lines as the longer of those two edges is long, in pixels;
  - diagonal lines leave holes, which VDP1 fills by writing some pixels twice. So diagonal fill
    costs extra pixels, and a half-transparent polygon comes out uneven.
- **Pre-clipping only works for screen-straight lines.**
  - A line wholly outside the clip area is skipped (up to 5 clocks to find out).
  - A line that starts outside is begun at the edge **only if it's horizontal or vertical**.
  - A diagonal line half off the screen is walked pixel by pixel, the unseen part included.
  - So it's the fill direction that makes cutting cheap. That's why `lay_wide_pieces_flat`
    (~480 lines down to ~20) and `level_side_lies` paid off.
- Two bits in CMDPMOD (the command's draw mode word) exist only on VDP1 version 1. MODR
  (0x25D00016) bits 15-12 give the version; check it before using them.
  - Pclp (bit 11) turns pre-clipping off: up to 5 clocks a line saved for small parts wholly on
    screen, but dearer for big ones past the edge.
  - HSS (bit 12): shrunk textured sprites sample only even (or odd: FBCR EOS) texels. It's
    faster, and the same texels are picked each time. It ignores end codes.
- Polygons, polylines and lines (untextured) hardly read VRAM; sprites fetch every texel.
- **Mesh** (CMDPMOD bit 8) draws only the pixels where x + y is even. It's a see-through screen
  door with no frame-buffer read, so it doesn't cost the 6x. A 45-degree line that starts on an
  odd pixel vanishes.
- **CPU or DMA access to VDP1 RAM or the frame buffer always beats drawing.** Drawing stops and
  waits. Write VDP1 tables when it has finished (EDSR CEF = 1) or in VBlank, not while it draws.

### VDP1: order, coordinates, colour

- **No depth buffer.** Commands draw in list order, the first furthest away (painter's
  algorithm). SGL's Z sort builds that order.
- **Coordinates:**
  - Vertices are 11-bit signed, -1024..1023. Outside that, "operation cannot be guaranteed".
  - The clip area goes up to (1023, 511). The system clip's top-left is fixed at (0, 0); a user
    clip must lie inside it. Points on the clip line are drawn.
- **Degenerate quads:**
  - Two equal corners draw a triangle.
  - Four equal corners draw **one pixel**. `glass_anchor`'s dot draws one black pixel, which the
    floor drawn after it covers.
  - Twisted or concave quads can fill outside their outline, since they're drawn as lines.
- **Overlaps:** a polyline writes the pixels at its corners twice; half-transparent, they come
  out 3/4 + 1/4.
- **Colour codes:**
  - RGB codes have the MSB set. A frame-buffer pixel of 0x0000 is transparent to VDP2.
  - Black is 0x8000, not 0.
- **Gouraud table:**
  - Four RGB555 words, for A, B, C, D (a sprite's top-left, top-right, bottom-right, bottom-left;
    a line uses only A and B).
  - Each channel adds (value - 16), clamped: -16..+15, 16 = no change.
  - It works only on RGB pixels (palette ones: undefined), and per channel, so a table that isn't
    grey shifts the hue.
- **Colour calculation** (CMDPMOD bits 2-0):
  - 0 replace, 1 shadow, 2 half-luminance, 3 half-transparent, 4 Gouraud;
  - 6 Gouraud + half-luminance, 7 Gouraud + half-transparent;
  - half-transparent blends only over a pixel whose MSB is set. Over a palette pixel or an empty
    one it just replaces.
- **Untextured commands:** set ECD = 1, SPD = 1 and colour mode 0. SGL's sprPolygon / sprPolyLine
  bits already do.
- **MSB ON** (CMDPMOD bit 15) sets the MSB of the pixels already in the frame buffer, for VDP2's
  shadow and sprite window.
- **Textures:**
  - 8..504 wide in steps of 8, 1..255 high, 32-byte aligned.
  - VRAM address 0 is the first command, so no texture can start there.

### VDP1: the frame buffer and the free load gauge

- Two 512x256x16 buffers swap every field in 1-cycle mode. If VDP1 hasn't reached the end command
  by the swap ("transfer-over"), the unfinished frame is shown.
- **EDSR (0x25D00010):** bit 1 (CEF) says the end was reached this frame, bit 0 (BEF) that the
  previous frame finished.
- **COPR (0x25D00014)** is the command being drawn now (address / 8); **LOPR (0x25D00012)** the
  last one drawn the frame before.
- Read at VBlank, these say whether VDP1 is the bottleneck and how far it got. That's better
  than guessing from "low CPU, low fps".
- Command tables are 32 bytes. Jump modes (next, assign, call, return, and skip versions) can
  skip a command without taking it out of the list.

### VDP2: layers, priority, blending

- **Layers:**
  - NBG0-3 (normal), RBG0/RBG1 (rotation), LNCL (line colour screen), BACK (back screen).
  - One rotation screen can show with normal ones; two rotation screens leave no normal ones.
- **RBG0 has two parameter sets, A and B.** It can switch between them per line or per dot, by
  the coefficient MSB (mode 2) or by a window (mode 3). That's jo's floor and 3D sky.
- **Priority:**
  - Each layer has a 3-bit number; higher is on top, and **0 isn't displayed**. Equal numbers:
    sprites first, then RBG0, NBG0, NBG1 and so on.
  - VDP1 pixels in RGB format always use sprite priority register 0 and colour calculation
    ratio register 0. To VDP2, all our RGB polygons are one layer with one priority and one
    blend.
- **Blending (colour calculation):**
  - It mixes the top image with the second only. "Extended" mixes 3-4 layers, at normal
    resolution only.
  - The ratio is 5 bits: value v gives top : second = (31 - v) : (v + 1). 31 is all second image;
    0 still keeps 1/32 of it.
  - The ratio comes from the top image's register, or (CCRTMD) from the second image's.
  - "Add as is" can only brighten, clamped.
  - **There is no per-line or per-dot ratio table.** A ratio per row needs a register write per
    HBlank (our SCU DMA, `floor_fade_vdp2.c`).
- **Line colour screen:**
  - When it's enabled for a layer that's on top, it's slid in as the second image, so that layer
    blends with it.
  - Its colour is one for the screen or one per line (a VRAM table of 11-bit colour RAM
    addresses).
  - It can also come per dot from the rotation coefficient table (7 bits), so it turns with the
    floor.
- **Back screen** shows where every layer is transparent: one colour, or one RGB555 colour per
  line (a sky gradient for nothing).
- **Rotation (RBG0):**
  - VDP2 reads the parameter table (0x60 bytes a set) every line.
  - The coefficient table, per line or per dot, sets kx/ky (the zoom that makes perspective) or
    Xp. A coefficient with MSB = 1 makes its dots transparent.
  - The table lives in a VRAM bank set aside for it (per dot), or in colour RAM's second half
    (then colour RAM mode 1, 1024 colours left).
- **Windows:**
  - W0 and W1 are rectangles, or **line windows**: a start and end x for every line, from a VRAM
    table. The sprite window is the VDP1 pixels with MSB set.
  - Each layer can have inside or outside made transparent, windows combined by AND or OR.
  - There's also a colour calculation window (no blending inside) and the rotation parameter
    window.
  - Coordinates are screen pixels (x in 2-pixel steps at 320 wide).
- **Colour offset:** per layer, a signed 9-bit value added to R, G and B (two sets, A and B), after
  blending, clamped. Fades and flashes of the whole screen at no cost.
- **VDP2 shadow:** halves the brightness of layers under shadow-sprite pixels (palette sprites,
  normal or MSB shadow). It's done in VDP2: no VDP1 frame-buffer read.
- **Gradation:** a sideways blur of one layer (1 : 1 : 2 over the two pixels to its left). Normal
  resolution and colour RAM mode 0 only; it can't be used with the line colour screen.
- **Colour RAM:** 4 KB.
  - Mode 0: 1024 RGB555 colours, mirrored. Mode 1: 2048 RGB555. Mode 2: 1024 RGB888.
  - Word access only. Written during display it can disturb the picture: write it in VBlank.
- **VRAM:** 512 KB in four banks (A0, A1, B0, B1).
  - Each bank has 8 access slots a cycle (T0-T7), shared by every layer's pattern and character
    reads and the CPU. A layer without its slots shows garbage.
  - Rotation screens take whole banks (the RDBS bits).
  - The CPU gets VRAM during display only in slots set to CPU, and waits otherwise. With DISP = 0
    it's free.
- **Registers:**
  - Most clear to 0 at reset, and most are write-only.
  - The HV counter latches when EXTEN is read (EXLTEN = 0), as found from Mednafen.
  - TVSTAT (0x25F80004): VBLANK is bit 3, HBLANK bit 2.

## SEGA's SCU and dual CPU manuals: buses, DMA, the second SH-2

Also in `Manuals/`:
- the SCU User's Manual (ST-097-R5);
- **the SCU "Final Specs: Precautions"** (ST-097-R5, Oct 1994). It overrides the SCU manual where
  they differ: read it first;
- the Dual CPU User's Guide (ST-202-R1).

### The buses: who can reach what

- **The CPU bus** carries both SH-2s, Work RAM-H (0x06000000, 1 MB, SDRAM), Work RAM-L
  (0x00200000, 1 MB, DRAM), the BIOS ROM, backup RAM and the SMPC.
- **The A-bus** carries the CD block and the cartridge. **The B-bus** carries VDP1, VDP2 and the
  SCSP (sound). The SCU joins the three buses.
- The B-bus is 16 bits wide to each chip: the SCU sends 32 bits, which go on as two halves.
- **A CPU read on the A-bus or B-bus is always a full 4-byte access**, even for a byte or a word.
  Reading VRAM byte by byte costs far more than writing it. Writes go at their own size.
- **CPU writes to the B-bus come before SCU DMA**: while the CPU writes to a VDP non-stop, a DMA
  to the B-bus waits to start (precaution 12). While an SCU DMA runs between the A-bus and the
  B-bus, the CPU must not touch either (8): Work RAM-H stops being refreshed and can hang.
- Registers: VDP1's must be written in words (3). SCU registers only through the
  cache-through addresses 0x25FE0000.. (5): a cached read can touch a read-forbidden one.
- **The SH-2's cache doesn't watch the bus.** What a DMA or the other CPU writes isn't seen by a
  cached read. Read at the address + 0x20000000 (cache-through), or invalidate the cache's
  16-byte line by writing 0 (a 16-bit write) at the address + 0x40000000.

### SCU DMA (precautions win over the manual)

- **There are three levels.** Level 2 has the highest priority.
  - Level 0 moves up to 1 MB per transfer; levels 1 and 2 at most 4 KB.
  - Only 2 run at once in their priority order: a third, the DSP's DMA included, makes the order
    meaningless (20).
  - **Don't start level 2 while level 1 runs (35).**
- **What it can reach:**
  - **It can't use Work RAM-L (4):** only Work RAM-H, the A-bus and the B-bus. Anything to DMA to
    the VDPs (tables, textures, a level's data) must sit in Work RAM-H.
  - It can't write to the A-bus (1) or read from VDP2 (2).
- **Start factors:** V-Blank-IN, V-Blank-OUT, H-Blank-IN, Timer 0, Timer 1, an SCSP request,
  **VDP1's "sprite draw end"**, or a start bit. Each also needs the level's enable bit.
  - A trigger that comes while the level is still busy is held, once only (22). So a per-HBlank
    transfer must finish within one line.
- **Don't rewrite a level's registers while it runs (23): it hangs.** Clear its enable bit first.
  - Precisely: the mode/start factor register (D0MD, +0x14) and the address step register
    (D0AD, +0x0C). The read and write addresses aren't on the list.
  - Clearing the enable doesn't stop a transfer already running, and the status register that
    would say whether one is was deleted (13). For an HBlank-started transfer, wait a whole line
    after clearing it (the row counter moving on twice: `stop_dma` in `floor_fade_vdp2.c`),
    since such a transfer must end within its line (22).
  The transfer-count register is write-only (15).
- **Address steps:**
  - Read: the step bit must be 1 (4 bytes) everywhere but the A-bus I/O area (16, 17).
  - Write: B-bus writes may step 0, 2, 4 ... 128 bytes. Writing into every 32-byte VDP1 command
    takes a 32-byte step (the manual's example).
- **Indirect mode** runs a list of transfers in Work RAM-H, 3 longs each: count, write address,
  read address. Bit 31 of the last read address ends it. The list is aligned to its size,
  rounded up to a power of two (25).
- Transfers off a 4-byte boundary fall back to bytes at the ends.

### Interrupts and timers

- **SCU interrupts, level 15 (highest) down:** V-Blank-IN 0x40 (F), V-Blank-OUT 0x41 (E),
  H-Blank-IN 0x42 (D), Timer 0 0x43 (C), Timer 1 0x44 (B), DSP end 0x45 (A), sound 0x46 (9),
  SMPC 0x47 (8), pad 0x48 (8), DMA ends 0x49-0x4B (6, 6, 5), DMA illegal 0x4C (3), VDP1 sprite
  draw end 0x4D (2).
- **Don't write the interrupt status register (7).**
- **Timer 0** counts H-Blank-INs from V-Blank-OUT, and interrupts when it equals its compare
  value. It's a raster interrupt for any line:
  - in NTSC 224-line mode, 1 is just before the picture's first line and 224 just before its
    last;
  - 0 is the same moment as V-Blank-OUT;
  - over 263 never comes (30).
- **Timer 1** reloads every H-Blank-IN and counts down at ~7 MHz (one pixel), interrupting at 0:
  a point along a line, on every line or only Timer 0's (31).
- VDP1's sprite draw end interrupt marks when VDP1 finished. An FRC reading there would time
  VDP1 itself. Check what SGL already does with it before taking it.

### The SCU DSP

- An extra fixed-point processor in the SCU, at half the CPU clock (one step ~70 ns).
- Each step can do an ALU operation, two bus moves and a load at once.
- **Sizes:**
  - program RAM 256 x 32-bit words;
  - data RAM 4 x 64 words;
  - a 32 x 32 multiplier (48-bit result) and a 48-bit accumulator.
- It has its own DMA to Work RAM-H, the A-bus and the B-bus (B-bus into its data RAM: step 1,
  precaution 27).
- The CPU loads it through ports at 0x25FE0080-8C. Clear the end flag (bit 18) before starting
  it (26). Don't read the control port while it runs (36): that clears the overflow flag and
  can lose its end interrupt.
- It suits batched matrix x vertex work (it's how some games did their transforms). Nothing here
  uses it yet.

### The second SH-2 (dual CPU guide)

- **What they share:**
  - Both CPUs share every bus and run the same program.
  - When both reach outside their caches at once, one waits. That's why two CPUs aren't 2x.
  - Each has its own stack (set by the BIOS). Globals both write need care.
- **Starting the slave:**
  - Hold it in reset with SMPC SSHOFF.
  - Set its entry with `SYS_SETSINT(0x94, entry)`.
  - Release it with SMPC SSHON (0x02).
  - The BIOS gives it VBR 0x06000400 and SP 0x06001000, with only its FRT input-capture
    interrupt (vector 0x64, level 1) on.
- **Messages:** a 16-bit write to **0x21000000** signals the slave, to **0x21800000** the master,
  through the other CPU's FRT input capture.
  - Polling the capture flag (FTCSR bit 7) is quicker than the interrupt: mask the interrupt
    (TIER = 0x01) and clear the flag after each message.
  - A shared command word must be read cache-through (+0x20000000).
- **Changing the CPU clock (SMPC) resets the slave:** start it again afterwards.

## Look and feel decisions so far

- The floor grid is one scale in every view: it's the player's ruler for speed.
- Walls must look grounded on the grid (the 125-unit grid continues up the wall).
- The arena is walled in every direction, so a pink "far away" glow is wrong: the user wants
  darkness beyond: the floor fogs to black. The sky is on again (`SHOW_SKY`), seen only above
  the wall tops; the user liked it back once the fog was in. It's meant to be a never-ending
  sunset: never let it run to black (`tools/make_sky.py`; the top of the 1st person screen
  shows sky texel ~77, about 1.5 rows per texel).
- The curved world was tried and reverted: walls sank onto the still-flat floor.
- Turns: a double tap is a U-turn, held to a 50-unit gap (`turn_gate`, the user chose this over
  grid-snapped turns: instant turns matter more). The user will check visuals I didn't ask
  about: reproduce what they describe in slow motion (Z) before theorising, and prove a
  theory by building the old behaviour once (as with the gap 0 vs 50 trail hole).
- A tuning constant that depends on game rules (the message's frames, so the spectator's steady
  pan speed: `ATTRACT_TEXT_FRAMES`, `_PAN_X256`, `_PAN_PHASE`) is measured by a unit test that
  rides it with the real modules and prints the numbers, then pinned there: change the font,
  route or speeds and the test fails, saying the new values.
- Camera flights: keyframes through a Catmull-Rom spline (`intro_flight.c`) carry speed through
  each key; easing key to key stops at each. Keys a power of two of frames apart make t a shift.
  The ends start and stop at rest when the missing neighbour is the key on the other side.
  Plain Catmull-Rom overshoots a near key after a big move (10000 high down to 150 then 35 dipped
  below 35): `flight_at` limits each key's slope (Fritsch-Carlson: 0 where the value turns or
  holds, at most 3x the step either side). Other segment lengths: t = frame x (65536 / frames) >> 8.
- Drawing the arena's walls from a moving high camera: VDP1 wraps corners >1023 px off centre
  into spikes, and wall_clip assumes a level eye on the floor. Up high, draw only the walls in
  the box round where the screen's corner rays meet the floor and the wall tops (view.c,
  `arrival_footprint`); low down, keep the camera far from every wall and use wall_clip from its
  spot. From 10000 up even 1000-unit floor tiles break into dashes: the floor needs 2000.
- Big flat VDP1 surfaces (the raised ground round the arena, `surround.c`): cut each rectangle to
  the quad where the screen's corner rays meet its plane (Sutherland-Hodgman), so every corner is
  on screen and in front of the camera; flat colour, so the quads can be any size. Sort with
  `SORT_MIN` (nearest corner) to force a piece in front, `SORT_MAX` behind. A probe projecting the
  pieces on the PC proved the geometry right when the screen looked wrong.
- What looked like the surround covering the far wall was the far wall's own thin texture rows
  dropping in and out (2000-long LOD pieces): compare with a build without the change before
  hunting a bug in it.
- (SGL, before 2026-11-08; jo 2026 uses only level 0) SGL used all three SCU DMA levels: 2 streams PCM sound while a sample plays (`PCM_Trans`), 1
  its own transfers (`DMA_Trans`), 0 CD reads (GFS) and `DMA_ScuMemCopy`. The floor fog sat on
  level 2 and died at the first sound, unnoticed for days (in 1st person the fog is only ~10 rows
  under the horizon). It's on level 0 now, re-armed whole every VBlank. To find what a closed
  library touches, search the .A for the register addresses (big-endian) and read the member's
  symbol names (no nm here: a few lines of Python parse the archive).
- Finding it: compare pixel rows, not impressions ("the floor looks lit" was wrong about 1st
  person); a full-strength test value (fog black from 1 unit) shows at once whether the fog runs
  at all; then narrow by view (iso, menu and opening flight fogged; after the break nothing did).
  Print internal numbers on the debug line. VDP2 registers like CCCTL read back 0 (write-only).
- Finding a slowdown: time parts of the frame with the FRC (`perf_part_add`, utils/perf.h; the debug
  line shows `bots` and `trl`), then narrow with temporary extra parts. Mednafen times the SH-2
  closely (YabaSanshiro does not). `cpu` over 100% is real: that frame took longer than a vblank.
- A stress test must look like play: the debug serpentine left each trail's live end diagonal
  across the arena, and diagonals divide per piece, so it measured a cost play never has.
  Removing per-pair divisions barely helped there; the real waste was pieces built outside the
  view wedge and then dropped. Count what gets built against what gets drawn (`poly`).
- E2E step files written from PowerShell get CRLF and silently do nothing: the "new" shots were
  the last run's. Write them from Bash, and check the shots' times.
- Bash: `cat > file;` with no input waits on stdin for ever (it hung a command for 2 minutes).
- Anything the camera does must ease, never snap: the boost's FOV following the boost level
  (6 degrees a frame) looked like a glitch to the user. Ease it (`ease_out_step`) and ramp the
  cause too; check the biggest per-frame change in a unit test, and show a slow-motion GIF.
- Small integer speeds lose fractions: a share of speed 7 rounded *down* added nothing on the
  boost's first frame. Round (+ half before the shift), and test the low end of a range.
- Ease-out turn camera: 6/16 of the remaining angle per frame. Z compares it with the old turn
  and toggles 1/8 slow motion; slow motion is useful for any feel tuning.

- **Full-screen glitches that cost nothing** (the designed crash, `main.c` `apply_breach_glitch`):
  VDP2's colour offset (`jo_set_screen_color_filter_a`, -255..255 per channel, every layer at once);
  moving SGL's projection centre with `jo_3d_window` (slWindow's last two arguments) slides every
  polygon on screen while the VDP2 floor stays put; full-width VDP1 quads for tear bands. A colour
  offset left on while the terminal screen shows tints its black background (solid magenta).
- **E2E shot names must sort**: the contact sheet takes them alphabetically, so `breach_8` lands
  after `breach_35`. Pad them (`seq -w`).

- **(SGL, before 2026-11-08) High Work RAM ended at 0x060C0000 for us**: SGL's sort list and work areas started there. jo's
  malloc heap is a static array in our BSS (`JO_GLOBAL_MEMORY_SIZE_FOR_MALLOC`, jo's default 384 KB),
  so every static array we add eats into the same space. Past the line, a JO_DEBUG build doesn't
  start at all: a black screen, every E2E shot black. Check `.bss` start + size in `game.map`.
  Now: the heap is 352 KB (makefile) and main.c adds low Work RAM (1 MB at 0x00200000, unused by
  jo and SGL) as a second jo_malloc zone, used only once the first is full.
- **`view_floor_quad` refuses planes under 100 below the eye**, so it can't be used for the floor
  in 1st person (the eye's 35 up): `view_ground_quad` is the floor's.
- **VDP1 fill, not the SH-2, can be the limit**: huge near textured quads (tall walls a few hundred
  units either side) plus a VDP1 floor took two vblanks (cpu only 36%: look at fps, not cpu). The
  E2E debug line's fps over a run shows where: crop it from every shot into one strip
  (`convert x.png -crop 330x22+0+0`, then montage).
- **VDP1 walks the off-screen part of a polygon too**: a wall piece allowed 80 units from the eye
  reached ~1000 px above the screen and halved the frame rate; keep tall pieces' near end at
  HIGH_MIN_FORWARD (240). Drawing a floor twice (an underlay under tiles) did the same. Gaps between
  VDP1 tiles show the VDP2 floor for free: use that for seams.
- **A floor seen from a low eye is nearly edge on**: from 15 up, looking level, 16-unit seams on
  the floor vanish. Detail on the floor needs the eye higher or looking down.
- **SGL sort bits are the low 2 of ATTR.sort** (SORT_BFR 0, MIN 1, MAX 2, CEN 3): mask with ~3.
  SORT_BFR draws a polygon straight after the one before: for things that must lie on top.
- **Walls you can walk up to** (on foot, `level.c` corridor walls): the arena's strips start
  `HIGH_MIN_FORWARD` (240) in front, which leaves a hole in a wall you stand 40 from. Starting them
  10 in front only works with two more cuts, or VDP1 drops to 30 fps: each column's top capped
  just above the screen (eye + 1 x its distance in front; 2 x looking up), and each end slid into a
  cone in front (`wall_clip_strip_in_cone`, side <= 1.5 x forward). A wall running alongside you
  otherwise starts thousands of pixels off the screen's side, and VDP1 walks all of them.
- **Python edit scripts: no `\r\n` or `\\` in a Bash heredoc, and none in a Write-tool file either**
  if the file gets edited later: one round trip turned `'\r\n'` into a real line break and the
  script didn't parse. Use `chr(13)`, `chr(10)`. Several files here are CRLF (`wall_clip.*`,
  `test_wall_clip.c`): read with `newline=''`, replace on LF, write back as found.
- **E2E timing is real time**: `hold left 0.5` is half a second of the emulator, and the crash in
  `breach.steps` lands at slightly different times run to run. A press the game ignores (too
  soon) silently shifts every later step: look at the debug line's position and heading in each
  shot, not just the picture.
- **VDP1 fills a quad with lines between edges 0-3 and 1-2, one per pixel of the longer edge.**
  A tapered quad (a wall beside you: near column 800 px, far 50) listed top-first draws 800
  full-width lines crammed into the short end: heavy overdraw. Listed bottom-first (LB, LT, RT,
  RB; same winding) the lines run upright and each pixel's filled about once. Corridor walls
  (`level.c` `upright_lines`): 49 -> 60 fps looking at a wall. Only for untextured quads (a
  texture turns on its side); the trails already list theirs this way.
- **But upright is wrong for a quad that's wide and short on the screen**: the stairs' fronts,
  facing you, upright were ~480 lines of ~20 px each, and VDP1 went to two frames (30 fps,
  CPU only 67%); lying (top-first), ~20 lines, 60 fps. Each line costs VDP1 a set-up whatever
  its length. `lay_wide_pieces_flat` (level.c) picks per piece, after the tops are capped. Three
  limits, each found by measuring:
  - only square-on pieces (columns within 5:4), as a tapered one's lying lines slant;
  - only big ones (upright > 80 lines): small far fronts laid flat cost 60 -> 54 fps, cause unknown;
  - only faces across the view (forward along the face < 0.5): looping over the ~100 corridor
    pieces beside you cost ~6% CPU for nothing.
  Sizes without division: times d0 x d1 / F, a column's h0 x d1, the width |s1 x d0 - s0 x d1|.
- **Cutting a strip to the screen with straight edges: only where both ends are cut.** VDP1 draws
  a straight edge between two columns. Raise one column's bottom to the screen's bottom edge (a
  plane: eye - 4 - tan(37 + pitch) x forward) while its neighbour's bottom is above that plane,
  and the edge between them slices through the visible wall: the floor showed through the
  tower's front. `cap_band_bottoms` raises a column only if both neighbours' bottoms are at or
  under the plane; a box side gets an extra column where the plane crosses its bottom
  (`grey_side_crossing`, two divisions per side). `cap_band_tops` has the same flaw in principle
  (its margin, 1 x forward, hides it). The cap must run per `block_band_add`: columns of two
  strips side by side in a band aren't neighbours.
- **SORT_CEN draws a long wall piece over boxes that stand against it**: a 250-long corridor
  wall piece's middle came out nearer than the stairs' fronts, and it was drawn over their ends
  (a right-angled notch where the stairs meet the wall). The corridor's walls bound it, so they're
  sorted by their furthest corner (`SORT_MAX`, `draw_corridor_walls`): behind all that's in it.
- **Painter's sorting can't keep a room's insides and outsides apart.** The corridor's walls are
  sorted by their furthest corner, so anything outside them (the city's towers) was drawn over
  them. Fixed with a portal (`grey_portal`, level.c): from inside, a tower's drawn only through
  the doorway's wedge on the floor plan (sides cut where they cross its edges: a division each;
  tops by `corridor_clip` to the wedge as a quad); from outside, the corridor's insides aren't
  drawn. Cull whole boxes first: 21 towers worked out every frame took the CPU to 128% (30 fps).
- **The VDP2 floor can be moved down**: `slTranslate` before `jo_background_3d_plane_a_draw` in
  the world matrix (`view_ground_height`): the city's ground at -3000. Its fog still assumes 35.
- **E2E: a run's line varies with taps before it.** `hold left 0.2` is real time, so the yaw it
  leaves differs a frame or two between boots, and a 13 s run ends 300 units apart. For A/B
  films, hold straight; take the turns after you've stopped.
- **The debug line's 6th row times the level's parts** (`perf.h`: ar co sd tp fl gr bd wk, each
  a second's worst frame as % of a frame). Read it before guessing: down the corridor it showed the
  glass floor at 33% and the box sides at 27%, the walker and SGL at 0. `tests/e2e/parts.steps`
  stands still at the slow views. A new part: `int frc = jo_time_get_frc();` before,
  `perf_part_add(PERF_X, frc);` after, and a column on the row.
- **Fog a small flat polygon by its colour, not Gouraud**: one level per glass tile, the colour
  darkened as a Gouraud table would (`depth_fog_darken`). Per corner, each tile cost six fog
  levels and two tables written into VDP1 RAM while VDP1 drew.
- **A fast path for what's far and wholly in view**: the general strip path (cut to the cone,
  caps, crossing column, fill-line check) cost ~2000 cycles a side even for a stair front a few
  pixels high, 2000 away. `grey_box_whole` sends those as they are. The test for "safe uncut":
  every corner well in front (GLASS_MIN_FORWARD past the box's radius) and within VDP1's range
  (sideways from the view's reach, up and down 4 x the distance in front).
- **SH-2 64-bit maths**: multiplies compile to `dmuls` (cheap); a division is `__divdi3` and a
  shift by a varying count `__ashrdi3` (library calls). Check with `sh-elf-gcc -O2 -m2 -S` and grep
  for `__` calls. Bring sums down to 32 bits before dividing; shift 64-bit values by 1 in a loop.
- **Finding what VDP1 chokes on**: stand still (2.5 s, the averages are a second) at the worst
  spot (`tests/e2e/stairs_perf.steps`), then rebuild with one thing off at a time (a temporary
  `#define`, looped from PowerShell with `-replace`, one film per build). Low CPU with low fps
  is VDP1. Compare cpu between builds only under the same script: it moved 78 -> 84% with a pass
  that changed nothing on screen. A counter printed on the debug line (main.c) showed how many
  pieces the pass actually touched (0 there), which pointed at the loop's own cost.
- **Standing on roofs** (`game/walker.h`): roofs keep their real edges, and walls are roofs too
  tall to step onto (keep `WALKER_RADIUS` off them). Shrinking every rectangle by the radius
  left an 80 gap between two roofs that touch, so stairs and kerbs couldn't be walked. Height
  follows the highest roof under the whole footprint (step up as your foot reaches it); lips go
  by your middle. Stepping up only when the middle arrived put the eye (35 up) below the next
  step's top 3 units from its front, inside the 10 min-forward cut: you saw in under the stairs.
- **Parkour moves are worked out whole before they start** (`game/walker.c`: a jump's flight, a
  vault over, a slide), so they're predictable. Against the whole area (56 roofs) one slide's 36
  frames were a third of a frame of CPU in one frame (`wk34`, 46 fps). Cut the area to the roofs
  the move can reach first (`roofs_near`): same answer, a handful of roofs.
- **Level of detail for many small boxes**: twenty stair steps ~1700 off were ~10% of the CPU (each
  tested, cut, handed to SGL) at a pixel each. Far off, draw a stand-in (a ramp, a profile);
  only the drawing changes, never what the walker stands on. One-thing-off builds (a `tt_exp`
  switch flipped by a scratch script, three films in a row) found it in one go.
- **Occluders and exact cuts aren't free on VDP1-bound views**: a near box face as an occluder
  freed VDP1 but cost more CPU than it saved; cutting box tops exactly at the screen's edge made
  the crate's view *slower* (48 fps). Measure both chips before and after.
- **A higher jump changes what a "clean drop" is**: measured from the jump's top, a 45 hop made
  an 80 drop a long one and every jump down was refused. Measure the drop from where you left
  (`take_off`); keep the top only for which roof you can land on.
- **A higher, longer jump can hit walls the short one never reached**: the grey box's platform
  jump flew into the tower past the lower roof and was refused. Re-run the grey box tests after
  changing any move's reach.
- **The corridor's halves go by `across`, not your view**: facing out, its "right half" (across
  8 .. 16) is on your *left*. An E2E film strafed the wrong way and ran up the stairs.
- **`films.sh`'s tag must not start the shots' names**: it leaves out shots named `<tag>_*` (its
  own sheets), so tag `yd` with shots `yd_*` made a sheet of nothing. Pick another tag.
- **The host tests build with MSVC here** (no gcc on PATH): `/W4 /WX`, and a `printf` string
  edited through bash heredocs lost its `
` twice. Use the editor for C string literals.
- **`view_floor_quad` won't do a plane under 100 below the eye** (`VIEW_QUAD_MIN_ABOVE`, meant
  for the arena's wall tops): a roof you stand on is 35 below. `view_roof_quad` has no minimum.
- **Grey box overdraw**: a box's side hidden by a neighbour as high must be skipped, and one
  box spanning a stair's width drew its full-height front behind the stairs. Glass tiles under
  the boxes are dropped at `corridor_open` (each corner under some box). Climbing was 45-60 fps,
  30 stood at the stairs' foot: the fronts' fill lines (above), now 60.
- **Looking down from up high, the level cuts drop what's on screen** (`level.c`, on foot): the
  near-wall cuts (min forward 10, cone 1.5 x forward) and the tile/top side tests assume a level
  gaze. Looking down, a point d under the eye is d x sin(pitch) further in front of the tilted
  camera, so it's on screen further aside and nearer: `wall_clip_camera.side_extra16`, sized from
  a strip's deepest point (capped at 6 x its top's depth: keeps corners inside VDP1's range), and
  `corridor_near_below`. The wall body's split into bands by depth under the eye. Each mistake
  showed as the white floor (VDP2, white after the breach) through a gap.
- **The sky (RBG0 plane B) is drawn outside the world matrix**: it only follows yaw unless moved
  by the pitch too (`view_sky_rise`); looking up opened a band between its horizon and the floor's.
- **Glass tiles chained SORT_BFR after the far glass** go wrong when there's no far glass (near the
  corridor's end): the chain hangs off the first tile listed, behind you, and drew over the steps.
- **Python on Windows writes CRLF** unless `newline=''`: a .steps file with CRLF fails in the
  Docker runner ("unknown step: wait_boot"). And never `git checkout --` a file to undo a probe:
  it also throws away uncommitted work. Copy it aside and back.
- **Memory: each part's big buffers are on jo's heap, not static** (`src/scene_memory.h`): the
  arena's trail meshes and prologue message are freed at the hidden load, the corridor's taken at
  the breach and freed when the arena starts again. BSS went from ~1 KB to ~91 KB under the line
  (ends 0x060A9C00). `python tools/bss_sizes.py` lists every .o's BSS variables, statics too (the
  map names only globals; there's no nm). jo's heap overflows into low Work RAM (main.c's zone).
- **Freezing the screen (the hidden load, `input.c`)**: block in the game step, counting vblanks off
  `JO_VDP2_TVSTAT` bit 3 (as jo's core does); SGL gets no new frame, so VDP1 keeps showing the last
  picture. The frame after runs at most `FRAME_STEPS_MAX` steps. Loading 4 small TGAs from the
  emulated CD took ~1.5 s: the hold's ~2 s in all. Freeze on a frame you've chosen: on the tear's
  last frame it froze on a terminal-screen flash (an empty terminal and a cursor).
- **jo_sprite_add_tga works mid-game** (into VDP1 RAM, not BSS): textures for later scenes can wait
  for a load point. `tools/make_glass.py` makes the corridor's broken glass (`GLASS0-3.TGA`).
- **The game boots on foot in the corridor** (`BOOT_ON_FOOT 1`, src/input.c): round 5 entered,
  the corridor opened and the memory swapped (no freeze), you stood near the stairs. Corridor
  films run straight from the build: `tests\e2e\run_e2e.ps1 -Steps greybox.steps` (~50 s, with
  `wait_up` in place of `wait_boot`: the corridor's red-black isn't the arena's navy).
  `film_round.ps1` builds with 0 for the arena's rounds and puts it back after.

- **One 3D clipper for everything on foot** (`game/frustum.h`, level.c `level_face_add`): planes
  from the eye (near, the screen's edges, a portal's edges), Sutherland-Hodgman, the new corner
  worked out from an edge's inside end so two faces sharing it agree. It replaced every floor-plan
  cut on foot. What it took to make it as cheap as the special cases:
  - **The SH-2 shifts only by 1, 2, 8 and 16.** `1u << i` with a varying i, `>> 14`, and a signed
    `>> 8` or `>> 10` are library calls (`___ashlsi3`, `___ashiftrt_r4_N`). In a loop over planes
    and corners they were most of the time: walk a bit (`bit <<= 1`), and use `>> 16` of a 64-bit
    product (one `dmuls.l`). Grep the `-S` output for `___`.
  - **A box lined up with the axes against a plane**: its centre's distance plus and minus
    |nx| hx + |ny| hy + |nz| hz is its furthest and nearest corner (`frustum_classify_box`, tested
    against the 8 corners). Eight corners against eight planes was a sixth of the frame.
  - **Cull at the screen's edges, cut at a guard band** (2 x the screen): cutting every face that
    touched an edge was a fifth of the frame. But a face reaching past the guard band is cut at the
    screen's edge itself: the stairs' fronts (lying lines) cut at the guard band had VDP1 walk every
    line twice the screen's width, 30 fps.
  - **Test groups first** (the stairs as one box, the city's rows): `grey_groups`.
- **Don't fill the floor twice on VDP1**: a white floor under the tiles for the seams took the corridor
  to 30 fps and the stairs to 20, cut exactly to the screen or not. Seams as VDP1 polylines on the
  tiles' own corners cost no fill and no extra corners: a quad listed a, b, c, b draws just a-b and
  b-c (a polyline always closes, and a, b, c, d would draw d-a across the tile). One line a seam:
  each tile draws the three edges facing one way (corners 2 to 5), its neighbour the others.
- **Is it the new polygons' pixels?** The stairs went 60 to 30 fps with the outlines. Rebuilt with
  every outline a single point (the same polygons, no pixels), still 30: so not the lines. The view
  was at VDP1's edge already, from the stair fronts walked out to the guard band; cut at the
  screen's edge, it's 60 with the outlines. SGL's sort runs after our callbacks: not in `cpu`.
- **The VDP2 floor far below shimmers**: moved 3000 down under the city, its grid's lines were far
  finer than a pixel, so turning, the whole plane flipped between line and cell colours (navy and
  white). On foot its palette goes black (`set_floor_colours`): VDP2's the background, as the
  stable Saturn 3D games use it, and everything you stand on is VDP1.
- **A SORT_BFR chain needs a first polygon that's surely deepest**: moving the deepest tile to the
  front left its outline (added after it) before it, and the tile covered it. The glass floor's
  chain hangs off a dot at the floor's deepest point (`glass_anchor`: a quad with four equal corners).

## Reference projects in this repo

- **`Projects/FPS Collision demo`** (XL2, 2017: a rough first-person demo with walls, floors,
  slopes and stairs) is worth reading for Work RAM:
  - A two-level collision index: chunk boxes (`BSP_BB[i]`) each list a range of collision boxes
    (`BB_start..BB_end`). Test the chunks first, then only the boxes inside the ones you touch.
    Drawing culls by the same chunks. For us that's simpler as the plan's grid: cell = position
    shifted, and each cell lists its roofs.
  - A level is one file read into Low Work RAM (`Loading.h`, `LoadBinaryMAP`), laid out one after
    another: meshes, points, polygons, attributes, textures. Don't copy its 4-byte reads with a
    byte swap per value: write big-endian from the PC tool and read in big chunks.
  - Stair climbing is eased (at most 10 up a frame). Our `step_axis` snaps the feet: ease the eye
    only, Quake-style, so collision stays exact.
  - Blender to OBJ to a tool that writes the mesh and a separate list of collision boxes:
    collision is never the drawn mesh, as with our walker roofs.
  - Smaller ideas:
    - slopes as per-box height tables (800 bytes each; a slope x along >> 16 is cheaper);
    - auto-crouch under low ceilings;
    - camera roll = turn speed x 2;
    - turn speed halved each frame;
    - `slSetDepthTbl` and `slSetDepthLimit` to fade far polygons to black.
  - Its "faster one way" bug is rounding: `jo_cos_mult(...)/10` mixes `/` (rounds toward zero)
    with a shift (rounds down), so negative speeds lose more. Keep positions x256, with no
    division.
- **Running and vaulting (`game/walker.h`)**: the run is held forward (no button). At a full run,
  a lip is a jump only if `vault_off` flies the whole jump first, on a copy of the walker, and it
  lands clean. That's predictable because nothing steers in the air: the flight goes by
  `air_dx`/`air_dy`, fixed as you leave. Two traps found:
  - `fall()` tested the sideways move after the frame's drop, so on the landing frame the
    roof's own side stopped you, 1 under its top. It now moves across at the height before the
    drop.
  - A vault up via the footprint left your middle over the ground, so the next step was a lip
    and you jumped straight off. The vault up now carries you on by `WALKER_FOOT`.

  Tests that hold forward a long way now reach a run: start them near a lip or wall to test
  walking there.
- **Tracing a pure module on the PC**: copy the .c to the scratchpad, add printfs with a Python
  patch (`chr(92)+'n'` for the backslash), and build it with a small .bat that copies
  run_tests.bat's vcvars lines. Run the .exe in a separate call: run from the same .bat right
  after `cl`, it was "not recognized".
