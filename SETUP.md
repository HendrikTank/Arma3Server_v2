# Arma3Server_v2 Setup Guide

This guide will walk you through setting up and running the Arma3Server_v2 Docker-based dedicated server launcher.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Quick Start (Docker)](#quick-start-docker)
- [Configuration](#configuration)
  - [Directory Structure](#directory-structure)
  - [Steam Credentials](#steam-credentials)
  - [Server Configuration](#server-configuration)
- [Running the Server](#running-the-server)
- [Managing Mods and DLCs](#managing-mods-and-dlcs)
- [Headless Clients](#headless-clients)
- [Troubleshooting](#troubleshooting)
- [Advanced Configuration](#advanced-configuration)

## Prerequisites

Before you begin, ensure you have:

1. **Docker** installed on your host system
   - Install from: https://docs.docker.com/get-docker/
   - Verify: `docker --version`

2. **Steam Account** with Arma 3 license
   - Required for downloading game files and Workshop mods via SteamCMD
   - Recommended: Disable Steam Guard 2FA for automated downloads, or be prepared to handle authentication manually

3. **System Requirements**
   - **CPU**: Multi-core processor (4+ cores recommended)
   - **RAM**: 8GB minimum, 16GB+ recommended for production
   - **Storage**: 50GB+ free space (game files + mods)
   - **Network**: Stable internet connection for initial downloads
   - **Ports**: UDP ports 2302-2306 available

4. **Basic Knowledge**
   - Docker concepts (containers, volumes, images)
   - JSON configuration files
   - Linux file permissions and paths

## Quick Start (Docker)

### 1. Build the Docker Image

Clone the repository and build the image:

```bash
git clone https://github.com/HendrikTank/Arma3Server_v2.git
cd Arma3Server_v2
docker build -t arma3server:latest .
```

### 2. Prepare Directory Structure

Create the required directories on your host:

```bash
mkdir -p ~/arma3-server/arma3
mkdir -p ~/arma3-server/share/arma3/server-common
mkdir -p ~/arma3-server/share/arma3/this-server
```

### 3. Create Steam Credentials File

Create a file at `~/arma3-server/share/steam_credentials.json`:

```json
{
  "steam_user": "your_steam_username",
  "steam_password": "your_steam_password"
}
```

**Important**: Keep this file secure! Consider using file permissions:
```bash
chmod 600 ~/arma3-server/share/steam_credentials.json
```

### 4. Set Up Server Configuration

Create the following directory structure in `~/arma3-server/share/arma3/this-server/`:

```
this-server/
├── config/
│   ├── generated_a3server.cfg  # Server configuration
│   └── params.cfg               # Mission parameters (optional)
├── mpmissions/                  # Mission files
│   └── your_mission.pbo
├── mods/                        # Custom/local mods (optional)
└── servermods/                  # Server-only mods (optional)
```

In `~/arma3-server/share/arma3/server-common/`:

```
server-common/
├── basic.cfg                    # Basic server settings
├── dlcs/                        # DLC content directories
└── mods/                        # Shared mod storage
```

### 5. Create basic.cfg

Create `~/arma3-server/share/arma3/server-common/basic.cfg`:

```
MaxMsgSend = 128;
MaxSizeGuaranteed = 512;
MaxSizeNonguaranteed = 256;
MinBandwidth = 131072;
MaxBandwidth = 10000000000;
MinErrorToSend = 0.001;
MinErrorToSendNear = 0.01;
MaxCustomFileSize = 0;
```

### 6. Create Server Configuration

Create `~/arma3-server/share/arma3/this-server/config/generated_a3server.cfg`:

```
hostname = "My Arma 3 Server";
password = "";
passwordAdmin = "admin_password_here";
serverCommandPassword = "server_command_password";
maxPlayers = 32;

persistent = 1;
timeStampFormat = "short";
BattlEye = 1;

allowedFilePatching = 0;
allowedLoadFileExtensions[] = {"hpp","sqs","sqf","fsm","cpp","paa","txt","xml","inc","ext","sqm","ods","fxy","lip","csv","kb","bik","bikb","html","htm","biedi"};
allowedPreprocessFileExtensions[] = {"hpp","sqs","sqf","fsm","cpp","paa","txt","xml","inc","ext","sqm","ods","fxy","lip","csv","kb","bik","bikb","html","htm","biedi"};
allowedHTMLLoadExtensions[] = {"htm","html","xml","txt"};

class Missions {
    class Mission1 {
        template = "your_mission.Altis";
        difficulty = "veteran";
    };
};
```

### 7. Run the Server

Run the container with appropriate volume mounts:

```bash
docker run -d \
  --name arma3_server \
  -p 2302:2302/udp \
  -p 2303:2303/udp \
  -p 2304:2304/udp \
  -p 2305:2305/udp \
  -p 2306:2306/udp \
  -v ~/arma3-server/arma3:/arma3 \
  -v ~/arma3-server/share:/var/run/share \
  -e ARMA_CONFIG_JSON=/var/run/share/steam_credentials.json \
  -e LOG_LEVEL=INFO \
  arma3server:latest
```

### 8. Monitor the Server

Check logs to see the startup progress:

```bash
docker logs -f arma3_server
```

The launcher will:
1. Download Arma 3 server files via SteamCMD (first run only)
2. Download any configured Workshop mods
3. Link mods and DLCs into the server directory
4. Start the Arma 3 server and headless clients

## Configuration

### Directory Structure

The launcher expects the following directory structure:

```
Host System:
~/arma3-server/
├── arma3/                      # Main Arma 3 installation
│   ├── arma3server_x64         # Server binary (downloaded by SteamCMD)
│   ├── battleye/               # BattlEye files
│   ├── config/                 # Symlinked to this-server/config
│   ├── keys/                   # Mod signing keys
│   ├── logs/                   # Server and HC logs
│   ├── mods/                   # Active mod symlinks
│   ├── servermods/             # Active servermod symlinks
│   └── mpmissions/             # Symlinked to this-server/mpmissions
└── share/
    ├── steam_credentials.json  # Steam credentials file
    └── arma3/
        ├── server-common/      # Shared across all server instances
        │   ├── basic.cfg
        │   ├── dlcs/           # DLC directories
        │   └── mods/           # Shared mod storage
        └── this-server/        # Instance-specific files
            ├── config/
            │   ├── generated_a3server.cfg
            │   └── params.cfg
            ├── mods/           # Instance-specific mods
            ├── servermods/     # Instance-specific servermods
            └── mpmissions/     # Mission files

Container Paths:
/arma3                          → ~/arma3-server/arma3
/var/run/share                  → ~/arma3-server/share
/var/run/share/arma3/server-common
/var/run/share/arma3/this-server
```

### Steam Credentials

The launcher needs Steam credentials to download game files and Workshop mods. You can provide credentials in two ways:

#### Option 1: Credentials JSON File (Recommended)

Create `/var/run/share/steam_credentials.json` (or specify path with `ARMA_CONFIG_JSON` env var):

```json
{
  "steam_user": "your_username",
  "steam_password": "your_password"
}
```

#### Option 2: Environment Variables

Pass credentials as environment variables:

```bash
docker run ... \
  -e STEAM_USER=your_username \
  -e STEAM_PASSWORD=your_password \
  arma3server:latest
```

**Security Note**: The JSON file method is preferred as it keeps credentials out of process listings and Docker inspect output.

### Server Configuration

The launcher uses a JSON schema-based configuration system. While the README mentions `launcher/example.json`, you primarily configure the server through:

1. **Server.cfg** (`generated_a3server.cfg`): Standard Arma 3 server configuration
2. **Basic.cfg**: Network and performance settings
3. **Params.cfg** (optional): Mission parameter overrides
4. **Environment Variables**: Runtime behavior

#### Key Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ARMA_BINARY` | `/arma3/arma3server_x64` | Path to Arma 3 server binary |
| `ARMA_CONFIG_JSON` | `/var/run/share/steam_credentials.json` | Steam credentials file |
| `ARMA_ROOT` | `/arma3` | Arma 3 installation directory |
| `LOG_LEVEL` | `INFO` | Logging verbosity (DEBUG, INFO, WARNING, ERROR) |
| `LOG_JSON` | `false` | Enable JSON-formatted logs |
| `HC_START_DELAY` | `1` | Seconds between headless client starts |
| `SKIP_INSTALL` | `false` | Skip SteamCMD downloads (for testing) |
| `STEAM_USER` | - | Steam username (fallback) |
| `STEAM_PASSWORD` | - | Steam password (fallback) |

Example with custom settings:

```bash
docker run -d \
  --name arma3_server \
  -p 2302:2302/udp \
  -v ~/arma3-server/arma3:/arma3 \
  -v ~/arma3-server/share:/var/run/share \
  -e ARMA_CONFIG_JSON=/var/run/share/steam_credentials.json \
  -e LOG_LEVEL=DEBUG \
  -e LOG_JSON=true \
  -e HC_START_DELAY=2 \
  arma3server:latest
```

## Running the Server

### Starting the Server

```bash
docker run -d \
  --name arma3_server \
  -p 2302:2302/udp \
  -p 2303:2303/udp \
  -p 2304:2304/udp \
  -p 2305:2305/udp \
  -v ~/arma3-server/arma3:/arma3 \
  -v ~/arma3-server/share:/var/run/share \
  -e ARMA_CONFIG_JSON=/var/run/share/steam_credentials.json \
  arma3server:latest
```

### Stopping the Server

The launcher handles SIGINT (Ctrl+C) gracefully:

```bash
docker stop arma3_server
```

### Restarting the Server

```bash
docker restart arma3_server
```

### Removing the Container

```bash
docker stop arma3_server
docker rm arma3_server
```

### Viewing Logs

Real-time logs:
```bash
docker logs -f arma3_server
```

Log files are also written to `/arma3/logs/` (on host: `~/arma3-server/arma3/logs/`):
- `server.log` - Main server output
- `server_warnings.log` - Filtered warnings
- `hc_0.log`, `hc_1.log`, etc. - Headless client logs

## Managing Mods and DLCs

### Workshop Mods

The launcher can automatically download Workshop mods via SteamCMD. Configure mods in your server configuration or use the JSON schema system described in the launcher.

To add Workshop mods:
1. Find the Workshop item ID (from the Steam Workshop URL)
2. Add to your configuration
3. The launcher will download and link mods automatically

### DLC Management

DLCs are configured in the server schema JSON. The launcher supports:

- **Contact DLC**: Special handling (mutually exclusive with other paid DLCs)
- **Creator DLCs**: CSLA, Global Mobilization, S.O.G Prairie Fire, Western Sahara, Spearhead 1944, Reaction Forces, Expeditionary Forces
- **Expansion DLCs**: Apex, Helicopters, Jets, Karts, Laws of War, Marksmen, Tanks, Tac-Ops

**Important**: If using Contact DLC, no other paid DLCs can be active. Other DLCs switch to creator-branch automatically.

### Manual Mod Installation

For mods not on Workshop:

1. Place mod folder (e.g., `@ModName`) in:
   - `~/arma3-server/share/arma3/this-server/mods/` (instance-specific)
   - OR `~/arma3-server/share/arma3/server-common/mods/` (shared)

2. Ensure proper structure:
   ```
   @ModName/
   ├── addons/
   │   └── *.pbo files
   └── keys/
       └── *.bikey files
   ```

3. Restart the server - mods will be automatically linked

## Headless Clients

Headless clients (HCs) reduce server load by offloading AI calculations. The launcher can automatically spawn and manage multiple HCs.

### Configuration

Set the number of headless clients in your server configuration (typically via JSON schema or environment):

```json
{
  "headless_clients": 2
}
```

### Behavior

- Each HC starts with a delay (`HC_START_DELAY`) to avoid connection races
- Each HC gets its own log file: `hc_0.log`, `hc_1.log`, etc.
- HCs connect to localhost (server must be configured to allow HC connections)

### Server-Side HC Configuration

Add to your `generated_a3server.cfg`:

```
headlessClients[] = {"127.0.0.1"};
localClient[] = {"127.0.0.1"};
```

### Adjusting Start Delay

If HCs fail to connect or cause rate limiting issues:

```bash
docker run ... -e HC_START_DELAY=5 arma3server:latest
```

## Troubleshooting

### Issue: "result 26 (Request revoked)" or SteamCMD Failures

**Symptoms**: SteamCMD fails with "result 26" or "Request revoked" errors.

**Causes**:
- Steam rate limiting
- Concurrent SteamCMD sessions
- Steam Guard / 2FA issues
- System clock skew

**Solutions**:
1. Ensure only one SteamCMD session runs at a time
2. Increase `HC_START_DELAY` to 2-5 seconds
3. Disable Steam Guard 2FA on the account (or handle it manually)
4. Check system clock: `date` - ensure it's accurate
5. Wait and retry - rate limits typically reset after a few minutes
6. Use the launcher's built-in retry mechanism (automatic with backoff)

### Issue: Mods Not Loading

**Symptoms**: Server starts but mods aren't active.

**Checks**:
1. Verify mod directory structure includes `@ModName/addons/*.pbo`
2. Check logs for mod linking errors: `docker logs arma3_server`
3. Ensure Workshop download completed successfully
4. Verify symlinks: `ls -la /arma3/mods` (inside container)
5. Check file permissions - container needs read access

### Issue: Server Won't Start

**Symptoms**: Container exits immediately or server process fails.

**Checks**:
1. Check logs: `docker logs arma3_server`
2. Verify server configuration syntax in `.cfg` files
3. Ensure missions exist in `mpmissions/` directory
4. Check file permissions on mounted volumes
5. Verify Arma 3 server files were downloaded: `ls /arma3/arma3server_x64`
6. Ensure required ports aren't already in use: `netstat -tulpn | grep 2302`

### Issue: Headless Clients Not Connecting

**Symptoms**: Server starts but HCs timeout or fail to connect.

**Checks**:
1. Verify `headlessClients[]` and `localClient[]` in server config
2. Check HC logs: `~/arma3-server/arma3/logs/hc_*.log`
3. Increase `HC_START_DELAY`: `-e HC_START_DELAY=5`
4. Ensure server is fully started before HCs attempt connection
5. Check firewall rules (though localhost shouldn't need this)

### Issue: High Memory Usage

**Symptoms**: Container uses excessive RAM.

**Solutions**:
1. Reduce number of headless clients
2. Limit active mods (disable unused mods)
3. Optimize mission complexity
4. Set Docker memory limits:
   ```bash
   docker run --memory=8g --memory-swap=8g ... arma3server:latest
   ```

### Issue: Permission Denied Errors

**Symptoms**: Errors accessing files or creating directories.

**Solution**:
```bash
# On host, ensure proper ownership
sudo chown -R $(id -u):$(id -g) ~/arma3-server/
chmod -R 755 ~/arma3-server/
```

### Getting More Debug Information

Enable debug logging:

```bash
docker run ... -e LOG_LEVEL=DEBUG arma3server:latest
```

Or use JSON logs for structured output:

```bash
docker run ... -e LOG_LEVEL=DEBUG -e LOG_JSON=true arma3server:latest
```

## Advanced Configuration

### Using Docker Compose

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  arma3server:
    image: arma3server:latest
    container_name: arma3_server
    ports:
      - "2302:2302/udp"
      - "2303:2303/udp"
      - "2304:2304/udp"
      - "2305:2305/udp"
      - "2306:2306/udp"
    volumes:
      - ./arma3:/arma3
      - ./share:/var/run/share
    environment:
      - ARMA_CONFIG_JSON=/var/run/share/steam_credentials.json
      - LOG_LEVEL=INFO
      - HC_START_DELAY=2
    restart: unless-stopped
```

Run with:
```bash
docker-compose up -d
```

### Running Without Docker

The launcher can run directly on a Linux host:

1. Install dependencies:
   ```bash
   apt-get install python3 python3-pip steamcmd
   pip3 install json5 jsonschema
   ```

2. Set up directory structure as described above

3. Run the launcher:
   ```bash
   export ARMA_ROOT=/path/to/arma3
   export ARMA_CONFIG_JSON=/path/to/steam_credentials.json
   python3 launcher/launcher.py
   ```

### Custom SteamCMD Parameters

The `steam.py` module handles SteamCMD interactions with automatic retry and backoff. For advanced customization, you may need to modify the launcher code.

### Multiple Server Instances

To run multiple server instances:

1. Create separate directories for each instance
2. Use different ports for each container
3. Use different `this-server` directories
4. Optionally share `server-common` for DLCs and base mods

Example:
```bash
# Server 1
docker run -d --name arma3_server1 -p 2302:2302/udp \
  -v ~/arma3-server1/arma3:/arma3 \
  -v ~/arma3-server1/share/this-server:/var/run/share/arma3/this-server \
  -v ~/arma3-common/share/server-common:/var/run/share/arma3/server-common \
  arma3server:latest

# Server 2
docker run -d --name arma3_server2 -p 2312:2302/udp \
  -v ~/arma3-server2/arma3:/arma3 \
  -v ~/arma3-server2/share/this-server:/var/run/share/arma3/this-server \
  -v ~/arma3-common/share/server-common:/var/run/share/arma3/server-common \
  arma3server:latest
```

### Performance Tuning

#### Server Configuration

In `generated_a3server.cfg`:
```
MaxMsgSend = 256;           // Higher for better sync
MaxSizeGuaranteed = 512;
MaxSizeNonguaranteed = 256;
MinBandwidth = 1048576;     // 1 Mbps minimum
MaxBandwidth = 2147483647;  // ~2 Gbps maximum
```

#### Resource Limits

```bash
docker run ... \
  --cpus="4" \
  --memory="8g" \
  arma3server:latest
```

### Log Management

Logs grow over time. Implement rotation:

```bash
# Host cron job
0 0 * * * find ~/arma3-server/arma3/logs -name "*.log" -mtime +7 -delete
```

Or use Docker logging drivers:

```bash
docker run ... \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  arma3server:latest
```

## Additional Resources

- **Arma 3 Server Documentation**: https://community.bistudio.com/wiki/Arma_3_Dedicated_Server
- **SteamCMD Wiki**: https://developer.valvesoftware.com/wiki/SteamCMD
- **Docker Documentation**: https://docs.docker.com/
- **Project Repository**: https://github.com/HendrikTank/Arma3Server_v2
- **Issue Tracker**: https://github.com/HendrikTank/Arma3Server_v2/issues

## Getting Help

If you encounter issues:

1. Check the [Troubleshooting](#troubleshooting) section above
2. Review the logs: `docker logs arma3_server`
3. Enable debug logging: `-e LOG_LEVEL=DEBUG`
4. Open an issue on GitHub with:
   - Your Docker version
   - The full error message
   - Relevant log excerpts
   - Your configuration (redact credentials)

## Contributing

Contributions are welcome! See the main [README.md](README.md) for development guidelines.
