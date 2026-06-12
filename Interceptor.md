Interceptor — TryHackMe Write-up

<img width="1400" height="299" alt="1_r7jnIqXT8WuWFKytXVi6KA" src="https://github.com/user-attachments/assets/56dff059-4495-418a-998a-240194a91b4a" />

Overview

Interceptor is an interesting TryHackMe room that requires careful enumeration and attention to hidden functionality. Some of the hints can be misleading, so approaching it with a standard methodology proved to be the most effective strategy.

⸻

Enumeration

The first step was performing an Nmap scan to identify available services.

After the initial reconnaissance, I used directory enumeration to search for hidden files and directories that might reveal useful information.
Run the dirsearch with extensions like php.bak ,sql ,bak etc..

<img width="1400" height="875" alt="1_A60DpyLnESYStcLkBA-8aQ" src="https://github.com/user-attachments/assets/7760a4d5-dd4c-43b2-b7f9-5681d3050253" />

Discovery of a Backup File
During enumeration, I discovered:
/login.php.bak

<img width="1400" height="600" alt="1_y-evIR9MAWBVX4q7fYROXQ" src="https://github.com/user-attachments/assets/82e082c8-df80-43c3-aa3a-eee75c34d57c" />

Inspecting this backup file revealed valuable information, including an email address and a clue related to the password.

⸻

Authentication

Using the information gathered from the backup file, I attempted to identify the correct password.

I bruteforced based on the hint ,But to save time
Current year played a role in the credential pattern.

After successfully authenticating, the application redirected me to a page requesting a six-digit OTP.

⸻

OTP Bypass

My initial approach was to test whether the OTP mechanism could be brute-forced. However, session expiration prevented this from being a practical solution.

While inspecting the requests in Burp Suite, I noticed additional parameters involved in the verification process. Modifying these parameters revealed a logic flaw that allowed the verification step to be bypassed.

<img width="1400" height="505" alt="1_iJrpWtEoCCoKfnxWePaOTA" src="https://github.com/user-attachments/assets/6c3f717f-dc7b-4e53-925a-555ae91c042e" />


After forwarding the modified request and reloading the page, I gained access to the application.

<img width="1400" height="632" alt="1_NIiBUqZajkb1uhVScEJpyA" src="https://github.com/user-attachments/assets/4f855931-80fc-4be5-95e5-1d6f45d0838f" />

[Edited panel .tiff](https://github.com/user-attachments/files/28865320/Edited.panel.tiff)


⸻

Post-Authentication Analysis

Once authenticated, two features stood out:

1. A file upload functionality
2. An internal resource retrieval feature

The resource retrieval functionality appeared particularly interesting, so I focused my attention there before exploring file upload exploitation.

⸻

Internal Resource Fetcher

The feature behaved similarly to a server-side resource retrieval mechanism (using curl command).

Several common input manipulation techniques were tested but did not produce the expected results.

Further investigation revealed that the application processed user-supplied input in a way that led to command execution opportunities.
I tried symbols like ;, && , | 
But nothing worked ,then I heard about command substitution 

A example for understanding is given below


  <img width="1400" height="399" alt="1_HDX1BiI2FgDfXKZk06nTlA" src="https://github.com/user-attachments/assets/571f464e-ce90-4820-a28b-9b5e235ff4ab" />

⸻

Obtaining the Flag

After identifying the vulnerability and successfully leveraging it, I was able to retrieve the flag and complete the room.

[Edited flag.tiff](https://github.com/user-attachments/files/28865506/Edited.flag.tiff)


⸻

Lessons Learned

* Always inspect backup files discovered during enumeration.
* Authentication workflows should never trust client-controlled verification states.
* Logic flaws can often be more valuable than brute-force attempts.
* Internal resource fetching functionality can introduce serious security risks when input is not properly handled.
* Careful request analysis in Burp Suite can reveal overlooked attack paths.

⸻

Tools Used

* Nmap
* Dirsearch
* Burp Suite

⸻

TryHackMe Profile

https://tryhackme.com/p/SamAlex

⸻
