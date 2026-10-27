<template>
  <DashboardPage
    db-name="perfintDev"
    table="air"
    persistent-id="air_dashboard"
    initial-machine="Linux EC2 C6id.8xlarge (32 vCPU Xeon, 64 GB)"
    :with-installer="false"
  >
    <template #configurator>
      <SelectButton
        v-model="selectedSizes"
        :options="sizes"
        option-label="short"
        option-value="id"
        multiple
        :allow-empty="false"
        aria-label="The long session sizes that the charts show"
      />
    </template>
    <template
      v-for="group in longSessionGroups"
      :key="group.title"
    >
      <Divider :title="group.title" />
      <section>
        <GroupProjectsChart
          v-for="chart in group.charts"
          :key="chart.label"
          :label="chart.label"
          :description="chart.description"
          :measure="chart.measure"
          :projects="longSessionProjects"
          :aliases="longSessionAliases"
          :legend-formatter="sizeLegend"
        />
      </section>
    </template>
    <template
      v-for="group in acpGroups"
      :key="group.title"
    >
      <Divider :title="group.title" />
      <section>
        <GroupProjectsChart
          v-for="chart in group.charts"
          :key="chart.label"
          :label="chart.label"
          :description="chart.description"
          :measure="chart.measure"
          :projects="chart.projects"
          :legend-formatter="acpLegend"
        />
      </section>
    </template>
    <template
      v-for="group in otherGroups"
      :key="group.title"
    >
      <Divider :title="group.title" />
      <section>
        <GroupProjectsChart
          v-for="chart in group.charts"
          :key="chart.label"
          :label="chart.label"
          :description="chart.description"
          :measure="chart.measure"
          :projects="[group.project]"
          :legend-formatter="measureLegend"
        />
      </section>
    </template>
  </DashboardPage>
</template>

<script setup lang="ts">
import { useStorage } from "@vueuse/core"
import { computed } from "vue"
import GroupProjectsChart from "../charts/GroupProjectsChart.vue"
import { SERIES_NAME_SEPARATOR } from "../common/DataQueryExecutor"
import DashboardPage from "../common/DashboardPage.vue"
import Divider from "../common/Divider.vue"

interface AirChart {
  label: string
  description: string
  measure: string[]
}

interface AirGroup {
  title: string
  charts: AirChart[]
}

/**
 * The sizes of the long session test. The metric names and the projects keep the size names of the test, M, XL and XXL,
 * so their history and their degradation alerts stay. The legend names each size by what it is.
 */
const sizes = [
  {
    id: "M",
    short: "Small: 30 turns",
    label: "Small session: 30 turns, 9 MB journal",
  },
  {
    id: "XL",
    short: "Medium: 200 turns",
    label: "Medium session: 200 turns, 63 MB journal",
  },
  {
    id: "XXL",
    short: "Large: 500 turns",
    label: "Large session: 500 turns, 160 MB journal",
  },
]
const sizeLabels = new Set(sizes.map((size) => size.label))

/**
 * The sizes that every long session chart shows, chosen once for the whole page. A legend entry still hides one line of
 * one chart. The choice stays for the next visit of this browser.
 */
const selectedSizes = useStorage<string[]>(
  "air_dashboard_sizes",
  sizes.map((size) => size.id)
)
const shownSizes = computed(() => sizes.filter((size) => selectedSizes.value.includes(size.id)))
const longSessionProjects = computed(() => shownSizes.value.map((size) => `air/long-session/${size.id}`))
const longSessionAliases = computed(() => shownSizes.value.map((size) => size.label))

/** One chart of a long session step with one line for each size. `%s` in [template] is the size of the metric name. */
function bySize(label: string, description: string, template: string): AirChart {
  return {
    label,
    description,
    measure: sizes.map((size) => template.replace("%s", size.id)),
  }
}

/** A line of a long session chart is its size. The measure part of the series name is the same step for each size. */
function sizeLegend(name: string): string {
  return name.split(SERIES_NAME_SEPARATOR).find((part) => sizeLabels.has(part)) ?? name
}

const jumpPoints = [
  { point: 0, label: "the start of the history" },
  { point: 20, label: "20 % of the history" },
  { point: 40, label: "40 % of the history" },
  { point: 60, label: "60 % of the history" },
  { point: 80, label: "80 % of the history" },
]

