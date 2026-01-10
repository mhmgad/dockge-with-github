<div align="center" width="100%">
    <img src="./frontend/public/icon.svg" width="128" alt="" />
</div>

# Dockge with GitHub Integration

> **Note:** This is a fork of [louislam/dockge](https://github.com/louislam/dockge) with enhanced Git/GitHub integration and additional features for managing unmanaged Docker Compose stacks.

A fancy, easy-to-use and reactive self-hosted docker compose.yaml stack-oriented manager.

[![Upstream Repo](https://img.shields.io/badge/upstream-louislam%2Fdockge-blue?logo=github)](https://github.com/louislam/dockge) [![GitHub Repo stars](https://img.shields.io/github/stars/louislam/dockge?logo=github&style=flat&label=upstream%20stars)](https://github.com/louislam/dockge) [![Docker Pulls](https://img.shields.io/docker/pulls/mhmgad/dockge-plus?logo=docker&label=docker%20pulls)](https://hub.docker.com/r/mhmgad/dockge-plus)

<img src="https://github.com/louislam/dockge/assets/1336778/26a583e1-ecb1-4a8d-aedf-76157d714ad7" width="900" alt="" />

View Video: https://youtu.be/AWAlOQeNpgU?t=48

## ⭐ Features

### Enhanced Features in This Fork

- 🔀 **GitHub Repository Management** - Full Git/GitHub integration for managing stacks with version control:
  - Check git status (branch, changed files, commits ahead/behind)
  - Add files to staging area
  - Commit staged changes with custom messages
  - Push and pull changes to/from remote repositories
  - Clone GitHub repositories (public or private) to create new stacks
  - Secure credential management for private repositories
  
- 📁 **Nested Folder Support with Grouping** - Organize stacks in nested folders with automatic grouping:
  - Create stacks in nested directories (e.g., `/opt/stacks/homeserver/plex`, `/opt/stacks/homeserver/jellyfin`)
  - Automatic grouping by parent folder in the UI
  - Example: A `homeserver` folder containing `plex` and `jellyfin` subfolders will show as a "homeserver" group with 2 stacks
  - Git sync button at the folder/group level for batch operations
  
- 📦 **Unmanaged Stack Support** - View and control Docker Compose stacks not created by Dockge:
  - Automatically discover running Docker Compose stacks on the system
  - View status and container information for unmanaged stacks
  - Stop unmanaged stacks (via `docker compose stop` or `docker compose down`)
  - Separate "Unmanaged" group in the stack list for clear distinction
  - Note: Cannot start, restart, or edit unmanaged stacks (they remain read-only except for stop operations)

### Core Features (from upstream)

- 🧑‍💼 Manage your `compose.yaml` files
  - Create/Edit/Start/Stop/Restart/Delete
  - Update Docker Images
- ⌨️ Interactive Editor for `compose.yaml`
- 🦦 Interactive Web Terminal
- 🕷️ Multiple agents support - Manage stacks from different Docker hosts in one interface
- 🏪 Convert `docker run ...` commands into `compose.yaml`
- 📙 File based structure - Compose files stored on your drive, interact with normal `docker compose` commands
- 🚄 Reactive - Real-time progress and terminal output
- 🐣 Easy-to-use & fancy UI - If you love Uptime Kuma's UI/UX, you will love this one too

<img src="https://github.com/louislam/dockge/assets/1336778/cc071864-592e-4909-b73a-343a57494002" width=300 />

![](https://github.com/louislam/dockge/assets/1336778/89fc1023-b069-42c0-a01c-918c495f1a6a)

## 🔧 How to Install

### Quick Start (Using Pre-built Docker Images)

This fork provides pre-built Docker images at `mhmgad/dockge-plus` with all the enhanced features included.

Requirements:
- [Docker](https://docs.docker.com/engine/install/) 20+ / Podman
- (Podman only) podman-docker (Debian: `apt install podman-docker`)
- OS:
  - Major Linux distros that can run Docker/Podman such as:
     - ✅ Ubuntu
     - ✅ Debian (Bullseye or newer)
     - ✅ Raspbian (Bullseye or newer)
     - ✅ CentOS
     - ✅ Fedora
     - ✅ ArchLinux
  - ❌ Debian/Raspbian Buster or lower is not supported
  - ❌ Windows (Will be supported later)
- Arch: armv7, arm64, amd64 (a.k.a x86_64)

### Installation Steps

- Default Stacks Directory: `/opt/stacks`
- Default Port: 5001

```bash
# Create directories that store your stacks and stores Dockge's stack
mkdir -p /opt/stacks /opt/dockge
cd /opt/dockge

# Download the compose.yaml from this fork
curl https://raw.githubusercontent.com/mhmgad/dockge-with-github/master/compose.yaml --output compose.yaml

# Start the server
docker compose up -d

# If you are using docker-compose V1 or Podman
# docker-compose up -d
```

Dockge is now running on http://localhost:5001

### Advanced

If you want to store your stacks in another directory, you can generate your compose.yaml file by using the following URL with custom query strings.

```
# Download your compose.yaml
curl "https://dockge.kuma.pet/compose.yaml?port=5001&stacksPath=/opt/stacks" --output compose.yaml
```

- port=`5001`
- stacksPath=`/opt/stacks`

Interactive compose.yaml generator is available on: 
https://dockge.kuma.pet

## How to Update

```bash
cd /opt/dockge
docker compose pull && docker compose up -d
```

## 🐳 Docker Images

Pre-built Docker images are available at:
- **Docker Hub**: [`mhmgad/dockge-plus`](https://hub.docker.com/r/mhmgad/dockge-plus)
- **Tags**: `latest`, `<version>` (e.g., `1.4.2`), `master`

The images are automatically built and published via GitHub Actions on every push to the master branch. See [DOCKER_SETUP.md](./DOCKER_SETUP.md) for details about the build process.

## 🔨 Building This Fork from Source

If you want to use this fork with the Git integration and unmanaged stack features, you can build it from source:

### Prerequisites
- Node.js >= 22.14.0
- npm
- Git
- Docker (for building the Docker image)

### Steps

1. **Clone the repository:**
```bash
git clone https://github.com/mhmgad/dockge-with-github.git
cd dockge-with-github
```

2. **Install dependencies:**
```bash
npm install
```

3. **Build the frontend:**
```bash
npm run build
```

4. **Run in development mode (optional):**
```bash
# Terminal 1 - Backend
npm run dev:backend

# Terminal 2 - Frontend
npm run dev:frontend
```

5. **Build Docker image (for production):**
```bash
# Note: Pre-built images are available at mhmgad/dockge-plus
# Only build from source if you need custom modifications
docker build -t dockge-with-github:latest -f docker/Dockerfile .
```

6. **Run with Docker:**
```bash
# Create directories
mkdir -p /opt/stacks /opt/dockge
cd /opt/dockge

# Download the compose.yaml
curl https://raw.githubusercontent.com/mhmgad/dockge-with-github/master/compose.yaml --output compose.yaml

# Start the server
docker compose up -d
```

For more development details, see [CONTRIBUTING.md](./CONTRIBUTING.md).

## Screenshots

![](https://github.com/louislam/dockge/assets/1336778/e7ff0222-af2e-405c-b533-4eab04791b40)


![](https://github.com/louislam/dockge/assets/1336778/7139e88c-77ed-4d45-96e3-00b66d36d871)

![](https://github.com/louislam/dockge/assets/1336778/f019944c-0e87-405b-a1b8-625b35de1eeb)

![](https://github.com/louislam/dockge/assets/1336778/a4478d23-b1c4-4991-8768-1a7cad3472e3)


## Motivations

### Original Dockge (by Louis Lam)
- Portainer's stack management can be frustrating with unclear error messages and slow deployments
- Desire to develop with ES Module + TypeScript

### This Fork's Additions
- **GitOps Workflows**: Many teams use Git for infrastructure-as-code; having Git operations in the UI streamlines Docker Compose management
- **Nested Organization**: Real deployments often have logical groupings (e.g., media server, home automation, development) that benefit from folder-based organization
- **Complete System View**: Systems often have pre-existing Docker Compose stacks; this fork lets you monitor and control all stacks in one place

If you love this project, please consider giving it a ⭐.


## 🗣️ Community and Contribution

### For Issues Related to Enhanced Features (Git Integration, Unmanaged Stacks)
- **Bug Reports**: [mhmgad/dockge-with-github/issues](https://github.com/mhmgad/dockge-with-github/issues)
- **Discussions**: [mhmgad/dockge-with-github/discussions](https://github.com/mhmgad/dockge-with-github/discussions)

### For Core Dockge Issues
- **Bug Reports**: [louislam/dockge/issues](https://github.com/louislam/dockge/issues)
- **Discussions**: [louislam/dockge/discussions](https://github.com/louislam/dockge/discussions)

### Translation
If you want to translate Dockge into your language, please read [Translation Guide](https://github.com/louislam/dockge/blob/master/frontend/src/lang/README.md)

### Create a Pull Request

For contributions related to Git integration or unmanaged stack features, submit PRs to this fork. For core Dockge features, please submit to the [upstream repository](https://github.com/louislam/dockge) and read their [contribution guide](https://github.com/louislam/dockge/blob/master/CONTRIBUTING.md).

## FAQ

#### "Dockge"?

"Dockge" is a coinage word which is created by myself. I originally hoped it sounds like `Dodge`, but apparently many people called it `Dockage`, it is also acceptable.

The naming idea came from Twitch emotes like `sadge`, `bedge` or `wokege`. They all end in `-ge`.

#### Can I manage a single container without `compose.yaml`?

The main objective of Dockge is to try to use the docker `compose.yaml` for everything. If you want to manage a single container, you can just use Portainer or Docker CLI.

#### Can I manage existing stacks?

Yes, you can. However, you need to move your compose file into the stacks directory:

1. Stop your stack
2. Move your compose file into `/opt/stacks/<stackName>/compose.yaml`
3. In Dockge, click the " Scan Stacks Folder" button in the top-right corner's dropdown menu
4. Now you should see your stack in the list

#### Is Dockge a Portainer replacement?

Yes or no. Portainer provides a lot of Docker features. While Dockge is currently only focusing on docker-compose with a better user interface and better user experience.

If you want to manage your container with docker-compose only, the answer may be yes.

If you still need to manage something like docker networks, single containers, the answer may be no.

#### Can I install both Dockge and Portainer?

Yes, you can.

#### How does Git Integration work?

**Git Status Management:**

If your stack directory is a Git repository, you can use the "Git Status" button on the stack's page to:
- View the current git status (branch, changed files, commits ahead/behind)
- Add untracked files to staging
- Commit staged changes
- Push changes to the remote repository
- Pull changes from the remote repository

**Clone Repository:**

You can clone a Git repository to create a new stack:
1. Click the "Clone Repository" button on the Dashboard
2. Enter the repository URL (HTTPS or SSH)
3. Enter a name for the new stack
4. For private repositories, provide your GitHub credentials
5. The repository will be cloned and a new stack will be created

The first time you push, pull, or clone a private repository, you'll be asked for your GitHub credentials (username and password/token). These credentials are stored for future operations.

**Note**: For SSH remotes, please configure SSH keys separately as the credential injection only works with HTTPS URLs.

## 🙏 Credits and Attribution

This project is a fork of [Dockge](https://github.com/louislam/dockge) created by [Louis Lam](https://github.com/louislam), the creator of [Uptime Kuma](https://github.com/louislam/uptime-kuma).

**Upstream Project**: [louislam/dockge](https://github.com/louislam/dockge)

All core functionality and the beautiful, reactive UI are thanks to the original Dockge project. This fork adds Git/GitHub integration and enhanced support for managing unmanaged Docker Compose stacks.

If you love this project, please consider giving a ⭐ to both:
- This fork: [mhmgad/dockge-with-github](https://github.com/mhmgad/dockge-with-github)
- The original: [louislam/dockge](https://github.com/louislam/dockge)

## Others

Dockge is built on top of [Compose V2](https://docs.docker.com/compose/migrate/). `compose.yaml`  also known as `docker-compose.yml`.
