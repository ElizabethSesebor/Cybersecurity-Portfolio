*Disclaimer: This is an educational project. The intelligence comes from OpenCTI, the MITRE ATT&CK dataset and the AlienVault OTX community feed, and reflects what those sources reported at the time. Desert Oil is the company scenario used for the exercise. Do not expose this lab to the internet, and never commit credentials or API keys.*

# Threat Intelligence Lab: OpenCTI + AlienVault OTX

A hands-on threat intelligence project. I deployed OpenCTI on Ubuntu with Docker, connected it to a live AlienVault OTX feed, and used it to profile the threats targeting the Saudi Arabian oil and gas sector.

This repository is the technical guide: how to build the lab, the configuration I used, the problems I hit and how I fixed them, and a summary of the analysis. The full write-up is on Medium: [https://medium.com/@elizabethsesebor169]



<img width="1280" height="646" alt="IMG-20260930-WA0038" src="https://github.com/user-attachments/assets/b5a3cc2c-ea84-4e5d-981e-e6221f5fdf1d" />




| Category | Details |
|---|---|
| Sector | Oil and Gas |
| Focus country | Saudi Arabia — headquarters of Desert Oil |
| Tools | OpenCTI, Ubuntu, Docker, AlienVault OTX, Sublime Text |
| Frameworks | Diamond Model, Cyber Kill Chain, MITRE ATT&CK |

## Table of contents

1. Project goals
2. Stack and prerequisites
3. Deployment guide
4. Connecting AlienVault OTX
5. Troubleshooting
6. Safe shutdown and restart
7. Analysis summary
8. Key findings
9. Recommendations
10. Limitations


### 1. Project goals

For the oil and gas sector, with Saudi Arabia as the focus country (the headquarters of Desert Oil, the company used for this exercise):

Identify the top 2 threats currently targeting the country.
Identify the 3 most targeted victims and, for each, describe the top threat, break it down with the Diamond Model (Adversary, Capability, Infrastructure, Victim), and map it to the Cyber Kill Chain.
Use OpenCTI to find the most recent politically driven threat actor with high-profile activity, and analyse its motivation, targets, tactics, tools, campaigns and impact.
Draw out key findings and recommendations.

### 2. Stack and prerequisites
Component	Purpose
Ubuntu (VM)	Host machine
Docker + Compose plugin	Runs OpenCTI and its dependencies as containers
OpenCTI	Threat intelligence platform
OpenSearch, Redis, RabbitMQ, MinIO	OpenCTI dependencies
AlienVault OTX connector	Imports live threat pulses and indicators
Sublime Text	Editing config files (any editor works)

Resources: I gave the VM 12 GB RAM and 100 GB disk. Docker and OpenCTI consume a lot of both.

Accounts: a free AlienVault OTX account and API key.

### 3. Deployment guide
#### 3.1 Install Docker Engine and the Compose plugin
```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
```

#### Docker's official GPG key
```
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

#### Docker repository
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### Start on boot and run without sudo
```
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
newgrp docker
```
If newgrp is not found: 
```
sudo apt-get install -y util-linux-extra, then newgrp docker.
```

#### 3.2 Install Git (if not already installed)
```bash
sudo apt-get install -y git
```

Install Sublime Text
```
wget -qO - https://download.sublimetext.com/sublimehq-pub.gpg | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/sublimehq-archive.gpg > /dev/null
echo "deb https://download.sublimetext.com/ apt/stable/" | sudo tee /etc/apt/sources.list.d/sublime-text.list
sudo apt-get update
sudo apt-get install -y sublime-text
```

#### 3.3 Raise the virtual memory limit

OpenSearch needs a higher memory-map limit than the Ubuntu default.
```bash
sudo sysctl -w vm.max_map_count=1048576
echo "vm.max_map_count=1048576" | sudo tee -a /etc/sysctl.conf
```

#### 3.4 Clone OpenCTI's Docker repository
```bash
git clone https://github.com/OpenCTI-Platform/docker.git opencti
cd opencti
cp .env.sample .env
```

#### 3.5 Generate keys
UUIDs (admin token, connector IDs, healthcheck key)
```
cat /proc/sys/kernel/random/uuid
```
Or:
```
uuidgen
```

#### Encryption key / secret
For 32-byte hex
```
openssl rand -hex 32                 
```
For 32-byte Base64
```
openssl rand -base64 32              # 
```

#### 3.6 Configure .env

Replace every ChangeMe placeholder. Use strong, long passwords: weak or short ones are a common reason containers fail their health checks.
```
env
OPENCTI_ADMIN_EMAIL=admin@opencti.io
OPENCTI_ADMIN_PASSWORD=<a strong password>
OPENCTI_ADMIN_TOKEN=<UUID generated above>
MINIO_ROOT_USER=opencti
MINIO_ROOT_PASSWORD=<a strong password>
```

Never commit your real .env file. Add .env to .gitignore and commit a sanitised .env.example instead.

#### 3.7 Launch the stack

I started services one at a time on my constrained VM, dependencies first:
```
bash
docker compose up -d redis
docker compose up -d rabbitmq
docker compose up -d minio
docker compose up -d opensearch      # or elasticsearch, depending on the compose file
docker compose up -d opencti
```

Then bring up everything and confirm all services are healthy:

```
bash
docker compose up -d
docker compose ps
```

<img width="363" height="263" alt="Picture2" src="https://github.com/user-attachments/assets/eb714d54-a967-4681-9287-4b7f5c195656" />


#### 3.8 Open the dashboard

Browse to [http://localhost:8080] and log in with the admin email and password from .env.

### 4. Connecting AlienVault OTX

#### 4.1 Get an API key
Create a free account at [otx.alienvault.com]
Click your profile icon (top right) and choose API Integration.
Copy the 64-character secret API key.

#### 4.2 Add the connector variables to .env
```
env
CONNECTOR_ALIENVAULT_ID=<new UUID>
ALIENVAULT_API_KEY=<your OTX API key>
```

Every connector needs its own unique UUID v4 for CONNECTOR_ID.

#### 4.3 Define the connector in docker-compose.yml

This is the final working version, including the DNS fix (Section 5.3), the patched file (Section 5.4) and memory limits (Section 5.2):

```
yaml
connector-alienvault:
  image: opencti/connector-alienvault:latest
  dns:
    - 8.8.8.8
    - 1.1.1.1
  volumes:
    - ./alienvault_utils.py:/opt/opencti-connector-alienvault/alienvault/utils/__init__.py
  environment:
    - OPENCTI_URL=http://opencti:8080
    - OPENCTI_TOKEN=${OPENCTI_ADMIN_TOKEN}
    - CONNECTOR_ID=${CONNECTOR_ALIENVAULT_ID}
    - CONNECTOR_TYPE=EXTERNAL_IMPORT
    - CONNECTOR_NAME=AlienVault
    - CONNECTOR_SCOPE=alienvault
    - CONNECTOR_CONFIDENCE_LEVEL=15
    - CONNECTOR_UPDATE_EXISTING_DATA=true
    - CONNECTOR_LOG_LEVEL=info
    - ALIENVAULT_API_KEY=${ALIENVAULT_API_KEY}
    - ALIENVAULT_CREATE_INDICATORS=true
    - ALIENVAULT_INTERVAL=60
  deploy:
    resources:
      limits:
        memory: 512M
  restart: always
  depends_on:
    - opencti
```

#### 4.4 Start and verify
```
bash
docker compose up -d connector-alienvault
docker compose logs --tail 30 -f connector-alienvault
```

A healthy start logs, in order: 
- Connector registered with ID..., 
- Starting AlienVault connector..., 
- Running pulse importer..., 
- Fetching subscribed pulses....

The Integrations page in OpenCTI should then show AlienVault, MITRE ATT&CK and OpenCTI Datasets as active.

### 5. Troubleshooting

Problems I hit, in the order I hit them.

####	Symptom	Cause	Fix
- read: connection reset by peer during image pulls	Network drops on NAT-mode VMs	5.1
- Freezes, out-of-memory crashes	Default memory use too high for 12 GB	5.2
- NameResolutionError: otx.alienvault.com	Container cannot resolve DNS	5.3
- ValueError: unconverted data remains: Z	Connector date-parsing bug	5.4
- A service shows Unhealthy	Usually a mistake in .env	5.5

#### 5.1 Image pulls failing

Pull images in groups instead of all at once:

```
bash
cd ~/opencti
docker compose pull redis rabbitmq
docker compose pull minio
docker compose pull opensearch        # large, allow 2 to 5 minutes
docker compose pull opencti
```

If it persists, fix the VM's DNS:

```
bash
sudo bash -c 'echo "nameserver 8.8.8.8" > /etc/resolv.conf'
sudo bash -c 'echo "nameserver 1.1.1.1" >> /etc/resolv.conf'
sudo systemctl restart docker
```

If it still persists, disable offloading and lower Docker's MTU to avoid packet fragmentation:

```
bash
sudo ethtool -K $(ip route show default | awk '{print $5}') rx off tx off tso off gso off gro off 2>/dev/null || true

sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<EOF
{
  "mtu": 1400
}
EOF
```

```
sudo systemctl restart NetworkManager 2>/dev/null || sudo systemctl restart networking
sudo systemctl restart docker
cd ~/opencti && docker compose pull
```

#### 5.2 Out-of-memory crashes

Cap memory and heap sizes in docker-compose.yml:

```
yaml
services:
  opensearch:
    image: opensearchproject/opensearch:latest
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g"     # Java heap 1 GB
    deploy:
      resources:
        limits:
          memory: 2048M

  opencti:
    image: opencti/platform:latest
    environment:
      - NODE_OPTIONS=--max-old-space-size=2048   # Node.js 2 GB
    deploy:
      resources:
        limits:
          memory: 2500M

  connector-alienvault:
    image: opencti/connector-alienvault:latest
    deploy:
      resources:
        limits:
          memory: 512M
```

#### 5.3 DNS failures

The connector container could not resolve otx.alienvault.com. Add explicit DNS servers to the connector service (shown in Section 4.3):
```
yaml
    dns:
      - 8.8.8.8
      - 1.1.1.1
```

Confirm the host itself resolves the endpoint:

```
bash
ping -c 3 otx.alienvault.com
```

#### 5.4 Date-parsing bug

The connector's default date strings end in a trailing Z (for example 2020-05-01T00:00:00Z), which makes datetime.strptime raise ValueError: unconverted data remains: Z.

<img width="520" height="303" alt="Picture3" src="https://github.com/user-attachments/assets/565dd3c7-98a6-4964-8089-c2d9d1fdf471" />


Fix it by patching the utility file locally and mounting it over the original so it survives restarts.

##### Step 1. Extract the original from the image:

```
bash
cd ~/opencti
docker run --rm --entrypoint cat opencti/connector-alienvault:latest \
  /opt/opencti-connector-alienvault/alienvault/utils/__init__.py > ./alienvault_utils.py
```

##### Step 2. Patch it to strip the trailing Z:
```
bash
python3 -c '
file_path = "alienvault_utils.py"
with open(file_path, "r") as f:
    content = f.read()

new_content = content.replace(
    "def iso_datetime_str_to_datetime(string):",
    "def iso_datetime_str_to_datetime(string):\n    string = string.rstrip(\"Z\")"
)

with open(file_path, "w") as f:
    f.write(new_content)

print("Local alienvault_utils.py patched successfully!")
'
```

##### Step 3. Mount it using the volumes: entry in Section 4.3, then recreate the container:
```
bash
docker compose up -d connector-alienvault
docker compose logs --tail 30 -f connector-alienvault
```
This patch is fragile because it depends on the exact function line in the image. Pin the connector image version instead of :latest so an upgrade does not silently break it.

#### 5.5 Unhealthy containers

Read the logs of the failing service first:
```
bash
docker compose logs xtm-one      # replace with the failing service
```

Most of the time the cause is the .env file: a stray space, a missing character or symbol, or a password that is too short. After fixing it:

```
bash
docker compose down
docker compose up -d xtm-one     # or the dependency you fixed
docker compose ps xtm-one
docker compose up -d             # then the full stack
docker compose ps                # all should show Healthy
```

Reference block for the xtm-one variables (placeholders only):
```
env
XTM_ONE_HOST=localhost
XTM_ONE_PORT=8090
XTM_ONE_EXTERNAL_SCHEME=http
XTM_ONE_ADMIN_EMAIL=<same as OPENCTI_ADMIN_EMAIL>
XTM_ONE_ADMIN_PASSWORD=<a strong password>
XTM_ONE_SECRET_KEY=<32-byte generated key>
XTM_ONE_POSTGRES_USER=xtmone
XTM_ONE_POSTGRES_PASSWORD=<a strong password>
XTM_ONE_S3_BUCKET=xtm-one-files
XTM_ONE_ENTERPRISE_LICENSE=
XTM_ONE_LOG_LEVEL=info
XTM_ONE_LOG_FORMAT=json
```

### 6. Safe Shutdown and Restart

State lives in Docker volumes, so your data, indicators, API keys and the patched file survive as long as you never run docker compose down -v (the -v flag deletes volumes). 
Stop containers gracefully before powering off so OpenSearch, PostgreSQL and Redis can flush to disk.
```
bash
cd ~/opencti
docker compose stop
sudo shutdown -h now          # or: sudo reboot
```
To log in again
```
cd ~/opencti
docker compose up -d
```

### 7. Analysis Summary

The full narrative with screenshots is in the Medium article: [https://medium.com/@elizabethsesebor169]

#### 7.1 Top 2 threats targeting Saudi Arabia

An OpenCTI search for Saudi Arabia returned five groups: APT33, BITTER, CopyKittens, HEXANE and MuddyWater. I ranked them on oil and gas focus, Saudi focus, state backing and recent activity.


<img width="580" height="243" alt="Picture4" src="https://github.com/user-attachments/assets/edb393a5-250c-49b1-baa6-9eb2184c1982" />


| Threat | Backing | Why it is a priority |
|---|---|---|
| MuddyWater (G0069) | Iran's MOIS | Targets oil and gas, telecoms, government, and defence in Saudi Arabia and the UAE since at least 2017 |
| HEXANE (G1001) | Iranian-linked | Targets oil and gas, telecoms, and aviation in Saudi Arabia, Kuwait, Morocco, Tunisia, and Israel since at least 2017 |


<img width="1283" height="529" alt="Picture5" src="https://github.com/user-attachments/assets/1e2ab20e-c31b-4f6f-b391-b9162dcb21b2" />


#### MuddyWater

- Entry: emails from hijacked real accounts; internet-facing flaws such as Log4Shell and ProxyShell.
- Inside: Mimikatz and LaZagne for credentials; Remote Desktop and PowerShell to move.
- Persistence and exfiltration: hidden PowerShell scripts; Wasabi and OneDrive.
- Newer tactic: Starlink used for control in late 2025 to early 2026.
- Scale: 68 ATT&CK techniques, 11 malware families (e.g. RustyWater, SHARPSTATS, PowGoop, POWERSTATS), 10 tools (e.g. Koadic, Empire, CrackMapExec, Rclone).
- Campaigns and aliases: "Outer Space", "Juicy Mix"; Earth Vetala, MERCURY, Static Kitten, Seedworm, TEMP.Zagros, Mango Sandstorm, TA450, MuddyKrill.


<img width="560" height="230" alt="Picture9" src="https://github.com/user-attachments/assets/8976aa06-b402-49d4-bc9f-f0496a35db48" />

#### HEXANE

- Entry: spearphishing from compromised internal accounts; fake LinkedIn job offers.
- Persistence: Registry Run Keys, scheduled tasks, WMI event subscriptions.
- Exfiltration: cloud storage such as OneDrive.
- Scale: 36 ATT&CK techniques; malware Kevin, DanBot, DnsSystem, Shark, Milan; tools PoshC2, Mimikatz, Empire, BITSAdmin.
- Hunt for: kl.ps1 (PowerShell keylogger) and MicrosoftUpdator.vbs.
- Aliases: Lyceum, Siamesekitten, Spirlin.


#### 7.2 Three targeted sectors and their lead threats

| Sector | Lead threat | What it is | Kill Chain alignment |
|---|---|---|---|
| Government | CL-CRI-1171 | Pay-per-install cybercrime campaign, undetected for 2+ years; OfferLoader delivers Insomnia RAT, ARKTunnel, Docro Hijacker; YouTube lures and SEO poisoning | Delivery, Exploitation and Installation, Command and Control, Session/Cookie Theft |
| Finance | Cerberus | Rented mobile banking trojan; actor alias DukeEugene | Installation, Command and Control, Actions on Objectives |
| Technology | PolinRider | Supply-chain compromise with dead drop resolvers and cloud exfiltration | Delivery, Exploitation and Installation, Command and Control, Actions on Objectives |


#### Diamond Model summary

| Axis | CL-CRI-1171 | Cerberus | PolinRider |
|---|---|---|---|
| Adversary | Not recorded | DukeEugene | Not recorded |
| Capability | OfferLoader, Socks5Systemz, Docro Hijacker, GCleaner, ARKTunnel, and Insomnia RAT; phishing, cookie theft, and defence impairment | Banking trojan; input capture, credential harvesting, anti-analysis, and SMS interception | PolinRider; supply-chain compromise, DLL search-order hijacking, and browser/token theft |
| Infrastructure | Not recorded; delivery via pay-per-install, YouTube, and SEO poisoning | Not recorded | Dead-drop resolvers and web protocols |
| Victim | Government, Energy | Finance | Technology |


| Cerberus Diamond Model in OpenCTI | PolinRider Diamond Model in OpenCTI |
|---|---|
<img width="560" height="307" alt="Picture7" src="https://github.com/user-attachments/assets/6cfcebfd-aecb-46ff-9dc5-870ca276164e" /> | (<img width="560" height="271" alt="Picture8" src="https://github.com/user-attachments/assets/512f9641-bf2d-4abd-90c2-567aecac2ef4" />

Techniques seen across the three (full ATT&CK tables are in the Medium article and the project document): Drive-by Compromise (T1189), Phishing (T1566), Steal Web Session Cookie (T1539), Impair Defenses (T1562.001), Compromise Software Supply Chain (T1195), Exfiltration to Cloud Storage (T1567.002), Web Protocols (T1071.001), Dead Drop Resolver (T1102.001).

*OpenCTI lists techniques under MITRE ATT&CK tactics. The Kill Chain alignment above is my own mapping of those techniques to the seven Cyber Kill Chain stages*

#### 7.3 Politically driven actor: CyberAv3ngers
	
| Category | Details |
|---|---|
| Assessed origin | Iran, linked to the IRGC (MITRE G1027) |
| Type | State-sponsored group operating under a hacktivist identity |
| Motivation | Political and ideological, not financial |
| Targets | Critical-infrastructure OT: energy, oil and gas, water, and fuel |
| Method | Scans for exposed industrial devices; exploits unpatched flaws and default passwords, such as `1111` |
| Known activity | Late 2023 attacks on Israeli-made Unitronics controllers at US water utilities, including Aliquippa, Pennsylvania; IOCONTROL malware targeting IoT/OT and fuel-management systems; US Treasury sanctions on six Iranian officials |


<img width="1283" height="465" alt="Picture11" src="https://github.com/user-attachments/assets/e116b6e6-5ee1-4bde-954c-8146eef2dfd4" />



Relevance: Desert Oil's operational technology estate is exactly what this group targets, and its attacks need no advanced skills, only exposed devices.

### 8. Key Findings

1. Iranian state-linked espionage is the main threat to Saudi oil and gas (MuddyWater, HEXANE), persistent since at least 2017.
2. Initial access is people and exposed systems, not novel exploits: hijacked-account phishing, fake LinkedIn offers, unpatched internet-facing servers.
3. Attackers use legitimate tools and trusted services (Mimikatz, PowerShell, RDP, OneDrive, Wasabi), so detection must be behaviour-based.
4. Tradecraft evolves quickly (Starlink for control), so static indicators age fast.
5. Commodity crime reaches high-value targets (CL-CRI-1171, Cerberus).
6. Credential and session theft is the common thread across every threat analysed.
7. OT is the most consequential exposure (CyberAv3ngers, IOCONTROL).
8. Intelligence gaps remain: missing adversary and infrastructure data for some threats; early kill chain stages not visible.


### 9. Recommendations

| Priority | Action | Addresses |
|---|---|---|
| Immediate | Find and remove internet exposure of OT/ICS devices | CyberAv3ngers |
| Immediate | Eliminate default and weak passwords on industrial and network devices | CyberAv3ngers |
| Immediate | Patch internet-facing systems first, including Log4Shell and ProxyShell | MuddyWater |
| Immediate | Enforce MFA on email, VPN, remote access, and administrator accounts | MuddyWater, HEXANE, PolinRider |
| Immediate | Load known indicators into detection tools, including domains, `kl.ps1`, and `MicrosoftUpdator.vbs` | MuddyWater, HEXANE |
| 1 to 3 months | Harden email defences and train staff to recognize fake recruiter approaches | MuddyWater, HEXANE |
| 1 to 3 months | Protect credentials using LSASS protection and Credential Guard; restrict RDP and enable PowerShell logging | MuddyWater, HEXANE |
| 1 to 3 months | Alert on Run Keys, scheduled tasks, and WMI subscriptions | MuddyWater, HEXANE |
| 1 to 3 months | Control outbound data and alert on Rclone and unapproved cloud uploads | MuddyWater, HEXANE, PolinRider |
| 1 to 3 months | Use application allow-listing and block unapproved installers and browser extensions | CL-CRI-1171 |
| 1 to 3 months | Secure the software supply chain, scan for secrets, and rotate tokens | PolinRider |
| 1 to 3 months | Deploy MDM for finance staff phones and block sideloading | Cerberus |
| 3 to 12 months | Segment IT from OT and monitor OT passively | CyberAv3ngers, MuddyWater |
| 3 to 12 months | Rehearse OT incident response, including manual-operation fallback | CyberAv3ngers |
| 3 to 12 months | Implement threat-informed detection mapped to recurring ATT&CK techniques | All |
| 3 to 12 months | Operationalize OpenCTI: feed the SIEM, review monthly, and close attribution gaps | All |


### 10. Limitations
- Victims were chosen by sector from the OpenCTI dashboard, not by individual organisation.
- I did not independently verify that every campaign falls within the last three months.
- OpenCTI recorded no adversary for CL-CRI-1171 and PolinRider, and no infrastructure for CL-CRI-1171 and Cerberus.
- Early kill chain stages (reconnaissance, weaponization) are not visible in the data.
- The lab uses :latest image tags and a locally patched connector file. Pin versions and use a secrets store for anything beyond a lab.


### References
- OpenCTI and OpenCTI Docker repository
- AlienVault OTX
- MITRE ATT&CK: MuddyWater (G0069), HEXANE (G1001), CyberAv3ngers (G1027)
- Docker documentation
