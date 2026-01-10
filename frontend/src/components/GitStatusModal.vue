<template>
    <BModal v-model="show" size="lg" :title="$t('gitSync')" @hide="onHide">
        <div v-if="loading" class="text-center">
            <div class="spinner-border" role="status">
                <span class="visually-hidden">Loading...</span>
            </div>
        </div>

        <div v-else-if="!isGitRepo" class="alert alert-warning">
            {{ $t('notGitRepository') }}
        </div>

        <div v-else>
            <!-- Git Status Info -->
            <div class="mb-3">
                <strong>{{ $t('branch') }}:</strong> {{ gitStatus.current }}
                <span v-if="gitStatus.tracking"> → {{ gitStatus.tracking }}</span>
            </div>

            <div v-if="gitStatus.ahead > 0 || gitStatus.behind > 0" class="mb-3">
                <span v-if="gitStatus.ahead > 0" class="badge bg-info me-2">
                    ↑ {{ gitStatus.ahead }} {{ $t('commitsAhead') }}
                </span>
                <span v-if="gitStatus.behind > 0" class="badge bg-warning">
                    ↓ {{ gitStatus.behind }} {{ $t('commitsBehind') }}
                </span>
            </div>

            <!-- Tabs for Local and Remote -->
            <ul class="nav nav-tabs mb-3">
                <li class="nav-item">
                    <a
                        class="nav-link"
                        :class="{ active: activeTab === 'local' }"
                        href="#"
                        @click.prevent="activeTab = 'local'"
                    >
                        <font-awesome-icon icon="file-alt" class="me-1" />
                        {{ $t('localChanges') }}
                        <span v-if="gitStatus.files.length > 0" class="badge bg-secondary ms-1">
                            {{ gitStatus.files.length }}
                        </span>
                    </a>
                </li>
                <li class="nav-item">
                    <a
                        class="nav-link"
                        :class="{ active: activeTab === 'remote' }"
                        href="#"
                        @click.prevent="activeTab = 'remote'"
                    >
                        <font-awesome-icon icon="cloud" class="me-1" />
                        {{ $t('remoteDiff') }}
                        <span v-if="totalRemoteChanges > 0" class="badge bg-secondary ms-1">
                            {{ totalRemoteChanges }}
                        </span>
                    </a>
                </li>
            </ul>

            <!-- Local Changes Tab -->
            <div v-if="activeTab === 'local'">
                <!-- Unstaged Files List -->
                <div v-if="unstagedFiles.length > 0" class="mb-3">
                    <h6>{{ $t('unstagedChanges') }}:</h6>
                    <div class="file-list">
                        <div v-for="file in unstagedFiles" :key="file.path" class="file-item d-flex align-items-center">
                            <input
                                :id="'file-' + file.path"
                                v-model="selectedFiles"
                                type="checkbox"
                                :value="file.path"
                                class="form-check-input me-2"
                            />
                            <span :class="getFileStatusClass(file.status)" class="me-2 status-badge">
                                {{ getFileStatusLabel(file.status) }}
                            </span>
                            <span class="file-path">{{ file.path }}</span>
                        </div>
                    </div>
                </div>

                <!-- Staged Files List -->
                <div v-if="stagedFiles.length > 0" class="mb-3">
                    <h6>{{ $t('stagedChanges') }}:</h6>
                    <div class="file-list staged-list">
                        <div v-for="file in stagedFiles" :key="file.path" class="file-item d-flex align-items-center">
                            <input
                                :id="'staged-file-' + file.path"
                                v-model="selectedStagedFiles"
                                type="checkbox"
                                :value="file.path"
                                class="form-check-input me-2"
                            />
                            <span :class="getFileStatusClass(file.status)" class="me-2 status-badge">
                                {{ getFileStatusLabel(file.status) }}
                            </span>
                            <span class="file-path">{{ file.path }}</span>
                        </div>
                    </div>
                </div>

                <div v-if="gitStatus.files.length === 0" class="alert alert-info">
                    {{ $t('noChanges') }}
                </div>

                <!-- Commit Message -->
                <div v-if="stagedFiles.length > 0" class="mb-3">
                    <label for="commitMessage" class="form-label">{{ $t('commitMessage') }}</label>
                    <input
                        id="commitMessage"
                        v-model="commitMessage"
                        type="text"
                        class="form-control"
                        :placeholder="$t('commitMessagePlaceholder')"
                    />
                </div>
            </div>

            <!-- Remote Diff Tab -->
            <div v-if="activeTab === 'remote'">
                <div class="mb-3">
                    <button
                        class="btn btn-sm btn-outline-secondary"
                        :disabled="processing"
                        @click="fetchRemote"
                    >
                        <font-awesome-icon icon="sync" class="me-1" :spin="fetching" />
                        {{ $t('fetchRemote') }}
                    </button>
                </div>

                <!-- Incoming Commits (commits to pull) -->
                <div v-if="gitStatus.incomingCommits && gitStatus.incomingCommits.length > 0" class="mb-3">
                    <h6>
                        <font-awesome-icon icon="download" class="me-1 text-warning" />
                        {{ $t('incomingCommits') }} ({{ gitStatus.incomingCommits.length }})
                    </h6>
                    <div class="commit-list">
                        <div v-for="commit in gitStatus.incomingCommits" :key="commit.hash" class="commit-item">
                            <span class="commit-hash">{{ commit.hash }}</span>
                            <span class="commit-message">{{ commit.message }}</span>
                            <span class="commit-author text-muted">{{ commit.author }}</span>
                        </div>
                    </div>
                </div>

                <!-- Outgoing Commits (commits to push) -->
                <div v-if="gitStatus.outgoingCommits && gitStatus.outgoingCommits.length > 0" class="mb-3">
                    <h6>
                        <font-awesome-icon icon="upload" class="me-1 text-info" />
                        {{ $t('outgoingCommits') }} ({{ gitStatus.outgoingCommits.length }})
                    </h6>
                    <div class="commit-list">
                        <div v-for="commit in gitStatus.outgoingCommits" :key="commit.hash" class="commit-item">
                            <span class="commit-hash">{{ commit.hash }}</span>
                            <span class="commit-message">{{ commit.message }}</span>
                            <span class="commit-author text-muted">{{ commit.author }}</span>
                        </div>
                    </div>
                </div>

                <div v-if="(!gitStatus.incomingCommits || gitStatus.incomingCommits.length === 0) && (!gitStatus.outgoingCommits || gitStatus.outgoingCommits.length === 0)" class="alert alert-info">
                    {{ $t('noRemoteChanges') }}
                </div>
            </div>

            <!-- Credentials Prompt -->
            <div v-if="needsCredentials && !showCredentialsForm" class="mb-3">
                <div class="alert alert-warning d-flex align-items-center justify-content-between">
                    <span>
                        <font-awesome-icon icon="exclamation-triangle" class="me-2" />
                        {{ $t('gitCredentialsRequired') }}
                    </span>
                    <button class="btn btn-sm btn-outline-primary" @click="showCredentialsForm = true">
                        <font-awesome-icon icon="key" class="me-1" />
                        {{ $t('addCredentials') }}
                    </button>
                </div>
            </div>

            <!-- Credentials Form -->
            <div v-if="showCredentialsForm" class="mb-3 credentials-form">
                <div class="card">
                    <div class="card-header d-flex justify-content-between align-items-center">
                        <span>
                            <font-awesome-icon icon="key" class="me-2" />
                            {{ $t('gitCredentials') }}
                        </span>
                        <button class="btn btn-sm btn-close" @click="showCredentialsForm = false"></button>
                    </div>
                    <div class="card-body">
                        <div class="mb-2">
                            <label for="gitUsername" class="form-label">{{ $t('username') }}</label>
                            <input
                                id="gitUsername"
                                v-model="credentials.username"
                                type="text"
                                class="form-control"
                                :placeholder="$t('githubUsername')"
                            />
                        </div>
                        <div class="mb-2">
                            <label for="gitPassword" class="form-label">{{ $t('Password') }} / {{ $t('token') }}</label>
                            <input
                                id="gitPassword"
                                v-model="credentials.password"
                                type="password"
                                class="form-control"
                                :placeholder="$t('githubPasswordOrToken')"
                            />
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <template #footer>
            <button class="btn btn-secondary" @click="onHide">
                {{ $t('close') }}
            </button>
            <button
                v-if="isGitRepo && activeTab === 'local' && selectedFiles.length > 0"
                class="btn btn-primary"
                :disabled="processing"
                @click="addFiles"
            >
                <font-awesome-icon icon="plus" class="me-1" />
                {{ $t('addFiles') }}
            </button>
            <button
                v-if="isGitRepo && activeTab === 'local' && selectedStagedFiles.length > 0"
                class="btn btn-warning"
                :disabled="processing"
                @click="unstageFiles"
            >
                <font-awesome-icon icon="minus" class="me-1" />
                {{ $t('unstageFiles') }}
            </button>
            <button
                v-if="isGitRepo && activeTab === 'local' && gitStatus.files.some(f => f.staged)"
                class="btn btn-success"
                :disabled="processing || !commitMessage"
                @click="commitAndPush"
            >
                <font-awesome-icon icon="upload" class="me-1" />
                {{ $t('commitAndPush') }}
            </button>
            <button
                v-if="isGitRepo && gitStatus.behind > 0"
                class="btn-pull"
                :disabled="processing"
                @click="pullChanges"
            >
                <font-awesome-icon icon="download" class="me-1" />
                {{ $t('pull') }}
            </button>
            <button
                v-if="isGitRepo && hasOutgoingCommits"
                class="btn-push"
                :disabled="processing"
                @click="pushChanges"
            >
                <font-awesome-icon icon="upload" class="me-1" />
                {{ $t('push') }}
            </button>
        </template>
    </BModal>
