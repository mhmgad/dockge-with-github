<template>
    <div class="item-wrapper">
        <router-link :to="url" :class="{ 'dim' : !stack.isManagedByDockge }" class="item">
            <Uptime :stack="stack" :fixed-width="true" class="me-2" />
            <div class="title">
                <span>{{ stackName }}</span>
                <div v-if="stack.repo && stack.repo !== 'Default'" class="repo-label">{{ stack.repo }}</div>
                <div v-if="$root.agentCount > 1" class="endpoint">{{ endpointDisplay }}</div>
            </div>
        </router-link>
        <span v-if="isGitRepo && lastSyncTime" class="stack-last-sync">
            <font-awesome-icon icon="clock" class="me-1" />
            {{ formatSyncTime(lastSyncTime) }}
        </span>
        <button
            v-if="isGitRepo"
            class="btn-git-sync-stack"
            :title="$t('gitSync')"
            @click.stop.prevent="$emit('open-git-sync', stack)"
        >
            <font-awesome-icon icon="sync" />
        </button>
    </div>
</template>

<script>
import Uptime from "./Uptime.vue";
import { FontAwesomeIcon } from "@fortawesome/vue-fontawesome";

export default {
    components: {
        Uptime,
        FontAwesomeIcon,
    },
    emits: [ "open-git-sync" ],
    props: {
        /** Stack this represents */
        stack: {
            type: Object,
            default: null,
        },
        /** If the user is in select mode */
        isSelectMode: {
            type: Boolean,
            default: false,
        },
        /** How many ancestors are above this stack */
        depth: {
            type: Number,
            default: 0,
        },
        /** Callback to determine if stack is selected */
        isSelected: {
            type: Function,
            default: () => {}
        },
        /** Callback fired when stack is selected */
        select: {
            type: Function,
            default: () => {}
        },
        /** Callback fired when stack is deselected */
        deselect: {
            type: Function,
            default: () => {}
        },
        /** Whether this stack is a git repository */
        isGitRepo: {
            type: Boolean,
            default: false,
        },
        /** Last sync time for git repo stacks */
        lastSyncTime: {
            type: String,
            default: "",
        },
    },
    data() {
        return {
            isCollapsed: true,
        };
    },
    computed: {
        endpointDisplay() {
            return this.$root.endpointDisplayFunction(this.stack.endpoint);
        },
        url() {
            const encodedStackName = encodeURIComponent(this.stack.name);
            if (this.stack.endpoint) {
                return `/compose/${encodedStackName}/${this.stack.endpoint}`;
            } else {
                return `/compose/${encodedStackName}`;
            }
        },
        depthMargin() {
            return {
                marginLeft: `${31 * this.depth}px`,
            };
        },
        stackName() {
            // If stack is in a repo group (not "Default"), show only the stack name without parent directory
            if (this.stack.repo && this.stack.repo !== "Default") {
                // Remove the repo prefix from the stack name
                const repoPrefix = this.stack.repo + "/";
                if (this.stack.name.startsWith(repoPrefix)) {
                    return this.stack.name.substring(repoPrefix.length);
                }
            }
            return this.stack.name;
        }
    },
    watch: {
        isSelectMode() {
            // TODO: Resize the heartbeat bar, but too slow
            // this.$refs.heartbeatBar.resize();
        }
    },
    beforeMount() {

    },
    methods: {
        /**
         * Changes the collapsed value of the current stack and saves
         * it to local storage
         * @returns {void}
         */
        changeCollapsed() {
            this.isCollapsed = !this.isCollapsed;

            // Save collapsed value into local storage
            let storage = window.localStorage.getItem("stackCollapsed");
            let storageObject = {};
            if (storage !== null) {
                storageObject = JSON.parse(storage);
            }
            storageObject[`stack_${this.stack.id}`] = this.isCollapsed;

            window.localStorage.setItem("stackCollapsed", JSON.stringify(storageObject));
        },

        /**
         * Toggle selection of stack
         * @returns {void}
         */
        toggleSelection() {
            if (this.isSelected(this.stack.id)) {
                this.deselect(this.stack.id);
            } else {
                this.select(this.stack.id);
            }
        },

        /**
         * Format the sync time for display
         * @param {string} timestamp - ISO timestamp
         * @returns {string} Formatted time
         */
        formatSyncTime(timestamp) {
            if (!timestamp) {
                return "";
            }

            const MINUTE_MS = 60000;
            const HOUR_MS = 3600000;
            const DAY_MS = 86400000;

            const date = new Date(timestamp);
            const now = new Date();
            const diffMs = now - date;
            const diffMins = Math.floor(diffMs / MINUTE_MS);
            const diffHours = Math.floor(diffMs / HOUR_MS);
            const diffDays = Math.floor(diffMs / DAY_MS);

            if (diffMins < 1) {
                return this.$t("justNow") || "Just now";
            } else if (diffMins < 60) {
                return `${diffMins}m`;
            } else if (diffHours < 24) {
                return `${diffHours}h`;
            } else if (diffDays < 7) {
                return `${diffDays}d`;
            } else {
                return date.toLocaleDateString();
            }
        },
    },
};
</script>

<style lang="scss" scoped>
@import "../styles/vars.scss";

.small-padding {
    padding-left: 5px !important;
    padding-right: 5px !important;
}

.collapse-padding {
    padding-left: 8px !important;
    padding-right: 2px !important;
}

.item {
    text-decoration: none;
    display: flex;
    align-items: center;
    min-height: 52px;
    border-radius: 10px;
    transition: all ease-in-out 0.15s;
    width: 100%;
    padding: 5px 8px;
    &.disabled {
        opacity: 0.3;
    }
    &:hover {
        background-color: $highlight-white;
    }
    &.active {
        background-color: #cdf8f4;
    }
    .title {
        margin-top: -4px;
    }
    .repo-label {
        font-size: 12px;
        color: $dark-font-color3;
        font-weight: 500;
    }
    .endpoint {
        font-size: 12px;
        color: $dark-font-color3;
    }
}

.collapsed {
    transform: rotate(-90deg);
}

.animated {
    transition: all 0.2s $easing-in;
}

.select-input-wrapper {
    float: left;
    margin-top: 15px;
    margin-left: 3px;
    margin-right: 10px;
    padding-left: 4px;
    position: relative;
    z-index: 15;
}

.dim {
    opacity: 0.5;
}

.item-wrapper {
    display: flex;
    align-items: center;
    width: 100%;
}

.item-wrapper .item {
    flex: 1;
}

.stack-last-sync {
    font-size: 11px;
    color: $dark-font-color3;
    margin-right: 8px;
    white-space: nowrap;
    flex-shrink: 0;
}

.btn-git-sync-stack {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 28px;
    height: 28px;
    padding: 0;
    font-size: 12px;
    color: $primary;
    background-color: transparent;
    border: 1px solid $primary;
    border-radius: 50%;
    cursor: pointer;
    transition: all ease-in-out 0.15s;
    margin-right: 8px;
    flex-shrink: 0;

    &:hover {
        background-color: $primary;
        color: #fff;
    }

    .dark & {
        color: $primary;
        border-color: $primary;

        &:hover {
            background-color: $primary;
            color: $dark-font-color2;
        }
    }
}

</style>
