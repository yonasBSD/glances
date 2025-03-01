<template>

    <!-- Display processes -->
    <section class="plugin" id="processlist" v-if="!args.programs">
        <div class="extendedstats" v-if="extended_stats !== null">
            <div>
                <span class="title">Pinned thread: </span>
                <span>{{ $filters.limitTo(extended_stats.cmdline, 80) }}</span>
                <span><button class="button" v-on:click="disableExtendedStats()">Upin</button></span>
            </div>
            <div>
                <span>CPU Min/Max/Mean: </span>
                <span class="careful">{{ $filters.number(extended_stats.cpu_min, 1)
                    }}% / {{
                        $filters.number(extended_stats.cpu_max, 1) }}% / {{ $filters.number(extended_stats.cpu_mean, 1)
                    }}%</span>
                <span>Affinity: </span>
                <span class="careful">{{ extended_stats.cpu_affinity | length }}</span>
            </div>
            <div>
                <span>MEM Min/Max/Mean: </span>
                <span class="careful">{{ $filters.number(extended_stats.memory_min, 1) }}% / {{
                    $filters.number(extended_stats.memory_max, 1) }}% / {{ $filters.number(extended_stats.memory_mean,
                        1)
                    }}%</span>
                <span>Memory info: </span>
                <span class="careful">
                    {{ $filters.dictToString(extended_stats.memory_info) }}
                </span>
            </div>
        </div>
        <div class="table-responsive d-lg-none">
            <table class="table table-sm table-borderless table-striped table-hover">
                <thead>
                    <tr>
                        <td scope="col" :class="['sortable', sorter.column === 'cpu_percent' && 'sort']"
                            @click="$emit('update:sorter', 'cpu_percent')"
                            v-show="!getDisableStats().includes('cpu_percent')">
                            CPU%
                        </td>
                        <td scope="col" :class="['sortable', sorter.column === 'memory_percent' && 'sort']"
                            @click="$emit('update:sorter', 'memory_percent')"
                            v-show="!getDisableStats().includes('memory_percent')">
                            MEM%
                        </td>
                        <td scope="col" v-show="!getDisableStats().includes('pid')">
                            PID
                        </td>
                        <td scope="col" :class="['sortable', sorter.column === 'username' && 'sort']"
                            @click="$emit('update:sorter', 'username')"
                            v-show="!getDisableStats().includes('username')">
                            USER
                        </td>
                        <td scope="col" :class="['sortable', sorter.column === 'name' && 'sort']"
                            @click="$emit('update:sorter', 'name')" v-show="!getDisableStats().includes('cmdline')">
                            Command (click to pin)
                        </td>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="(process, processId) in processes" :key="processId" @click="setExtendedStats(process)"
                        style="cursor: pointer">
                        <td scope="row" :class="getCpuPercentAlert(process)"
                            v-show="!getDisableStats().includes('cpu_percent')">
                            {{ process.cpu_percent == -1 ? '?' : $filters.number(process.cpu_percent, 1) }}
                        </td>
                        <td scope="row" :class="getMemoryPercentAlert(process)"
                            v-show="!getDisableStats().includes('memory_percent')">
                            {{ process.memory_percent == -1 ? '?' : $filters.number(process.memory_percent, 1) }}
                        </td>
                        <td scope="row" v-show="!getDisableStats().includes('pid')">
                            {{ process.pid }}
                        </td>
                        <td scope="row" class="text-truncate"
                            v-show="args.process_short_name && !getDisableStats().includes('cmdline')">
                            {{ process.name }}
                        </td>
                        <td scope="row" class="text-truncate"
                            v-show="!args.process_short_name && !getDisableStats().includes('cmdline')">
                            {{ process.cmdline }}
                        </td>
                    </tr>
                </tbody>
            </table>
        </div>
        <div class="table-responsive-md d-none d-lg-block">
            <table class="table table-sm table-borderless table-striped table-hover">
                <thead>
                    <tr>
                        <td scope="col" :class="['sortable', sorter.column === 'cpu_percent' && 'sort']"
                            @click="$emit('update:sorter', 'cpu_percent')"
                            v-show="!getDisableStats().includes('cpu_percent')">
                            CPU%
                        </td>
                        <td scope="col" :class="['sortable', sorter.column === 'memory_percent' && 'sort']"
                            @click="$emit('update:sorter', 'memory_percent')"
                            v-show="!getDisableStats().includes('memory_percent')">
                            MEM%
                        </td>
                        <td scope="col" v-show="!getDisableStats().includes('memory_info')">
                            VIRT
                        </td>
                        <td scope="col" v-show="!getDisableStats().includes('memory_info')">
                            RES
                        </td>
                        <td scope="col" v-show="!getDisableStats().includes('pid')">
                            PID
                        </td>
                        <td scope="col" :class="['sortable', sorter.column === 'username' && 'sort']"
                            @click="$emit('update:sorter', 'username')"
                            v-show="!getDisableStats().includes('username')">
                            USER
                        </td>
                        <td scope="col" :class="['sortable', sorter.column === 'timemillis' && 'sort']"
                            @click="$emit('update:sorter', 'timemillis')"
                            v-show="!getDisableStats().includes('cpu_times')">
                            TIME+
                        </td>
                        <td scope="col" :class="['sortable', sorter.column === 'num_threads' && 'sort']"
                            @click="$emit('update:sorter', 'num_threads')"
                            v-show="!getDisableStats().includes('num_threads')">
                            THR
                        </td>
                        <td scope="col" v-show="!getDisableStats().includes('nice')">NI</td>
                        <td scope="col" v-show="!getDisableStats().includes('status')">S
                        </td>
                        <td scope="col" class="" :class="['sortable', sorter.column === 'io_counters' && 'sort']"
                            v-show="ioReadWritePresentProcesses && !getDisableStats().includes('io_counters')"
                            @click="$emit('update:sorter', 'io_counters')">
                            IORps
                        </td>
                        <td scope="col" class="text-start"
                            :class="['sortable', sorter.column === 'io_counters' && 'sort']"
                            v-show="ioReadWritePresentProcesses && !getDisableStats().includes('io_counters')"
                            @click="$emit('update:sorter', 'io_counters')">
                            IOWps
                        </td>
                        <td scope="col" :class="['sortable', sorter.column === 'name' && 'sort']"
                            @click="$emit('update:sorter', 'name')" v-show="!getDisableStats().includes('cmdline')">
                            Command (click to pin)
                        </td>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="(process, processId) in processes" :key="processId"
                        @click="setExtendedStats(process.pid)" style="cursor: pointer">
                        <td scope="row" :class="getCpuPercentAlert(process)"
                            v-show="!getDisableStats().includes('cpu_percent')">
                            {{ process.cpu_percent == -1 ? '?' : $filters.number(process.cpu_percent, 1) }}
                        </td>
                        <td scope="row" :class="getMemoryPercentAlert(process)"
                            v-show="!getDisableStats().includes('memory_percent')">
                            {{ process.memory_percent == -1 ? '?' : $filters.number(process.memory_percent, 1) }}
                        </td>
                        <td scope="row" v-show="!getDisableStats().includes('memory_info')">
                            {{ $filters.bytes(process.memvirt) }}
                        </td>
                        <td scope="row" v-show="!getDisableStats().includes('memory_info')">
                            {{ $filters.bytes(process.memres) }}
                        </td>
                        <td scope="row" v-show="!getDisableStats().includes('pid')">
                            {{ process.pid }}
                        </td>
                        <td scope="row" v-show="!getDisableStats().includes('username')">
                            {{ process.username }}
                        </td>
                        <td scope="row" class="" v-show="!getDisableStats().includes('cpu_times')">
                            {{ process.timeforhuman }}
                        </td>
                        <td scope="row" class="" v-if="process.timeplus == '?'"
                            v-show="!getDisableStats().includes('cpu_times')">?</td>
                        <td scope="row" class="" v-show="!getDisableStats().includes('num_threads')">
                            {{ process.num_threads == -1 ? '?' : process.num_threads }}
                        </td>
                        <td scope="row" :class="{ nice: process.isNice }" v-show="!getDisableStats().includes('nice')">
                            {{ $filters.exclamation(process.nice) }}
                        </td>
                        <td scope="row" :class="{ status: process.status == 'R' }"
                            v-show="!getDisableStats().includes('status')">
                            {{ process.status }}
                        </td>
                        <td scope="row" class=""
                            v-show="ioReadWritePresentProcesses && !getDisableStats().includes('io_counters')">
                            {{ $filters.bytes(process.io_read) }}
                        </td>
                        <td scope="row" class="text-start"
                            v-show="ioReadWritePresentProcesses && !getDisableStats().includes('io_counters')">
                            {{ $filters.bytes(process.io_write) }}
                        </td>
                        <td scope="row" class="text-truncate"
                            v-show="args.process_short_name && !getDisableStats().includes('cmdline')">
                            {{ process.name }}
                        </td>
                        <td scope="row" class="text-truncate"
                            v-show="!args.process_short_name && !getDisableStats().includes('cmdline')">
                            {{ process.cmdline }}
                        </td>
                    </tr>
                </tbody>
            </table>
        </div>
    </section>


    <!-- Display programs -->
    <section class="plugin" id="processlist" v-if="args.programs">
        <div class="table-responsive d-lg-none">
            <table class="table table-sm table-borderless table-striped table-hover">
                <thead>
                    <tr>
                        <td :class="['sortable', sorter.column === 'cpu_percent' && 'sort']"
                            @click="$emit('update:sorter', 'cpu_percent')"
                            v-show="!getDisableStats().includes('cpu_percent')">
                            CPU%
                        </td>
                        <td :class="['sortable', sorter.column === 'memory_percent' && 'sort']"
                            @click="$emit('update:sorter', 'memory_percent')"
                            v-show="!getDisableStats().includes('memory_percent')">
                            MEM%
                        </td>
                        <td v-show="!getDisableStats().includes('nprocs')">
                            NPROCS
                        </td>
                        <td scope="row" :class="['sortable', sorter.column === 'name' && 'sort']"
                            @click="$emit('update:sorter', 'name')" v-show="!getDisableStats().includes('cmdline')">
                            Command (click to pin)
                        </td>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="(process, processId) in programs" :key="processId">
                        <td scope="row" :class="getCpuPercentAlert(process)"
                            v-show="!getDisableStats().includes('cpu_percent')">
                            {{ process.cpu_percent == -1 ? '?' : $filters.number(process.cpu_percent, 1) }}
                        </td>
                        <td scope="row" :class="getMemoryPercentAlert(process)"
                            v-show="!getDisableStats().includes('memory_percent')">
                            {{ process.memory_percent == -1 ? '?' : $filters.number(process.memory_percent, 1) }}
                        </td>
                        <td scope="row" v-show="!getDisableStats().includes('nprocs')">
                            {{ process.nprocs }}
                        </td>
                        <td scope="row" class="text-truncate"
                            v-show="args.process_short_name && !getDisableStats().includes('cmdline')">
                            {{ process.name }}
                        </td>
                        <td scope="row" v-show="!args.process_short_name && !getDisableStats().includes('cmdline')">
                            {{ process.cmdline }}
                        </td>
                    </tr>
                </tbody>
            </table>
        </div>
        <div class="table-responsive d-none d-lg-block">
            <table class="table table-sm table-borderless table-striped table-hover">
                <thead>
                    <tr>
                        <td :class="['sortable', sorter.column === 'cpu_percent' && 'sort']"
                            @click="$emit('update:sorter', 'cpu_percent')"
                            v-show="!getDisableStats().includes('cpu_percent')">
                            CPU%
                        </td>
                        <td :class="['sortable', sorter.column === 'memory_percent' && 'sort']"
                            @click="$emit('update:sorter', 'memory_percent')"
                            v-show="!getDisableStats().includes('memory_percent')">
                            MEM%
                        </td>
                        <td class="" v-show="!getDisableStats().includes('memory_info')">
                            VIRT
                        </td>
                        <td class="" v-show="!getDisableStats().includes('memory_info')">
                            RES
                        </td>
                        <td v-show="!getDisableStats().includes('nprocs')">
                            NPROCS
                        </td>
                        <td scope="row" :class="['sortable', sorter.column === 'username' && 'sort']"
                            @click="$emit('update:sorter', 'username')"
                            v-show="!getDisableStats().includes('username')">
                            USER
                        </td>
                        <td scope="row" class="" :class="['sortable', sorter.column === 'timemillis' && 'sort']"
                            @click="$emit('update:sorter', 'timemillis')"
                            v-show="!getDisableStats().includes('cpu_times')">
                            TIME+
                        </td>
                        <td scope="row" class="" :class="['sortable', sorter.column === 'num_threads' && 'sort']"
                            @click="$emit('update:sorter', 'num_threads')"
                            v-show="!getDisableStats().includes('num_threads')">
                            THR
                        </td>
                        <td scope="row" v-show="!getDisableStats().includes('nice')">NI</td>
                        <td scope="row" class="table-cell widtd-60" v-show="!getDisableStats().includes('status')">S
                        </td>
                        <td scope="row" class="" :class="['sortable', sorter.column === 'io_counters' && 'sort']"
                            v-show="ioReadWritePresentPrograms && !getDisableStats().includes('io_counters')"
                            @click="$emit('update:sorter', 'io_counters')">
                            IORps
                        </td>
                        <td scope="row" class="text-start"
                            :class="['sortable', sorter.column === 'io_counters' && 'sort']"
                            v-show="ioReadWritePresentPrograms && !getDisableStats().includes('io_counters')"
                            @click="$emit('update:sorter', 'io_counters')">
                            IOWps
                        </td>
                        <td scope="row" :class="['sortable', sorter.column === 'name' && 'sort']"
                            @click="$emit('update:sorter', 'name')" v-show="!getDisableStats().includes('cmdline')">
                            Command (click to pin)
                        </td>
                    </tr>
                </thead>
                <tbody>
                    <tr v-for="(process, processId) in programs" :key="processId">
                        <td scope="row" :class="getCpuPercentAlert(process)"
                            v-show="!getDisableStats().includes('cpu_percent')">
                            {{ process.cpu_percent == -1 ? '?' : $filters.number(process.cpu_percent, 1) }}
                        </td>
                        <td scope="row" :class="getMemoryPercentAlert(process)"
                            v-show="!getDisableStats().includes('memory_percent')">
                            {{ process.memory_percent == -1 ? '?' : $filters.number(process.memory_percent, 1) }}
                        </td>
                        <td scope="row" v-show="!getDisableStats().includes('memory_info')">
                            {{ $filters.bytes(process.memvirt) }}
                        </td>
                        <td scope="row" v-show="!getDisableStats().includes('memory_info')">
                            {{ $filters.bytes(process.memres) }}
                        </td>
                        <td scope="row" v-show="!getDisableStats().includes('nprocs')">
                            {{ process.nprocs }}
                        </td>
                        <td scope="row" v-show="!getDisableStats().includes('username')">
                            {{ process.username }}
                        </td>
                        <td scope="row" class="" v-show="!getDisableStats().includes('cpu_times')">
                            {{ process.timeforhuman }}
                        </td>
                        <td scope="row" class="" v-show="!getDisableStats().includes('num_threads')">
                            {{ process.num_threads == -1 ? '?' : process.num_threads }}
                        </td>
                        <td scope="row" :class="{ nice: process.isNice }" v-show="!getDisableStats().includes('nice')">
                            {{ $filters.exclamation(process.nice) }}
                        </td>
                        <td scope="row" :class="{ status: process.status == 'R' }"
                            v-show="!getDisableStats().includes('status')">
                            {{ process.status }}
                        </td>
                        <td scope="row" class=""
                            v-show="ioReadWritePresentPrograms && !getDisableStats().includes('io_counters')">
                            {{ $filters.bytes(process.io_read) }}
                        </td>
                        <td scope="row" class="text-start"
                            v-show="ioReadWritePresentPrograms && !getDisableStats().includes('io_counters')">
                            {{ $filters.bytes(process.io_write) }}
                        </td>
                        <td scope="row" class="text-truncate"
                            v-show="args.process_short_name && !getDisableStats().includes('cmdline')">
                            {{ process.name }}
                        </td>
                        <td scope="row" v-show="!args.process_short_name && !getDisableStats().includes('cmdline')">
                            {{ process.cmdline }}
                        </td>
                    </tr>
                </tbody>
            </table>
        </div>
    </section>