const longSessionGroups: AirGroup[] = [
  {
    title: "Long session: open after an IDE restart",
    charts: [
      bySize(
        "Open a session: from the click in Agent Sessions to the open tab",
        "The user double-clicks the session row after an IDE restart. The time runs from the mouse event to the moment the session screen shows. The tab can still be empty.",
        "open.%s.gestureToTabMs"
      ),
      bySize(
        "Open a session: from the click to the first visible text",
        "Until the loading cover goes away. The page removes it when the text and the code editors near the viewport are built. A removal by the timeout of the page fails the test.",
        "open.%s.coverRemovedMs"
      ),
      bySize(
        "Open a session: from the click to a settled transcript",
        "The later of the cover removal and the last move or height change of the transcript.",
        "open.%s.gestureToSettledMs"
      ),
    ],
  },
  {
    title: "Long session: open, the steps that the IDE logs",
    charts: [
      bySize(
        "Open, backend: replay the session journal into the projection",
        "From the request of the projection until it is ready, with the lock waits. The [AIR_SESSION_RESTORE] rich projection ready line of the IDE log.",
        "checkpoint.%s.rich_projection_ready.totalMs"
      ),
      bySize(
        "Open, frontend: apply the session snapshot and render it on the EDT",
        "From the arrival of the snapshot in the frontend until the screen rendered it. The [AIR_SESSION_RESTORE] frontend reload applied line.",
        "checkpoint.%s.frontend_reload_applied.totalMs"
      ),
      bySize(
        "Open, frontend: build the rows of the transcript",
        "Build the rows of the messages, the tool calls and the code blocks. The [AIR_SESSION_RESTORE] frontend transcript rows applied line.",
        "checkpoint.%s.frontend_transcript_rows_applied.elapsedMs"
      ),
      bySize(
        "Open, frontend: from the session panel to the first painted text",
        "From the creation of the session panel until the first paint of the transcript. The [AIR_SESSION_RESTORE] frontend first content paint finished line.",
        "checkpoint.%s.frontend_first_content_paint_finished.totalSincePanelCreationMs"
      ),
    ],
  },
  {
    title: "Long session: continue the old session with a prompt",
    charts: [
      bySize(
        "Continue a session: from Send to the session/load request at the agent",
        "Air starts the agent process and sends session/load. The time includes the start of the agent.",
        "continue.%s.sendToLoadMs"
      ),
      bySize("Continue a session: from Send to the prompt at the agent", "Until the agent got the prompt that the user sent.", "continue.%s.sendToPromptMs"),
    ],
  },
  {
    title: "Long session: a long streamed answer",
    charts: [
      bySize(
        "Long answer: from the last chunk to the completed turn in the UI",
        "Until the running status row of the turn goes away. It shows how far the UI lags behind the agent.",
        "answer.%s.streamEndToCompletedMs"
      ),
      bySize(
        "Long answer: from the last chunk to a settled transcript",
        "Until the end of the first frame with the built text after the last change of the transcript.",
        "answer.%s.streamEndToSettledMs"
      ),
    ],
  },
  {
    title: "Long session: typing and scrolling latency",
    charts: [
      bySize(
        "Mouse wheel up from the bottom of the transcript: p95 latency of one wheel event",
        "10 bursts of 400 wheel events, one each 16 ms. The time runs to the end of the first frame that shows the scroll. The p95 of the worst burst. An event over 2 s fails the test.",
        "scroll.%s.wheelP95Ms"
      ),
      bySize(
        "Mouse wheel up from the bottom of the transcript: the slowest single wheel event",
        "The slowest wheel event to the frame of its scroll in the 10 bursts.",
        "scroll.%s.wheelMaxMs"
      ),
      bySize(
        "Mouse wheel right after the long answer: p95 latency of one wheel event",
        "200 wheel events over the transcript next to the long answer, each to the frame of its scroll.",
        "answer.%s.wheelP95Ms"
      ),
      bySize(
        "Typing prose into the prompt: p95 latency of one character",
        "1000 characters of prose at the pace of a fast typist. The time runs from the key event to the paint of the editor.",
        "prompt.%s.proseP95Ms"
      ),
      bySize(
        "Typing code into the prompt: p95 latency of one character",
        "50 lines of code at the pace of a fast typist, from the key event to the paint of the editor. A character over 500 ms fails the test.",
        "prompt.%s.codeP95Ms"
      ),
    ],
  },
  {
    title: "Long session: jumps far into the history",
    charts: jumpPoints.map(({ point, label }) =>
      bySize(
        `Jump to ${label}: time until the transcript settles`,
        "The scrollbar jumps to this point of the history, as a user drags it there. The transcript builds far text only near the visible area. The time runs to the end of the first frame with no empty part in the viewport, after the last change. Over 60 s fails the test.",
        `scroll.%s.jump${point}.settledMs`
      )
    ),
  },
  {
    title: "Long session: memory",
    charts: [
      bySize("Live heap before the open of the session", "The used heap after a full garbage collection, before the click.", "heap.%s.before_the_open.MiB"),
      bySize("Live heap right after the open of the session", "The difference to the heap before the open is what the open session keeps.", "heap.%s.after_the_open.MiB"),
      bySize(
        "Live heap at the end of the scenario, after a full garbage collection",
        "What the IDE keeps after the open, the scroll, the answer and the typing. Over 3/4 of the max heap fails the test.",
        "heap.%s.heapLive.MiB"
      ),
      bySize(
        "Peak heap during the scenario",
        "The largest used heap after a pause of the garbage collector, without the collections that the test calls itself.",
        "heap.%s.workingSet.MiB"
      ),
    ],
  },
]