</template>

<script>
import { BModal } from "bootstrap-vue-next";
import { FontAwesomeIcon } from "@fortawesome/vue-fontawesome";

export default {
    components: {
        BModal,
        FontAwesomeIcon,
    },
    props: {
        repoName: {
            type: String,
            required: true,
        },
        endpoint: {
            type: String,
            default: "",
        },
        stackName: {
            type: String,
            default: "",
        },
    },
    data() {
        return {
            show: false,
            loading: false,
            processing: false,
            fetching: false,
            isGitRepo: true,
            activeTab: "local",
            gitStatus: {
                files: [],
                current: "",
                tracking: null,
                ahead: 0,
                behind: 0,
                incomingCommits: [],
                outgoingCommits: [],
            },
            selectedFiles: [],
            selectedStagedFiles: [],
            commitMessage: "",
            needsCredentials: false,
            showCredentialsForm: false,
            credentials: {
                username: "",
                password: "",
            },
        };
    },
    computed: {
        unstagedFiles() {
            return this.gitStatus.files.filter(f => !f.staged);
        },
        stagedFiles() {
            return this.gitStatus.files.filter(f => f.staged);
        },
        totalRemoteChanges() {
            const incoming = this.gitStatus.incomingCommits?.length || 0;
            const outgoing = this.gitStatus.outgoingCommits?.length || 0;
            return incoming + outgoing;
        },
        // For Default stacks, use stack-based API; otherwise use repo-based API
        isDefaultStack() {
            return this.repoName === "Default" && this.stackName;
        },
        // The identifier to use for git operations
        gitIdentifier() {
            return this.isDefaultStack ? this.stackName : this.repoName;
        },
        // Check if there are outgoing commits to push
        hasOutgoingCommits() {
            return (this.gitStatus.outgoingCommits && this.gitStatus.outgoingCommits.length > 0) || this.gitStatus.ahead > 0;
        },
    },
    methods: {
        async open() {
            this.show = true;
            await this.loadGitStatus();
            await this.checkCredentials();
        },

        onHide() {
            this.show = false;
            this.selectedFiles = [];
            this.selectedStagedFiles = [];
            this.commitMessage = "";
            this.needsCredentials = false;
            this.showCredentialsForm = false;
            this.credentials = { username: "",
                password: "" };
            this.activeTab = "local";
        },

        async loadGitStatus() {
            this.loading = true;
            const event = this.isDefaultStack ? "getStackGitStatus" : "getRepoGitStatus";
            this.$root.emitAgent(this.endpoint, event, this.gitIdentifier, (res) => {
                this.loading = false;
                if (res.ok) {
                    if (res.gitStatus) {
                        this.gitStatus = res.gitStatus;
                        this.isGitRepo = true;
                    } else {
                        this.isGitRepo = false;
                    }
                } else {
                    this.isGitRepo = false;
                    this.$root.toastError(res.msg || "Failed to get git status");
                }
            });
        },

        async fetchRemote() {
            this.fetching = true;
            const creds = this.credentials.username && this.credentials.password
                ? this.credentials
                : null;

            const event = this.isDefaultStack ? "gitFetchStack" : "gitFetch";
            this.$root.emitAgent(this.endpoint, event, this.gitIdentifier, creds, (res) => {
                this.fetching = false;
                if (res.ok) {
                    this.loadGitStatus();
                } else {
                    this.$root.toastError(res.msg || "Failed to fetch from remote");
                }
            });
        },

        async checkCredentials() {
            this.$root.emitAgent(this.endpoint, "getGitCredentials", (res) => {
                if (res.ok && !res.hasCredentials) {
                    this.needsCredentials = true;
                }
            });
        },

        async addFiles() {
            if (this.selectedFiles.length === 0) {
                return;
            }

            this.processing = true;
            const event = this.isDefaultStack ? "gitAddFiles" : "gitAddFilesRepo";
            this.$root.emitAgent(this.endpoint, event, this.gitIdentifier, this.selectedFiles, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
                if (res.ok) {
                    this.selectedFiles = [];
                    this.loadGitStatus();
                }
            });
        },

        async unstageFiles() {
            if (this.selectedStagedFiles.length === 0) {
                return;
            }

            this.processing = true;
            const event = this.isDefaultStack ? "gitUnstageFiles" : "gitUnstageFilesRepo";
            this.$root.emitAgent(this.endpoint, event, this.gitIdentifier, this.selectedStagedFiles, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
                if (res.ok) {
                    this.selectedStagedFiles = [];
                    this.loadGitStatus();
                }
            });
        },

        async commitAndPush() {
            if (!this.commitMessage) {
                this.$root.toastError(this.$t("pleaseEnterCommitMessage"));
                return;
            }

            this.processing = true;

            // First commit
            const commitEvent = this.isDefaultStack ? "gitCommit" : "gitCommitRepo";
            this.$root.emitAgent(this.endpoint, commitEvent, this.gitIdentifier, this.commitMessage, (res) => {
                if (res.ok) {
                    // Then push
                    const creds = this.credentials.username && this.credentials.password
                        ? this.credentials
                        : null;

                    const pushEvent = this.isDefaultStack ? "gitPush" : "gitPushRepo";
                    this.$root.emitAgent(this.endpoint, pushEvent, this.gitIdentifier, creds, (pushRes) => {
                        this.processing = false;
                        this.$root.toastRes(pushRes);
                        if (pushRes.ok) {
                            this.commitMessage = "";
                            this.needsCredentials = false;
                            this.showCredentialsForm = false;
                            this.loadGitStatus();
                        }
                    });
                } else {
                    this.processing = false;
                    this.$root.toastRes(res);
                }
            });
        },

        async pullChanges() {
            this.processing = true;

            const creds = this.credentials.username && this.credentials.password
                ? this.credentials
                : null;

            const event = this.isDefaultStack ? "gitPull" : "gitPullRepo";
            this.$root.emitAgent(this.endpoint, event, this.gitIdentifier, creds, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
                if (res.ok) {
                    this.needsCredentials = false;
                    this.showCredentialsForm = false;
                    this.loadGitStatus();
                }
            });
        },

        async pushChanges() {
            this.processing = true;

            const creds = this.credentials.username && this.credentials.password
                ? this.credentials
                : null;

            const pushEvent = this.isDefaultStack ? "gitPush" : "gitPushRepo";
            this.$root.emitAgent(this.endpoint, pushEvent, this.gitIdentifier, creds, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
                if (res.ok) {
                    this.needsCredentials = false;
                    this.showCredentialsForm = false;
                    this.loadGitStatus();
                }
            });
        },

        getFileStatusClass(status) {
            const statusClasses = {
                "M": "badge status-modified",
                "A": "badge status-added",
                "D": "badge status-deleted",
                "R": "badge status-renamed",
                "?": "badge status-untracked",
            };
            return statusClasses[status] || "badge status-untracked";
        },

        getFileStatusLabel(status) {
            const statusLabels = {
                "M": "Modified",
                "A": "Added",
                "D": "Deleted",
                "R": "Renamed",
                "?": "Untracked",
            };
            return statusLabels[status] || status;
        },
    },
};
</script>

