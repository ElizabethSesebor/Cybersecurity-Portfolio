# Endpoint Forensics - Involving a Trojan

## Objective


The EDR Lab project aimed to analyze a memory dump of an affected Windows workstation to uncover the details of the malicious activity, assess the malware’s behavior, and determine the extent of the compromise.The primary focus was to investigate running processes, trace network activity, locate suspicious files, and uncover mechanisms the malware used to establish persistence. And this was done by applying various Volatility3 plugins to extract critical information and build a comprehensive understanding of the attack.

## Skills Learned

- Advanced understanding and practical application of Endpoint Forensic.
- Proficiency in investigating network connections, and payload delivery.
- Adept identification and response to malware threats and its processes.
- Enhanced ability to detect persistent mechanisms.
- Development of critical thinking and problem-solving skills in cybersecurity.

### Tools Used

- Volatility3 for memory forensics.

## Steps
First is to enumerate the running processes within the captured memory image. I used the "pstree" plugin, executed it with the following commands in the below picture. "vol.py -f ~/Desktop/Start\ here/Artifacts/Windows\ 7\ x64-Snapshot4.vmem windows.pstree.PsTree"

<img width="1917" height="848" alt="Screenshot 2026-02-27 223916" src="https://github.com/user-attachments/assets/0267f716-7ee5-4777-9f43-328c5b8a4184" />
img1

The export command is used to add the path of Volatility3 to the PATH environment variable so that I can directly reference vol.py without typing the full path.

Next is to identify the name of the parent process that triggered the malicious behavior. From the output of the above command in img1, attention was drawn to the process "lsass.exe" with PID 508 as seen in the below picture. 

<img width="1905" height="614" alt="Screenshot 2026-02-27 223946" src="https://github.com/user-attachments/assets/1a8c98c7-728b-4f46-9910-b1185ea538a5" />
img2

The observed anomaly was a kind name of that process which has an extra 's' in its name posing as a masquarde process designed to evade detection. The malicious process is "lssass.exe" with PID 2748. It also have a linked child process "rundll32.exe (PID 3064)" which is frequently used to execute malicious code via DLL files.

Now we need to know the exact location of the process within the file system on the workstation. This helps to know its origin, whether it is inside a trusted directory or suspicious location such as temporary folders often used by malware. I used the "windows.cmdline" plugin in the Volatility3 framework. "vol.py -f ~/Desktop/Start\ here/Artifacts/Windows\ 7\ x64-Snapshot4.vmem windows.cmdline --pid 2748"

<img width="1917" height="342" alt="Screenshot 2026-02-27 224347" src="https://github.com/user-attachments/assets/0ff7cc86-47b5-4a7a-861f-f88e3ce1c634" />
img3

The above command and output revealed that the executable is in the Temp directory further raises red flags, as this location is commonly exploited by malware to drop payloads or store transient files needed for execution. Given this evidence, the executable located at C:\Users\0XSH3R~1\AppData\Local\Temp\925e7e99C5\lssass.exe is highly suspicious. 

Next in the step is to identify the Command and Control (C2) Server IP that the process interacts with. I used the plugin "windows.netscan" in the Volatility3 framework to analyze the network connections from the memory dump. It helped me to see the active and close network connections including IP addresses, ports amd associated processes. "vol.py -f ~/Desktop/Start\ here/Artifacts/Windows\ 7\ x64-Snapshot4.vmem windows.netscan | grep 2748"

<img width="1904" height="163" alt="Screenshot 2026-02-27 224846" src="https://github.com/user-attachments/assets/22f221d8-7f01-4b6c-89b8-cdbefd2f2858" />
img4

As shown in the above imag4, the IP address 41.75.84.12 over port 80 which is commonly used for HTTP traffic is being associated with the masqurade process lssass.exe. The use of port 80 further suggests an attempt to disguise malicious traffic as normal web communication, exploiting a common protocol to avoid detection by firewalls and intrusion detection systems.

Now that we know its connections, I probed further to know how many trojan files the malware is trying to bring onto the compromised workstation. I did this by utilizing the "windows.memap" plugin in the Volatility3 with the commands as follows "vol.py -f ~/Desktop/Start\ here/Artifacts/Windows\ 7\ x64-Snapshot4.vmem windows.memmap --pid 2748 --dump". This command helped me to generate a memory dump file, named "pid.2748.dmp", which can be parsed for artifacts. Once the dump was created, I used the "strings" utility to extract readable text from the memory file with this command "strings pid.2748.dmp | grep "GET /". This command searches for HTTP GET requests within the memory dump, which are often indicative of attempts to fetch remote resources or establish communication with external servers. The below image shows the outputs.

<img width="1928" height="875" alt="Screenshot 2026-02-27 225611" src="https://github.com/user-attachments/assets/3b69af03-d6cd-47ec-9917-c70a8dd5dcd9" />
img5

The HTTP requests showed that the process attempted to retrieve DLL files, potentially as part of its payload delivery or plugin-based modular structure, which aligns with behaviors seen in Trojan malware. Such DLL files might be used to extend functionality, escalate privileges, or exfiltrate sensitive data.

I decided to also know the full path of the file downloaded and used by the malware in its malicious activities. Here i used the "windows.filescan". This plugin scans memory for file objects and lists their metadata, including file paths, which can be cross-referenced against previously identified artifacts. The command executed for this purpose is: "vol.py -f ~/Desktop/Start\ here/Artifacts/Windows\ 7\ x64-Snapshot4.vmem 
windows.filescan | grep -E "cred64|clip64". This command specifically filters results for filenames containing "cred64" and "clip64," as these were identified earlier in HTTP GET requests made by the suspicious process.

<img width="1922" height="871" alt="Screenshot 2026-02-27 230222" src="https://github.com/user-attachments/assets/fe2f673f-12f8-4664-943f-3d729db0f28d" />
img6

The output showed this file path: '\Users\0xSh3rl0ck\AppData\Roaming\116711e5a2ab05\clip64.dll' The clip64.dll file from the previous filter was stored in the user's AppData Roaming directory under a subfolder named 116711e5a2ab05. The use of the AppData directory is significant because it is a common location exploited by malware to store payloads or configuration files.

Furthermore, if you see in the above img6, i also checked for where else the malware might be consistenly present. And that showed that it is also present in the \Windows\System32\Tasks\ directory. Files stored in the Tasks directory are typically associated with Windows Task Scheduler, a legitimate tool used to automate tasks. Malware often abuses scheduled tasks to establish persistence by creating tasks that execute malicious files at predefined intervals or system events.

The presence of lssass.exe in the Tasks directory implies that the malware likely registered itself as a scheduled task. This approach allows it to automatically restart upon reboot or at scheduled times, ensuring continuous execution. Task Scheduler persistence is particularly stealthy because it leverages a trusted Windows component, making it less likely to trigger alarms in security monitoring tools.



