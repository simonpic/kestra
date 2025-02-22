<template>
    <el-card shadow="never" v-if="execution">
        <table>
            <thead>
                <tr>
                    <th>
                        <duration :histories="execution.state.histories" />
                    </th>
                    <td v-for="(date, i) in dates" :key="i">
                        {{ date }}
                    </td>
                </tr>
            </thead>
            <tbody v-for="currentTaskRun in partialSeries" :key="currentTaskRun.id">
                <tr>
                    <th>
                        <el-tooltip placement="right" :persistent="false" transition="" :hide-after="0">
                            <template #content>
                                <code>{{ currentTaskRun.name }}</code>
                                <span v-if="currentTaskRun.task && currentTaskRun.task.value"><br>{{ currentTaskRun.task.value }}</span>
                            </template>
                            <code>{{ currentTaskRun.name }}</code>
                            <small v-if="currentTaskRun.task && currentTaskRun.task.value"> {{ currentTaskRun.task.value }}</small>
                        </el-tooltip>
                    </th>
                    <td :colspan="dates.length">
                        <el-tooltip placement="top" :persistent="false" transition="" :hide-after="0">
                            <template #content>
                                <span style="white-space: pre-wrap;">
                                    {{ currentTaskRun.tooltip }}
                                </span>
                            </template>
                            <div
                                :style="{left: Math.max(1, (currentTaskRun.start - 1)) + '%', width: currentTaskRun.width - 1 + '%'}"
                                class="task-progress"
                                @click="onTaskSelect(currentTaskRun.task)"
                            >
                                <div class="progress">
                                    <div
                                        class="progress-bar"
                                        :style="{left: currentTaskRun.left + '%', width: (100-currentTaskRun.left) + '%'}"
                                        :class="'bg-' + currentTaskRun.color + (currentTaskRun.running ? ' progress-bar-striped progress-bar-animated' : '')"
                                        role="progressbar"
                                    />
                                </div>
                            </div>
                        </el-tooltip>
                    </td>
                </tr>
                <tr v-if="taskRun && taskRun.id === currentTaskRun.id">
                    <td :colspan="dates.length + 1" class="p-0 pb-2">
                        <log-list
                            :task-run-id="taskRun.id"
                            :exclude-metas="['namespace', 'flowId', 'taskId', 'executionId']"
                            level="TRACE"
                            @follow="forwardEvent('follow', $event)"
                        />
                    </td>
                </tr>
            </tbody>
        </table>
    </el-card>
