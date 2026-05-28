# DevContainer Configuration

## Overview
This repository includes a devcontainer configuration (`.devcontainer/devcontainer.json`) that defines a consistent development environment for the ShaDo checklists application.

## What the devcontainer.json configures

### 1. Base Docker Image
- **Configuration**: `"image": "mcr.microsoft.com/devcontainers/universal:2"`
- **Description**: Uses Microsoft's Universal Dev Container image (version 2), which includes:
  - Common development tools and runtimes (Node.js, Python, etc.)
  - Git and other version control tools
  - Common shell utilities
  - Pre-configured development environment

### 2. Features
- **Configuration**: `"features": {}`
- **Description**: Currently empty, but this section can be used to add additional development tools and capabilities to the container
- **Examples of what could be added**:
  - Docker-in-Docker
  - Additional language runtimes
  - CLI tools
  - Extensions

### 3. Remote Environment Variables
- **Configuration**: `"remoteEnv": { "TEST_REMOTE_ENV": "checklists" }`
- **Description**: Sets custom environment variables that will be available inside the dev container
- **Current variable**: `TEST_REMOTE_ENV=checklists` - appears to be used for testing or identification purposes

## Benefits of Using DevContainers

1. **Consistency**: All developers work in the same environment regardless of their host OS
2. **Easy Setup**: New developers can get started quickly without manual environment configuration
3. **Isolation**: Development environment is isolated from the host system
4. **Reproducibility**: The environment can be easily recreated and versioned

## How to Use

To use this devcontainer:
1. Install VS Code and the Dev Containers extension
2. Open this repository in VS Code
3. Click "Reopen in Container" when prompted (or use Command Palette: "Dev Containers: Reopen in Container")
4. VS Code will build and start the dev container with all the configured settings

## Technology Stack Context

This devcontainer supports development of:
- **Frontend**: React
- **Backend**: Node.js
- **Database**: MongoDB

The Universal Dev Container image includes the necessary tools for this stack.