/** The readable name of each measure of the charts that have one line for each measure. */
const measureLabels: Record<string, string> = {
  "manyAgents.edtP95Ms": "EDT latency, p95",
  "manyAgents.edtMaxMs": "EDT latency, max",
  "manyAgents.focusedTabLagMs": "Focused tab",
  "manyAgents.openTabLagMaxMs": "Slowest of the hidden open tabs",
  "manyAgents.heapLive.MiB": "Live heap after a full garbage collection",
  "manyAgents.workingSet.MiB": "Peak heap after a collector pause",
  "manyAgents.stream.gcPauseMs": "Collector pauses in the stream",
  "sessionList.firstShowMs": "First show after the restart",
  "sessionList.refreshMedianMs": "Refresh, median of 9",
  "sessionList.wheelP95Ms": "Wheel event latency, p95",
  "sessionList.wheelMaxMs": "Wheel event latency, max",
  "sessionList.heapLive.MiB": "Live heap after a full garbage collection",
  "sessionList.workingSet.MiB": "Peak heap after a collector pause",
}

function measureLegend(name: string): string {
  const measure = name.split(SERIES_NAME_SEPARATOR).find((part) => part in measureLabels)
  return measure == undefined ? name : measureLabels[measure]
}

const otherGroups: (AirGroup & { project: string })[] = [
  {
    title: "Many agents: 10 sessions stream at once for 5 minutes",
    project: "air/many-agents",
    charts: [
      {
        label: "Ten agents stream at once: EDT latency",
        description: "10 sessions of a test agent stream with 5 subagents each. One is in the focused tab, 3 in hidden tabs, 6 without a tab.",
        measure: ["manyAgents.edtP95Ms", "manyAgents.edtMaxMs"],
      },
      {
        label: "Ten agents stream at once: from the end of the stream to the completed turn in a tab",
        description: "Until the running status row of the turn goes away in the tab.",
        measure: ["manyAgents.focusedTabLagMs", "manyAgents.openTabLagMaxMs"],
      },
      {
        label: "Ten agents stream at once: memory",
        description: "Over 3/4 of the max heap fails the test.",
        measure: ["manyAgents.heapLive.MiB", "manyAgents.workingSet.MiB"],
      },
      {
        label: "Ten agents stream at once: garbage collection time",
        description: "The pauses of the collector in the stream, without the collections that the test calls itself.",
        measure: ["manyAgents.stream.gcPauseMs"],
      },
    ],
  },
  {
    title: "Session list: the Agent Sessions window with 2500 sessions",
    project: "air/session-list",
    charts: [
      {
        label: "Session list of 2500 sessions: first show after a restart, and a refresh",
        description: "10 sessions of 2000 turns, 100 of 150 turns, the rest of one turn, each with a valid summary.json. A journal read fails the test.",
        measure: ["sessionList.firstShowMs", "sessionList.refreshMedianMs"],
      },
      {
        label: "Session list of 2500 sessions: mouse wheel latency over the tree",
        description: "200 wheel events over the Agent Sessions tree. Only a burst that moved the tree has a value.",
        measure: ["sessionList.wheelP95Ms", "sessionList.wheelMaxMs"],
      },
      {
        label: "Session list of 2500 sessions: memory",
        description: "Over 3/4 of the max heap fails the test.",
        measure: ["sessionList.heapLive.MiB", "sessionList.workingSet.MiB"],
      },
    ],
  },
]

/**
 * The ACP adapter performance tests (lane acp-perf, IJAI-2949). Each night they install the latest claude-acp, codex-acp
 * and junie from the ACP registry and talk to them with a bare JSON-RPC client and a mock model, without Air. A project is
 * one adapter, `air/acp/<agent>`. The big sessions have the sizes of the long session test, and a measure names its size.
 * The test writes the sessions of junie through prompts, so junie has no sizes and no list scenarios.
 */
const acpAgents = [
  { project: "air/acp/claude-acp", label: "claude-acp" },
  { project: "air/acp/codex-acp", label: "codex-acp" },
  { project: "air/acp/junie", label: "junie" },
]
const acpAgentLabels = new Map(acpAgents.map((agent) => [agent.project, agent.label]))
const acpSizedProjects = acpAgents.filter((agent) => agent.label != "junie").map((agent) => agent.project)
const acpAllProjects = acpAgents.map((agent) => agent.project)