</template>
<script>
    import LogList from "../logs/LogList.vue";
    import {mapState} from "vuex";
    import State from "../../utils/state";
    import Duration from "../layout/Duration.vue";
    import Utils from "../../utils/utils";

    const ts = date => new Date(date).getTime();
    const TASKRUN_THRESHOLD = 50
    export default {
        components: {LogList, Duration},
        data() {
            return {
                colors: State.colorClass(),
                series: [],
                realTime: true,
                dates: [],
                duration: undefined,
                usePartialSerie: true,
            };
        },
        watch: {
            execution(newValue, oldValue) {
                if (oldValue.id !== newValue.id) {
                    this.realTime = true;
                    this.paint();
                } else {
                    this.compute()
                }
            },
            $route(oldValue, newValue) {
                if (oldValue.name === newValue.name) {
                    this.compute()
                }
            }
        },
        mounted() {
            this.paint();
        },
        computed: {
            ...mapState("execution", ["taskRun", "execution"]),
            taskRunsCount() {
                return this.execution && this.execution.taskRunList ? this.execution.taskRunList.length : 0
            },
            partialSeries() {
                return (this.series || []).slice(0, this.usePartialSerie ? TASKRUN_THRESHOLD : this.taskRunsCount)
            },
            start() {
                return this.execution ? ts(this.execution.state.histories[0].date) : 0;
            },
            tasks () {
                const rootTasks = []
                const childTasks = []
                const sortedTasks = []
                const tasksById = {}
                for (let task of (this.execution.taskRunList || [])) {
                    const taskWrapper = {task}
                    if (task.parentTaskRunId) {
                        childTasks.push(taskWrapper)
                    } else {
                        rootTasks.push(taskWrapper)
                    }
                    tasksById[task.id] = taskWrapper
                }
                while (childTasks.length) {
                    const taskWrapper = childTasks.pop()
                    const parentTask = tasksById[taskWrapper.task.parentTaskRunId]
                    if (parentTask) {
                        tasksById[taskWrapper.task.id] = taskWrapper
                        if (!parentTask.children) {
                            parentTask.children = []
                        }
                        parentTask.children.push(taskWrapper)
                    } else {
                        childTasks.unshift(taskWrapper)
                    }

                }
                const nodeStart = node => ts(node.task.state.histories[0].date)
                const childrenSort = nodes => {
                    nodes.sort((n1,n2) => {
                        return nodeStart(n1) > nodeStart(n2) ? 1 : -1
                    })
                    for (let node of nodes) {
                        sortedTasks.push(node.task)
                        if (node.children) {
                            childrenSort(node.children)
                        }
                    }
                }
                childrenSort(rootTasks)
                return sortedTasks
            }
        },
        methods: {
            forwardEvent(type, event) {
                this.$emit(type, event);
            },
            paint() {
                const repaint = () => {
                    this.compute()
                    if (this.realTime) {
                        const delay = this.taskRunsCount < TASKRUN_THRESHOLD ? 40 : 500
                        setTimeout(repaint, delay);
                    }
                }
                setTimeout(repaint);
                setTimeout(() => {
                    this.usePartialSerie = false
                }, 500);
            },
            compute() {
                this.computeSeries();
                this.computeDates();
            },
            delta() {
                return this.stop() - this.start;
            },
            stop() {
                if (!this.execution || State.isRunning(this.execution.state.current)) {
                    return +new Date();
                }

                return Math.max(...(this.execution.taskRunList || []).map(r => {
                    let lastIndex = r.state.histories.length - 1
                    return ts(r.state.histories[lastIndex].date)
                }));
            },
            computeSeries() {
                if (!this.execution) {
                    return;
                }

                if (!State.isRunning(this.execution.state.current)) {
                    this.stopRealTime();
                }

                const series = [];
                const executionDelta = this.delta(); //caching this value matters
                for (let task of this.tasks) {
                    let stopTs;
                    if (State.isRunning(task.state.current)) {
                        stopTs = ts(new Date());
                    } else {
                        const lastIndex = task.state.histories.length - 1;
                        stopTs = ts(task.state.histories[lastIndex].date);
                    }

                    const startTs = ts(task.state.histories[0].date);

                    const runningState = task.state.histories.filter(r => r.state === State.RUNNING);
                    const left = runningState.length > 0 ? ((ts(runningState[0].date) - startTs) / (stopTs - startTs) * 100) : 0;

                    const start = startTs - this.start;
                    let stop = stopTs - this.start - start;

                    const delta = stopTs - startTs;
                    const duration = this.$moment.duration(delta);

                    let tooltip = `${this.$t("duration")} : ${Utils.humanDuration(duration)}`

                    if (runningState.length > 0) {
                        tooltip += `\n${this.$t("queued duration")} : ${Utils.humanDuration((ts(runningState[0].date) - startTs) / 1000)}`;
                        tooltip += `\n${this.$t("running duration")} : ${Utils.humanDuration((stopTs - ts(runningState[0].date)) / 1000)}`;
                    }

                    let width = (stop / executionDelta) * 100
                    if (State.isRunning(task.state.current)) {
                        width = ((this.stop() - startTs) / executionDelta) * 100 //(stop / executionDelta) * 100
                    }

                    series.push({
                        id: task.id,
                        name: task.taskId,
                        start: (start / executionDelta) * 100,
                        width,
                        left: left,
                        tooltip,
                        color: this.colors[task.state.current],
                        running: State.isRunning(task.state.current),
                        task,
                        flowId: task.flowId,
                        namespace: task.namespace,
                        executionId: task.outputs && task.outputs.executionId
                    });
                }
                this.series = series;
            },
            computeDates() {
                const ticks = 5;
                const date = ts => this.$moment(ts).format("h:mm:ss");
                const start = this.start;
                const delta = this.delta() / ticks;
                const dates = [];
                for (let i = 0; i < ticks; i++) {
                    dates.push(date(start + i * delta));
                }
                this.dates = dates;
            },
            onTaskSelect(taskRun) {
                taskRun = this.taskRun && this.taskRun.id === taskRun.id ? undefined : taskRun;
                this.$store.commit("execution/setTaskRun", taskRun);
            },
            stopRealTime() {
                this.realTime = false
            }
        },
        unmounted() {
            this.stopRealTime();
        }
    };
</script>
<style lang="scss" scoped>
    .el-card {
        :deep(.el-card__body) {
            padding: 0;
        }

    }

    table {
        table-layout: fixed;
        width: 100%;
        color: var(--bs-body-color);

        & th, td {
            border-bottom: 1px solid var(--bs-border-color);
            padding: 0.3rem;
        }

        tr:last-child th, tr:last-child td {
            border-bottom: 0;
        }

        thead th, thead td {
            text-align: right;
        }

        thead {
            font-size: var(--font-size-sm);
            background-color: var(--bs-gray-200);

            th {
                width: 150px;
                background-color: var(--bs-gray-100-darken-5);
            }
        }
        th {
            max-width: 150px;
            white-space: nowrap;
            overflow: hidden;
            text-overflow: ellipsis;
        }

        code {
            font-size: 0.7rem;
        }

        tbody {
            th {
                code {
                    font-weight: normal;
                }
            }

            td {
                position: relative;

                .task-progress {
                    position: relative;
                    transition: all 0.3s;
                    min-width: 5px;

                    .progress {
                        height: 21px;
                        border-radius: var(--bs-border-radius-sm);
                        position: relative;
                        cursor: pointer;
                        background-color: var(--bs-gray-200);

                        .progress-bar {
                            position: absolute;
                            height: 21px;
                            transition: none;
                        }
                    }
                }
            }
        }
    }

    :deep(.log-wrapper .attempt-wrapper) {
        margin-bottom: 0;
    }

</style>