</template>

<script>
import { orderBy, last } from 'lodash';
import { timemillis, timedelta, limitTo, number, dictToString } from '../filters.js';
import { GlancesHelper } from '../services.js';
import { store } from '../store.js';

export default {
    props: {
        data: {
            type: Object
        },
        sorter: {
            type: Object
        }
    },
    data() {
        return {
            store
        };
    },
    computed: {
        args() {
            return this.store.args || {};
        },
        config() {
            return this.store.config || {};
        },
        stats_processlist() {
            return this.data.stats['processlist'];
        },
        extended_stats() {
            return this.stats_processlist.find(item => item['extended_stats'] === true) || null;
        },
        processes() {
            const { sorter } = this;
            const processes = (this.stats_processlist || []).map((process) => {
                return this.updateProcess(process, this.data.stats['isWindows']);
            });

            return orderBy(
                processes,
                [sorter.column].reduce((retval, col) => {
                    if (col === 'io_counters') {
                        col = ['io_read', 'io_write']
                    }
                    return retval.concat(col);
                }, []),
                [sorter.isReverseColumn(sorter.column) ? 'desc' : 'asc']
            ).slice(0, this.limit);
        },
        ioReadWritePresentProcesses() {
            return (this.stats_processlist || []).some(({ io_counters }) => io_counters);
        },
        stats_programlist() {
            return this.data.stats['programlist'];
        },
        programs() {
            const { sorter } = this;
            const isWindows = this.data.stats['isWindows'];
            const programs = (this.stats_programlist || []).map((process) => {
                process.memvirt = '?';
                process.memres = '?';
                if (process.memory_info) {
                    process.memvirt = process.memory_info.vms;
                    process.memres = process.memory_info.rss;
                }

                if (isWindows && process.username !== null) {
                    process.username = last(process.username.split('\\'));
                }

                process.timeforhuman = '?';
                if (process.cpu_times) {
                    process.timeplus = timedelta([process.cpu_times['user'], process.cpu_times['system']]);
                    process.timeforhuman = process.timeplus.hours.toString().padStart(2, '0') + ':' +
                        process.timeplus.minutes.toString().padStart(2, '0') + ':' +
                        process.timeplus.seconds.toString().padStart(2, '0')
                }

                if (process.num_threads === null) {
                    process.num_threads = -1;
                }

                if (process.cpu_percent === null) {
                    process.cpu_percent = -1;
                }

                if (process.memory_percent === null) {
                    process.memory_percent = -1;
                }

                process.io_read = null;
                process.io_write = null;

                if (process.io_counters) {
                    process.io_read =
                        (process.io_counters[0] - process.io_counters[2]) /
                        process.time_since_update;
                    process.io_write =
                        (process.io_counters[1] - process.io_counters[3]) /
                        process.time_since_update;
                }

                process.isNice =
                    process.nice !== undefined &&
                    ((isWindows && process.nice != 32) || (!isWindows && process.nice != 0));

                if (Array.isArray(process.cmdline)) {
                    process.cmdline = process.cmdline.join(' ').replace(/\n/g, ' ');
                }

                if (process.cmdline === null || process.cmdline.length === 0) {
                    process.cmdline = process.name;
                }

                return process;
            });

            return orderBy(
                programs,
                [sorter.column].reduce((retval, col) => {
                    if (col === 'io_counters') {
                        col = ['io_read', 'io_write']
                    }
                    return retval.concat(col);
                }, []),
                [sorter.isReverseColumn(sorter.column) ? 'desc' : 'asc']
            ).slice(0, this.limit);
        },
        ioReadWritePresentPrograms() {
            return (this.stats_programlist || []).some(({ io_counters }) => io_counters);
        },
        limit() {
            return this.config.outputs !== undefined
                ? this.config.outputs.max_processes_display
                : undefined;
        }
    },
    methods: {
        updateProcess(process, isWindows) {
            process.memvirt = '?';
            process.memres = '?';
            if (process.memory_info) {
                process.memvirt = process.memory_info.vms;
                process.memres = process.memory_info.rss;
            }

            if (isWindows && process.username !== null) {
                process.username = last(process.username.split('\\'));
            }

            process.timeforhuman = '?';
            if (process.cpu_times) {
                process.timeplus = timedelta([process.cpu_times['user'], process.cpu_times['system']]);
                process.timeforhuman = process.timeplus.hours.toString().padStart(2, '0') + ':' +
                    process.timeplus.minutes.toString().padStart(2, '0') + ':' +
                    process.timeplus.seconds.toString().padStart(2, '0')
            }

            if (process.num_threads === null) {
                process.num_threads = -1;
            }

            if (process.cpu_percent === null) {
                process.cpu_percent = -1;
            }

            if (process.memory_percent === null) {
                process.memory_percent = -1;
            }

            process.io_read = null;
            process.io_write = null;

            if (process.io_counters) {
                process.io_read =
                    (process.io_counters[0] - process.io_counters[2]) /
                    process.time_since_update;
                process.io_write =
                    (process.io_counters[1] - process.io_counters[3]) /
                    process.time_since_update;
            }

            process.isNice =
                process.nice !== undefined &&
                ((isWindows && process.nice != 32) || (!isWindows && process.nice != 0));

            if (Array.isArray(process.cmdline)) {
                process.name = process.name + ' ' + process.cmdline.slice(1).join(' ').replace(/\n/g, ' ');
                process.cmdline = process.cmdline.join(' ').replace(/\n/g, ' ');
            }

            if (process.cmdline === null || process.cmdline.length === 0) {
                process.cmdline = process.name;
            }
            return process
        },
        getCpuPercentAlert(process) {
            return GlancesHelper.getAlert('processlist', 'processlist_cpu_', process.cpu_percent);
        },
        getMemoryPercentAlert(process) {
            return GlancesHelper.getAlert('processlist', 'processlist_mem_', process.cpu_percent);
        },
        getDisableStats() {
            return GlancesHelper.getLimit('processlist', 'processlist_disable_stats') || [];
        },
        setExtendedStats(pid) {
            fetch('api/4/processes/extended/' + pid.toString(), { method: 'POST' })
                .then((response) => response.json());
            this.$forceUpdate()
        },
        disableExtendedStats() {
            fetch('api/4/processes/extended/disable', { method: 'POST' })
                .then((response) => response.json());
            this.$forceUpdate()
        }
    }
};
</script>