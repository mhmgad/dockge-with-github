<template>
    <BModal v-model="show" size="lg" :title="$t('gitStatus')" @hide="onHide">
        <div v-if="loading" class="text-center">
            <div class="spinner-border" role="status">
                <span class="visually-hidden">Loading...</span>
            </div>
        </div>

        <div v-else-if="!isGitRepo" class="alert alert-warning">
            {{ $t('notGitRepository') }}
        </div>

        <!-- Sync Preview Mode -->
        <div v-else-if="showSyncPreview">
            <h5>{{ $t('syncPreview') || 'Sync Preview' }}</h5>

            <!-- Credentials Dialog (if needed) -->
            <div v-if="syncPreview.needsCredentials && showCredentialsDialog" class="mb-3">
                <div class="alert alert-info">
                    {{ $t('gitCredentialsRequired') }}
                </div>
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
                    <label for="gitPassword" class="form-label">{{ $t('passwordOrToken') || 'Password / Token' }}</label>
                    <input
                        id="gitPassword"
                        v-model="credentials.password"
                        type="password"
                        class="form-control"
                        :placeholder="$t('githubPasswordOrToken')"
                    />
                </div>
            </div>

            <!-- Local Changes to Push -->
            <div v-if="syncPreview.hasLocalChanges" class="mb-3">
                <h6 class="text-success">{{ $t('localChangesToPush') || 'Local Changes to Push' }}:</h6>
                <div v-if="syncPreview.ahead > 0" class="mb-2">
                    <span class="badge bg-info">
                        ↑ {{ syncPreview.ahead }} {{ $t('commitsAhead') || 'commits ahead' }}
                    </span>
                </div>
                <div v-if="syncPreview.localChanges.length > 0" class="file-list">
                    <div v-for="file in syncPreview.localChanges" :key="file.path" class="file-item d-flex align-items-center mb-2">
                        <span :class="getFileStatusClass(file.status)" class="me-2">
                            {{ file.status }}
                        </span>
                        <span class="text-monospace">{{ file.path }}</span>
                    </div>
                </div>
            </div>

            <!-- Remote Changes to Pull -->
            <div v-if="syncPreview.hasRemoteChanges" class="mb-3">
                <h6 class="text-primary">{{ $t('remoteChangesToPull') || 'Remote Changes to Pull' }}:</h6>
                <div class="mb-2">
                    <span class="badge bg-warning">
                        ↓ {{ syncPreview.behind }} {{ $t('commitsBehind') || 'commits behind' }}
                    </span>
                </div>
                <div v-if="syncPreview.remoteCommits.length > 0" class="commit-list">
                    <div v-for="commit in syncPreview.remoteCommits" :key="commit.hash" class="commit-item mb-2">
                        <div class="d-flex">
                            <span class="badge bg-secondary me-2">{{ commit.hash }}</span>
                            <span class="flex-grow-1">{{ commit.message }}</span>
                        </div>
                        <div class="commit-meta">
                            <small class="text-muted">{{ commit.author }} - {{ formatDate(commit.date) }}</small>
                        </div>
                    </div>
                </div>
            </div>

            <div v-if="!syncPreview.hasLocalChanges && !syncPreview.hasRemoteChanges" class="alert alert-success">
                {{ $t('alreadyInSync') || 'Everything is already in sync!' }}
            </div>
        </div>

        <!-- Normal Git Status Mode -->
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

            <!-- Unstaged Files List -->
            <div v-if="unstagedFiles.length > 0" class="mb-3">
                <h6>{{ $t('unstagedChanges') }}:</h6>
                <div class="file-list">
                    <div v-for="file in unstagedFiles" :key="file.path" class="file-item d-flex align-items-center mb-2">
                        <input
                            :id="'file-' + file.path"
                            v-model="selectedFiles"
                            type="checkbox"
                            :value="file.path"
                            class="form-check-input me-2"
                        />
                        <span :class="getFileStatusClass(file.status)" class="me-2">
                            {{ file.status }}
                        </span>
                        <span class="text-monospace">{{ file.path }}</span>
                    </div>
                </div>
            </div>

            <!-- Staged Files List -->
            <div v-if="stagedFiles.length > 0" class="mb-3">
                <h6>{{ $t('stagedChanges') }}:</h6>
                <div class="file-list staged-list">
                    <div v-for="file in stagedFiles" :key="file.path" class="file-item d-flex align-items-center mb-2">
                        <input
                            :id="'staged-file-' + file.path"
                            v-model="selectedStagedFiles"
                            type="checkbox"
                            :value="file.path"
                            class="form-check-input me-2"
                        />
                        <span :class="getFileStatusClass(file.status)" class="me-2">
                            {{ file.status }}
                        </span>
                        <span class="text-monospace">{{ file.path }}</span>
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

            <!-- Credentials Dialog -->
            <div v-if="showCredentialsDialog" class="mb-3">
                <div class="alert alert-info">
                    {{ $t('gitCredentialsRequired') }}
                </div>
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
                    <label for="gitPassword" class="form-label">{{ $t('passwordOrToken') || 'Password / Token' }}</label>
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

        <template #footer>
            <button class="btn btn-secondary" @click="onHide">
                {{ $t('close') }}
            </button>

            <!-- Sync Preview Mode Buttons -->
            <template v-if="showSyncPreview">
                <button
                    class="btn btn-secondary"
                    :disabled="processing"
                    @click="cancelSyncPreview"
                >
                    {{ $t('cancel') || 'Cancel' }}
                </button>
                <button
                    v-if="syncPreview.needsCredentials && (!credentials.username || !credentials.password)"
                    class="btn btn-primary"
                    @click="retrySyncPreview"
                >
                    {{ $t('retryWithCredentials') || 'Retry with Credentials' }}
                </button>
                <template v-if="!syncPreview.needsCredentials || (credentials.username && credentials.password)">
                    <button
                        v-if="syncPreview.hasRemoteChanges"
                        class="btn btn-info"
                        :disabled="processing"
                        @click="pullOnly"
                    >
                        <font-awesome-icon icon="download" class="me-1" />
                        {{ $t('pullOnly') || 'Pull Only' }}
                    </button>
                    <button
                        v-if="syncPreview.hasLocalChanges"
                        class="btn btn-warning"
                        :disabled="processing"
                        @click="pushOnly"
                    >
                        <font-awesome-icon icon="upload" class="me-1" />
                        {{ $t('pushOnly') || 'Push Only' }}
                    </button>
                    <button
                        class="btn btn-success"
                        :disabled="processing || (!syncPreview.hasLocalChanges && !syncPreview.hasRemoteChanges)"
                        @click="confirmSync"
                    >
                        <font-awesome-icon icon="sync" class="me-1" />
                        {{ $t('confirmSync') || 'Confirm Sync' }}
                    </button>
                </template>
            </template>

            <!-- Normal Mode Buttons -->
            <template v-else>
                <button
                    v-if="isGitRepo && selectedFiles.length > 0"
                    class="btn btn-primary"
                    :disabled="processing"
                    @click="addFiles"
                >
                    <font-awesome-icon icon="plus" class="me-1" />
                    {{ $t('addFiles') }}
                </button>
                <button
                    v-if="isGitRepo && selectedStagedFiles.length > 0"
                    class="btn btn-warning"
                    :disabled="processing"
                    @click="unstageFiles"
                >
                    <font-awesome-icon icon="minus" class="me-1" />
                    {{ $t('unstageFiles') }}
                </button>
                <button
                    v-if="isGitRepo && gitStatus.files.some(f => f.staged)"
                    class="btn btn-success"
                    :disabled="processing || !commitMessage"
                    @click="commit"
                >
                    <font-awesome-icon icon="check" class="me-1" />
                    {{ $t('commit') }}
                </button>
                <button
                    v-if="isGitRepo && (gitStatus.ahead > 0 || gitStatus.behind > 0)"
                    class="btn btn-info"
                    :disabled="processing"
                    @click="startSync"
                >
                    <font-awesome-icon icon="sync" class="me-1" />
                    {{ $t('sync') }}
                </button>
            </template>
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
        stackName: {
            type: String,
            required: true,
        },
        endpoint: {
            type: String,
            default: "",
        },
    },
    data() {
        return {
            show: false,
            loading: false,
            processing: false,
            isGitRepo: true,
            gitStatus: {
                files: [],
                current: "",
                tracking: null,
                ahead: 0,
                behind: 0,
            },
            selectedFiles: [],
            selectedStagedFiles: [],
            commitMessage: "",
            showCredentialsDialog: false,
            credentials: {
                username: "",
                password: "",
            },
            showSyncPreview: false,
            syncPreview: {
                hasLocalChanges: false,
                hasRemoteChanges: false,
                localChanges: [],
                remoteCommits: [],
                ahead: 0,
                behind: 0,
                needsCredentials: false,
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
            this.showCredentialsDialog = false;
            this.showSyncPreview = false;
            this.credentials = { username: "",
                password: "" };
        },

        async loadGitStatus() {
            this.loading = true;
            this.$root.emitAgent(this.endpoint, "getGitStatus", this.stackName, (res) => {
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

        async checkCredentials() {
            this.$root.emitAgent(this.endpoint, "getGitCredentials", (res) => {
                if (res.ok && !res.hasCredentials) {
                    this.showCredentialsDialog = true;
                }
            });
        },

        async addFiles() {
            if (this.selectedFiles.length === 0) {
                return;
            }

            this.processing = true;
            this.$root.emitAgent(this.endpoint, "gitAddFiles", this.stackName, this.selectedFiles, (res) => {
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
            this.$root.emitAgent(this.endpoint, "gitUnstageFiles", this.stackName, this.selectedStagedFiles, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
                if (res.ok) {
                    this.selectedStagedFiles = [];
                    this.loadGitStatus();
                }
            });
        },

        async commit() {
            if (!this.commitMessage) {
                this.$root.toastError(this.$t("pleaseEnterCommitMessage"));
                return;
            }

            this.processing = true;
            this.$root.emitAgent(this.endpoint, "gitCommit", this.stackName, this.commitMessage, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
                if (res.ok) {
                    this.commitMessage = "";
                    this.loadGitStatus();
                }
            });
        },

        async startSync() {
            this.showSyncPreview = true;
            this.loading = true;

            const creds = this.credentials.username && this.credentials.password
                ? this.credentials
                : null;

            this.$root.emitAgent(this.endpoint, "getSyncPreview", this.stackName, creds, (res) => {
                this.loading = false;
                if (res.ok && res.preview) {
                    this.syncPreview = res.preview;

                    // If credentials are needed, show the dialog
                    if (this.syncPreview.needsCredentials) {
                        this.showCredentialsDialog = true;
                    }
                } else {
                    this.$root.toastError(res.msg || "Failed to get sync preview");
                    this.showSyncPreview = false;
                }
            });
        },

        async retrySyncPreview() {
            if (!this.credentials.username || !this.credentials.password) {
                this.$root.toastError("Please enter both username and password");
                return;
            }
            await this.startSync();
        },

        async confirmSync() {
            this.processing = true;

            const creds = this.credentials.username && this.credentials.password
                ? this.credentials
                : null;

            this.$root.emitAgent(this.endpoint, "gitSync", this.stackName, creds, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
                if (res.ok) {
                    this.showSyncPreview = false;
                    this.showCredentialsDialog = false;
                    this.loadGitStatus();
                }
            });
        },

        async pullOnly() {
            this.processing = true;

            const creds = this.credentials.username && this.credentials.password
                ? this.credentials
                : null;

            this.$root.emitAgent(this.endpoint, "gitPull", this.stackName, creds, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
                if (res.ok) {
                    this.showSyncPreview = false;
                    this.showCredentialsDialog = false;
                    this.loadGitStatus();
                }
            });
        },

        async pushOnly() {
            this.processing = true;

            const creds = this.credentials.username && this.credentials.password
                ? this.credentials
                : null;

            this.$root.emitAgent(this.endpoint, "gitPush", this.stackName, creds, (res) => {
                this.processing = false;
                this.$root.toastRes(res);
                if (res.ok) {
                    this.showSyncPreview = false;
                    this.showCredentialsDialog = false;
                    this.loadGitStatus();
                }
            });
        },

        cancelSyncPreview() {
            this.showSyncPreview = false;
            this.showCredentialsDialog = false;
        },

        getFileStatusClass(status) {
            const statusClasses = {
                "modified": "badge bg-warning",
                "untracked": "badge bg-secondary",
                "new file": "badge bg-success",
                "deleted": "badge bg-danger",
                "renamed": "badge bg-info",
                "staged": "badge bg-primary",
            };
            return statusClasses[status] || "badge bg-secondary";
        },

        formatDate(dateStr) {
            const date = new Date(dateStr);
            return date.toLocaleString();
        },
    },
};
</script>

<style scoped>
.file-list {
    max-height: 300px;
    overflow-y: auto;
    border: 1px solid #dee2e6;
    border-radius: 0.25rem;
    padding: 0.5rem;
}

.file-list.staged-list {
    background-color: #f8f9fa;
    border-color: #28a745;
}

.file-item {
    padding: 0.25rem;
    border-bottom: 1px solid #f0f0f0;
}

.file-item:last-child {
    border-bottom: none;
}

.text-monospace {
    font-family: monospace;
    font-size: 0.9rem;
}

.commit-list {
    max-height: 200px;
    overflow-y: auto;
    border: 1px solid #dee2e6;
    border-radius: 0.25rem;
    padding: 0.5rem;
}

.commit-item {
    padding: 0.5rem;
    border-bottom: 1px solid #f0f0f0;
}

.commit-item:last-child {
    border-bottom: none;
}

.commit-meta {
    margin-top: 0.25rem;
    margin-left: 2rem;
}
</style>
