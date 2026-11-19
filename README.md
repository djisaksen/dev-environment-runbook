# Development Environment Setup (Windows + WSL 2 + Ubuntu)

This documents how the course development environment was set up on a Windows PC, in the order the steps were done. Project files live in the **Linux home** (`~/projects`), never under `/mnt/c`.

## Hardware and software assumptions

- Windows 10 (recent build) or Windows 11, 64-bit (x86_64)
- Virtualization enabled in BIOS/UEFI (needed for WSL 2 and Docker)
- 16 GB RAM recommended (8 GB works, but Docker plus VS Code gets tight)
- At least 20 GB free disk space on `C:` (Ubuntu, Docker images and `node_modules` add up)
- Internet access for package downloads and GitHub
- Administrator rights on Windows for the first WSL and Docker install
- Machine specs: _(add yours here: CPU, RAM, GPU, Windows version)_

## Commands run

Each block says which window to use. The prompt tells you: `PS C:\...>` is PowerShell, `daniel@PC:~$` is Ubuntu.

### 1. Install WSL with Ubuntu (PowerShell)

```powershell
wsl --install -d Ubuntu
```

Restart if asked, then open **Ubuntu** from the Start menu and create a Linux username and password.

Note: `wsl` is a Windows command. Running `wsl --status` inside Ubuntu gives "command not found", which is expected. Check WSL from PowerShell with `wsl --status` and `wsl -l -v` (Ubuntu should show VERSION 2).

### 2. Create the projects folder (Ubuntu)

```bash
mkdir -p ~/projects
cd ~/projects
pwd    # should print /home/<username>/projects
```

### 3. Base tools and Git identity (Ubuntu)

```bash
sudo apt install git curl build-essential
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

Replace the placeholder name and email with your own before running the `git config` lines. Use the email on your GitHub account.

### 4. Node.js through nvm (Ubuntu)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.41.0/install.sh | bash
```

Close and reopen Ubuntu (or load nvm in the current window), then:

```bash
nvm install --lts
node --version
npm --version
which node    # must start with /home/<username>/.nvm/
which npm     # must also start with /home/<username>/.nvm/
```

### 5. Docker Desktop (Windows, then Ubuntu)

Install Docker Desktop on **Windows** (not inside Ubuntu), using the WSL 2 backend. Then in Docker Desktop go to **Settings, Resources, WSL integration**, switch on **Ubuntu**, and click **Apply & restart**. Reopen Ubuntu and test:

```bash
which docker
docker version
docker run hello-world
```

`docker version` should show both a Client and a Server section. Before the WSL integration was switched on, these commands failed with "The command 'docker' could not be found in this WSL 2 distro" (see the troubleshooting appendix).

### 5b. SSH key for GitHub (Ubuntu)

```bash
ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub
```

Copy the single line that starts with `ssh-ed25519` and add it on GitHub under **Settings, SSH and GPG keys, New SSH key**. Never share the file without `.pub`; it is the private key. Then test:

```bash
ssh -T git@github.com
```

Answer `yes` to the first-connection fingerprint prompt. Success looks like: `Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.`

### 6. Ignore secrets and junk files

Every project gets a `.gitignore` that excludes at least `.env`, `__pycache__/`, `node_modules/`, `.venv/` and editor swap files. **Never commit secrets.** From later weeks on, put API keys in a local `.env` file that is gitignored, and commit a `.env.example` with fake values instead.

Check before your first commit:

```bash
git status                  # .env must not be listed
git check-ignore -v .env    # shows which rule ignores it
```

## Open the project in VS Code (Remote-WSL)

One-time setup on **Windows**:
1. Install Visual Studio Code.
2. Install the **WSL** extension (publisher: Microsoft, id `ms-vscode-remote.remote-wsl`).

Each time, from **Ubuntu**:

```bash
cd ~/projects/<your-project>
code .
```