const acpMeasureLabels: Record<string, string> = {
  "live.agentMs": "first turn",
  "live.second.agentMs": "second turn",
  "live.chunkLagP50Ms": "p50",
  "live.chunkLagP95Ms": "p95",
  "live.agentRssPeakMiB": "first turn",
  "live.second.agentRssPeakMiB": "second turn",
  "list.1k.activeFirst.warmMs": "1 000 sessions",
  "list.10k.activeFirst.warmMs": "10 000 sessions",
  "list.50k.activeFirst.warmMs": "50 000 sessions",
  "list.initialize.coldMs": "61 000 sessions in the store",
  "archive.ms": "archive",
  "unarchive.ms": "unarchive",
  "start.initializeMs": "initialize",
}

/** A line of an ACP chart is one adapter, with the size or the step that its measure names. */
function acpLegend(name: string): string {
  const parts = name.split(SERIES_NAME_SEPARATOR)
  const agent = parts.map((part) => acpAgentLabels.get(part)).find((label) => label != undefined) ?? name
  const measure = parts.find((part) => !acpAgentLabels.has(part)) ?? ""
  const size = sizes.find((it) => measure.includes(`.${it.id}.`))
  const step = size == undefined ? acpMeasureLabels[measure] : size.short
  return step == undefined ? agent : `${agent}, ${step}`
}

interface AcpChart extends AirChart {
  projects: string[]
}

/** One chart of an ACP step with one line for each adapter and size. `%s` in [template] is the size of the metric name. */
function acpBySize(label: string, description: string, template: string): AcpChart {
  return { label, description, measure: sizes.map((size) => template.replace("%s", size.id)), projects: acpSizedProjects }
}

const acpGroups: { title: string; charts: AcpChart[] }[] = [
  {
    title: "ACP adapters: open and continue a session of each size",
    charts: [
      {
        label: "Start an adapter: the initialize request",
        description: "The start of the adapter process and its answer to initialize, median of 3 starts. Air pays it for each new session.",
        measure: ["start.initializeMs"],
        projects: acpAllProjects,
      },
      acpBySize(
        "Open a session: session/load replays the history",
        "From session/load until its answer, with every replayed update. All user turns must arrive, and the model must not be called.",
        "open.%s.replayMs"
      ),
      acpBySize("Open a session: peak memory of the adapter", "The largest sum of the resident memory of the adapter process tree during the replay.", "open.%s.agentRssPeakMiB"),
      acpBySize(
        "Continue a session: from the prompt to the request to the model",
        "From session/prompt until the headers of the request reach the mock model. The time that the adapter prepares the turn: it reads the history and builds the request.",
        "continue.%s.toModelMs"
      ),
      acpBySize(
        "Continue a session: the size of the request to the model",
        "The body of the request of the continued turn, in bytes. It grows with the history that the adapter sends.",
        "continue.%s.requestBytes"
      ),
    ],
  },
  {
    title: "ACP adapters: a live turn of 241 model steps",
    charts: [
      {
        label: "Live turn: the time of the adapter",
        description:
          "20 000 text parts, 200 file reads, 20 outputs of 300 000 characters and 20 images, then a second turn in the same session. The time of the turn without the time of the mock model.",
        measure: ["live.agentMs", "live.second.agentMs"],
        projects: acpAllProjects,
      },
      {
        label: "Live turn: the delay of a text part",
        description: "From the flush of a stamped text part by the mock model to its arrival at the ACP client. junie sends no text parts, so it has no line.",
        measure: ["live.chunkLagP50Ms", "live.chunkLagP95Ms"],
        projects: acpSizedProjects,
      },
      {
        label: "Live turn: peak memory of the adapter",
        description: "The largest sum of the resident memory of the adapter process tree during the turn.",
        measure: ["live.agentRssPeakMiB", "live.second.agentRssPeakMiB"],
        projects: acpAllProjects,
      },
    ],
  },
  {
    title: "ACP adapters: the session list",
    charts: [
      {
        label: "Session list: the first page of 100 sessions",
        description: "session/list of a project with 1 000, 10 000 and 50 000 sessions, warm. A time that grows with the count means that a page reads the whole history.",
        measure: ["list.1k.activeFirst.warmMs", "list.10k.activeFirst.warmMs", "list.50k.activeFirst.warmMs"],
        projects: acpSizedProjects,
      },
      {
        label: "Session list: the first start after the index is gone",
        description: "The initialize of an adapter on a store of 61 000 sessions without its index, as after an update. Codex builds its state database here.",
        measure: ["list.initialize.coldMs"],
        projects: acpSizedProjects,
      },
      {
        label: "Archive and unarchive a session of the size XXL",
        description: "From _session/archive or _session/unarchive until session/list shows the session in its new place.",
        measure: ["archive.ms", "unarchive.ms"],
        projects: acpSizedProjects,
      },
    ],
  },
]
</script>