<style scoped lang="scss">
@import "../styles/vars.scss";

.file-list {
    max-height: 200px;
    overflow-y: auto;
    border: 1px solid $dark-border-color;
    border-radius: 10px;
    padding: 0.5rem;
    background-color: transparent;

    .dark & {
        border-color: $dark-border-color;
    }
}

.file-list.staged-list {
    border-color: $primary;
    border-width: 2px;
    background-color: rgba($primary, 0.05);
}

.file-item {
    padding: 0.5rem 0.25rem;
    border-bottom: 1px solid rgba($dark-border-color, 0.5);
}

.file-item:last-child {
    border-bottom: none;
}

.file-path {
    font-family: monospace;
    font-size: 0.9rem;
    color: inherit;
    word-break: break-all;
}

.status-badge {
    min-width: 70px;
    text-align: center;
    font-size: 0.75rem;
    border-radius: 25px;
}

/* Status colors using site color scheme */
.status-modified {
    background-color: $warning !important;
    color: #fff !important;
}

.status-added {
    background-color: #4caf50 !important;
    color: #fff !important;
}

.status-deleted {
    background-color: $danger !important;
    color: #fff !important;
}

.status-renamed {
    background-color: $primary !important;
    color: $dark-font-color2 !important;
}

.status-untracked {
    background-color: $dark-font-color3 !important;
    color: #fff !important;
}

