<!-- readme-v2 -->
<p align="center"><img src=".github/banner.svg" alt="Folia Spawn Governor" width="100%"></p>

<p align="center"><img src="https://img.shields.io/badge/Folia-plugin-111827?style=flat-square" alt="Folia"> <img src="https://img.shields.io/badge/Java-plugin-111827?style=flat-square&logo=openjdk&logoColor=white" alt="Java"> <img src="https://img.shields.io/badge/tests-8_passing-22c55e?style=flat-square" alt="tests"> <a href="README.ko.md"><img src="https://img.shields.io/badge/README-한국어-111827?style=flat-square" alt="README"></a></p>

<p align="center">Throttles natural mob spawning <b>only in Folia regions that are actually lagging</b>.<br>Everywhere else the server stays plain vanilla.</p>

<table>
<tr><td align="center" width="25%"><h3>4 → 19.9</h3><sub>TPS, 50 players in one region</sub></td><td align="center" width="25%"><h3>42 ms</h3><sub>only acts above this tick time</sub></td><td align="center" width="25%"><h3>0</h3><sub>changes while a region is healthy</sub></td><td align="center" width="25%"><h3>60 bots</h3><sub>real benchmark, data included</sub></td></tr>
</table>

<p align="center"><img src="results/worst-case.svg" width="720" alt="Worst case with and without the governor"></p>

### Install

1. Download **SpawnGovernor-1.0.0.jar** from [Releases](https://github.com/jayden0903/folia-spawn-governor/releases/latest).
2. Put it in `plugins/` and restart. Nothing to configure: it stays idle until a region's tick time passes 42 ms.
3. `/spawngovernor` shows the regions it is tracking. Settings: `plugins/SpawnGovernor/config.yml`, then `/spawngovernor reload`.

Folia 26.2+, Java 25. On Paper it does nothing (Paper has one main thread, no regions).

<details><summary>Build from source</summary>

```bash
cd plugin && mvn package      # also runs the 8 unit tests
```

</details>

### Details

<details>
<summary><b>The short version</b></summary>

A Folia plugin that throttles natural mob spawning **only in regions that are actually lagging**, plus the benchmark

The short version: Folia ticks each region on its own thread, and a region can end up much bigger than you'd expect.
In my bot tests, about 50 spread-out players ended up chained into a single region, and that region ran at
4 TPS (247 ms per tick). With this plugin (and a NUMA / huge-page JVM setup) the same case held 19.4–19.9 TPS.
Regions that are fine are never touched, so most of the time the server is plain vanilla.

</details>

<details>
<summary><b>How it behaves</b></summary>

- Region's 5-second average tick ≤ 42 ms: nothing happens.
- Above that: natural spawn ticks are skipped for that region only, by cancelling Paper's
  `PlayerNaturallySpawnCreaturesEvent` for its players. Existing mobs follow normal despawn rules, so mob density drifts
  down only as far as needed, and spawning comes back once the region recovers.
- Still above 46 ms with spawning fully cut: mobs vanilla could despawn anyway (far hostiles, bats, fish, squid) go a bit
  sooner, without drops. Named, leashed or ridden mobs, animals, villagers, bosses and raids are never touched.

Animals, villagers, breeding, iron golems, raids, patrols, phantoms, spawners and farms keep working as usual.
The design went through four versions before this one; [docs/controller-design.md](docs/controller-design.md) has the history.

</details>

<details>
<summary><b>Why regions get so big</b></summary>

Folia's regioniser counts every chunk *holder*, including about 12 rings of `INACCESSIBLE` holders from ticket
propagation, and pads regions with empty sections. So players roughly 1,000–1,100 blocks apart still end up on one thread.
In the same 60-bot bench, lowering simulation distance to 4 only saved 10–25%, while spreading first-join spawns wider
(±4500 → ±10000) kept every region at 20 TPS with vanilla settings. Details and code references:
[docs/region-merging.md](docs/region-merging.md).

![Spawn spread](results/spawn-spread.svg)

</details>

<details>
<summary><b>Build and use</b></summary>

JDK 25 and Maven.

```bash
cd plugin && mvn package      # also runs the 8 unit tests
```

Drop `plugin/target/spawn-governor-1.0.0.jar` into `plugins/` and restart. Settings live in
`plugins/SpawnGovernor/config.yml` (target, hold band, despawn assist). `/spawngovernor` lists tracked regions,
`/spawngovernor reload` applies config changes, and overloaded regions are logged every 30 s.

</details>

<details>
<summary><b>Running the benchmark yourself</b></summary>

Only on a separate test server with `online-mode=false` and a copy of your world, never with real players.

1. Put `bench/probe` on the test server.
2. `cd bench/clients && npm install`, then `node clients.js <host> <port> <version> bench` and raise `count` in `control.json`.
3. Arrange the bots from the console: `/spawnbench chain 400`, `grid 650` or `random 10000`.
4. Capture with `/profiler world 0 0 177 8000` and summarise with `bench/analysis/parse_profiler.py <folder>`.

Raw numbers are in `results/measurements.csv` (48 phases).

</details>

<details>
<summary><b>Caveats</b></summary>

Everything was measured with bots, not people. In village-heavy areas with ~50 spread players it bottoms out around
60–70 ms, because it deliberately leaves animals and villagers alone. `RegionClock` reads Folia internals through
reflection, so a Folia update can break it. More in [docs/limitations.md](docs/limitations.md).

GPL-3.0-only.

</details>


<p align="center"><sub>Made by <a href="https://github.com/jayden0903">jayden0903</a> · measured on a real Folia server</sub></p>
