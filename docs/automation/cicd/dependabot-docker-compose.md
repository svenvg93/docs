---
title: Dependabot Automation
categories:
- DevOps
- CI/CD
date: 2025-05-12
description: Keep your Docker Compose dependencies secure and up to date by automating
  Dependabot configuration with a simple Bash script and GitHub Actions.
draft: false
tags:
- dependabot
- docker
- github actions
---


Keeping dependencies up to date is essential for security and maintainability, but manually managing updates across multiple `docker-compose.yml` files in a project can be tedious. In this post, I’ll show you a small Bash script I wrote to automate the generation of a `dependabot.yml` file. It scans your repo for all Docker Compose files and configures Dependabot to check them for updates monthly. It’s lightweight, efficient, and ensures you never miss a patch. Let’s dive in. We will automate the updating the `dependabot.yml` with Github Actions.

## What is dependabot?
[Dependabot] is a built-in GitHub tool that automatically checks your project dependencies for updates. It can open pull requests when new versions of your dependencies are available - helping you stay secure and up to date with minimal effort. For Docker Compose projects, it monitors container image tags and notifies you when a newer version is published.

## Create generate-dependabot.sh

In the top level of your directory, create a script file to generate `dependabot.yml`:

```bash
nano generate-dependabot.sh
```

Paste the following content into the file:

```bash title="generate-dependabot.sh"
#!/bin/bash

# Script to generate or update dependabot.yml based on docker-compose.yml files
# Usage: ./generate-dependabot.sh

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Function to print colored output
print_status() {
    echo -e "${BLUE}[INFO]${NC} $1"
}

print_success() {
    echo -e "${GREEN}[SUCCESS]${NC} $1"
}

print_warning() {
    echo -e "${YELLOW}[WARNING]${NC} $1"
}

print_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

print_status "Starting dependabot.yml generation..."

mkdir -p .github

tmpfile=$(mktemp)
trap 'rm -f "$tmpfile"' EXIT

# Header
cat > "$tmpfile" <<'YAML'
version: 2
updates:
  - package-ecosystem: "docker-compose"
    directories:
YAML

# Find and sort all docker-compose.yml directories
print_status "Scanning for docker-compose.yml files..."

found_directories=0
while IFS= read -r file; do
    dir=$(dirname "$file" | sed 's|^\./||')
    print_status "Found compose file in: $dir"
    echo "      - \"/$dir\"" >> "$tmpfile"
    ((found_directories++))
done < <(find . -name "docker-compose.yml" -type f | sort)

if [[ $found_directories -eq 0 ]]; then
    print_warning "No docker-compose.yml files found"
    exit 0
fi

# Append the schedule block
cat >> "$tmpfile" <<'YAML'
    schedule:
      interval: "daily"
YAML

# Install if changed
if ! [ -f .github/dependabot.yml ] || ! cmp -s "$tmpfile" .github/dependabot.yml; then
  mv "$tmpfile" .github/dependabot.yml
  print_success "Updated .github/dependabot.yml!"
  print_status "Found $found_directories directories with compose files"
else
  print_status "No changes to .github/dependabot.yml"
  print_status "Found $found_directories directories with compose files"
fi

print_success "Dependabot configuration generation completed!"
```

Make the script executable:

```bash
chmod +x generate-dependabot.sh
```

When you run the script using `./generate-dependabot.sh`, it will create (or update) the `.github/dependabot.yml` file with a list of all directories containing docker-compose.yml files. Commit this file to your Git repository — Dependabot will then automatically check for updated Docker image versions every month and open a pull request if any updates are found.

You can change the interval to weekly or daily if you prefer.