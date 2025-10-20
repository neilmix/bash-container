# bash-container

I've been experimenting with using lightweight local docker containers for
AI assisted coding with claude code. This way I can (mostly) safely turn off permission 
checks. My primary concern is with restricting file access to a sandbox, I'm less concerned 
about rogue Internet download infiltration. I'm documenting this here just as a helpful 
guide for friends and colleagues who want to try this approach.

There is nothing revelatory here and could be improved in lots of ways. This is just
a simple hack for my situation that keeps me productive.

## installation
Copy and paste the below code into your .bash_profile and source it.

## usage

Invoke your container:
```bash
container
```

Rebuild after changing the dockerfile template:
```bash
container rebuild
```

Invoke the container opening a port:
```bash
container 8080
```

When invoking the container you'll be immediately dropped into a bash prompt. Then you
can invoke claude with permissions turned off. No more endless prompting for your
approval.
```bash
claude --dangerously-skip-permissions
```

## code

```bash
container() {
  # Find project root (git repo if available, else current dir)
  local top proj name image
  top="$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
  proj="$(basename "$top")"
  name="${proj}-vibe"
  image="${proj}-vibe:latest"
  local dockerfile_path="$top/.Dockerfile.vibe"

  # build a dockerfile
  cat > "$dockerfile_path" <<"EOF"
FROM node:lts-bookworm
RUN apt-get update \
 && apt-get install -y --no-install-recommends python3 python3-venv python3-pip \
 && rm -rf /var/lib/apt/lists/*
RUN pip install beads-mcp --break-system-packages
RUN npm install -g @anthropic-ai/claude-code@latest
RUN npm install -g chrome-devtools
RUN useradd -ms /bin/bash dev && echo "dev ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
USER dev
WORKDIR /repo
EOF

  # Optional: force rebuild from scratch
  if [[ "${1-}" == "rebuild" ]]; then
    echo "Rebuilding image $image (no cache) and recreating container $name..."
    docker rm -f "$name" >/dev/null 2>&1 || true
    (cd "$top" && docker build --no-cache -f "$dockerfile_path" -t "$image" .)
    return
  fi

  # Build the image if it doesn't exist
  if ! docker image inspect "$image" >/dev/null 2>&1; then
    echo "Building image $image..."
    (cd "$top" && docker build -f "$dockerfile_path" -t "$image" .)
  fi

  # all of our container working files and junk (i.e. not git tracked) go here
  mkdir -p .scratch

  # if we didn't rebuild (see above) treat an arg as a port to open up on the container
  if [ "$1" ]; then
  	echo "forwarding port $1"
  fi
  docker run --rm -it \
	--name $name \
	-v "$PWD":/repo -w /repo \
	--read-only --tmpfs /tmp:exec \
	-v "$PWD/.scratch":/scratch \
	--env-file .env \
	-e HOME=/scratch \
	${1:+-p $1:$1} \
	$name bash
}
```