.commit-list {
    max-height: 200px;
    overflow-y: auto;
    border: 1px solid $dark-border-color;
    border-radius: 10px;
    padding: 0.5rem;
    background-color: transparent;

    .dark & {
        border-color: $dark-border-color;
    }
}

.commit-item {
    padding: 0.5rem;
    border-bottom: 1px solid rgba($dark-border-color, 0.5);
    display: flex;
    flex-wrap: wrap;
    gap: 0.5rem;
}

.commit-item:last-child {
    border-bottom: none;
}

.commit-hash {
    font-family: monospace;
    font-size: 0.85rem;
    background-color: rgba($primary, 0.2);
    color: inherit;
    padding: 0.1rem 0.4rem;
    border-radius: 4px;
}

.commit-message {
    flex: 1;
    min-width: 200px;
    color: inherit;
}

.commit-author {
    font-size: 0.85rem;
    color: $dark-font-color3;
}

.nav-tabs {
    border-bottom-color: $dark-border-color;

    .nav-link {
        cursor: pointer;
        border-radius: 10px 10px 0 0;
        color: inherit;
        border: 1px solid transparent;
        transition: all ease-in-out 0.15s;

        &:hover {
            border-color: transparent;
            background-color: rgba($primary, 0.1);
        }

        &.active {
            font-weight: 600;
            background-color: transparent;
            border-color: $dark-border-color $dark-border-color transparent;
            color: $primary;

            .dark & {
                background-color: $dark-bg;
                border-color: $dark-border-color $dark-border-color $dark-bg;
            }
        }
    }
}