The first run installs a small VS Code Server inside Ubuntu, which takes a moment. Confirm it worked: the bottom-left corner of VS Code shows **WSL: Ubuntu**, and the integrated terminal prompt looks like `daniel@PC:~/projects/<your-project>$`.

Alternative: in VS Code press `Ctrl + Shift + P`, run **WSL: Connect to WSL**, then use **File, Open Folder** and pick a folder under `/home/<username>/`.

## Troubleshooting appendix

### Everything feels slow (git, `npm install`, builds)

You are probably working under `/mnt/c`, the Windows drive. Linux reads it slowly, and file watching and permissions (`chmod`) behave oddly there.

- Run `pwd`. If it starts with `/mnt/c`, move the project.
- Clone or copy it into the Linux home instead:

```bash
cd ~/projects
git clone git@github.com:<user>/<repo>.git
```

- To view Linux files from Windows, run `explorer.exe .` inside the folder, or browse to `\\wsl$\Ubuntu\home\<username>`.
- Do not copy project files into Windows with File Explorer and keep working there.

### `docker` not found, or "could not be found in this WSL 2 distro"

1. Make sure Docker Desktop is open and running (check the Windows system tray).
2. In Docker Desktop: **Settings, Resources, WSL integration**, switch on **Ubuntu**, then **Apply & restart**.
3. Close the Ubuntu window and open a new one.
4. Check where the command comes from:

```bash
which docker
docker version
```

- A path starting with `/mnt/c/` means Ubuntu is seeing the Windows `docker.exe`, not a connected one. Redo steps 2 and 3.
- If `wsl -l -v` in PowerShell shows Ubuntu as VERSION 1, convert it with `wsl --set-version Ubuntu 2`.
- Do not run `sudo apt install docker.io` as a shortcut, and do not install Docker Desktop and Docker Engine together.

### VS Code opens a Windows terminal (PowerShell or cmd) instead of Ubuntu

1. Look at the bottom-left corner. If it does not say **WSL: Ubuntu**, you are in a plain Windows window. Close it and run `code .` from the Ubuntu terminal instead (or use **WSL: Connect to WSL**).
2. If the window says **WSL: Ubuntu** but the terminal is still PowerShell, press `Ctrl + Shift + P`, run **Terminal: Select Default Profile**, and choose **bash** (or **Ubuntu (WSL)**).
3. Open a new terminal with `` Ctrl + ` `` and check the prompt looks like `daniel@PC:~/...$`.

### `nvm: command not found` right after installing

The open terminal window started before the installer edited `~/.bashrc`. Close Ubuntu and open a new window, or run:

```bash
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
```

### `npm` or `node` points to `/mnt/c/...`

That is the Windows copy leaking into Ubuntu. Run `nvm install --lts` in Ubuntu and check that `which node` and `which npm` start with `/home/<username>/.nvm/`.

### Scripts fail with `^M: bad interpreter` (CRLF line endings)

The file has Windows line endings. Fix a file with:

```bash
sed -i 's/\r$//' script.sh
```

To prevent it, create and edit files inside Ubuntu or through the VS Code WSL window (the status bar should show `LF`, not `CRLF`), and set:

```bash
git config --global core.autocrlf input
```

### `Permission denied` when running a script

Make it executable (this works reliably only in the Linux home, not on `/mnt/c`):

```bash
chmod +x myscript.sh
./myscript.sh
```

### `Permission denied (publickey)` with GitHub

The public key on GitHub is missing or was cut off when pasted. Print it again with `cat ~/.ssh/id_ed25519.pub`, copy the entire line, and re-add it under **Settings, SSH and GPG keys**.

### Lost the SSH passphrase

It cannot be recovered. Create a new key with `ssh-keygen -t ed25519 -C "you@example.com"`, add the new public key to GitHub, and delete the old one there. To change a passphrase you remember: `ssh-keygen -p -f ~/.ssh/id_ed25519`.