.credentials-form .card {
    border-color: $primary;
    border-radius: 10px;
    overflow: hidden;
}

.credentials-form .card-header {
    background-color: rgba($primary, 0.1);
    border-bottom-color: $primary;
    color: inherit;
}

.dark .credentials-form .card-header {
    background-color: rgba($primary, 0.15);
}

/* Pull button styling */
.btn-pull {
    display: inline-flex;
    align-items: center;
    padding: 6px 16px;
    font-size: 14px;
    font-weight: 500;
    color: $warning;
    background-color: transparent;
    border: 1px solid $warning;
    border-radius: 25px;
    cursor: pointer;
    transition: all ease-in-out 0.15s;

    &:hover {
        background-color: $warning;
        color: #fff;
    }

    &:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }

    .dark & {
        color: $warning;
        border-color: $warning;

        &:hover {
            background-color: $warning;
            color: #fff;
        }
    }
}

/* Push button styling */
.btn-push {
    display: inline-flex;
    align-items: center;
    padding: 6px 16px;
    font-size: 14px;
    font-weight: 500;
    color: $primary;
    background-color: transparent;
    border: 1px solid $primary;
    border-radius: 25px;
    cursor: pointer;
    transition: all ease-in-out 0.15s;

    &:hover {
        background-color: $primary;
        color: #fff;
    }

    &:disabled {
        opacity: 0.5;
        cursor: not-allowed;
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
