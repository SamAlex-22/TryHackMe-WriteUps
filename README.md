Interceptor — TryHackMe Write-up

<img width="1400" height="299" alt="1_r7jnIqXT8WuWFKytXVi6KA" src="https://github.com/user-attachments/assets/56dff059-4495-418a-998a-240194a91b4a" />

Overview

Interceptor is an interesting TryHackMe room that requires careful enumeration and attention to hidden functionality. Some of the hints can be misleading, so approaching it with a standard methodology proved to be the most effective strategy.

⸻

Enumeration

The first step was performing an Nmap scan to identify available services.

After the initial reconnaissance, I used directory enumeration to search for hidden files and directories that might reveal useful information.
Run the dirsearch with extensions like php.bak ,sql ,bak etc..



Discovery of a Backup File

During enumeration, I discovered:

/login.php.bak

Inspecting this backup file revealed valuable information, including an email address and a clue related to the password.

Insert Screenshot: Enumeration Result

⸻

Authentication

Using the information gathered from the backup file, I attempted to identify the correct password.

One observation that helped was that the current year played a role in the credential pattern.

After successfully authenticating, the application redirected me to a page requesting a six-digit OTP.

Insert Screenshot: Login Page

⸻

OTP Bypass

My initial approach was to test whether the OTP mechanism could be brute-forced. However, session expiration prevented this from being a practical solution.

While inspecting the requests in Burp Suite, I noticed additional parameters involved in the verification process. Modifying these parameters revealed a logic flaw that allowed the verification step to be bypassed.

Insert Screenshot: Burp Request

After forwarding the modified request and reloading the page, I gained access to the application.

Insert Screenshot: Successful Login

⸻

Post-Authentication Analysis

Once authenticated, two features stood out:

1. A file upload functionality
2. An internal resource retrieval feature

The resource retrieval functionality appeared particularly interesting, so I focused my attention there before exploring file upload exploitation.

⸻

Internal Resource Fetcher

The feature behaved similarly to a server-side resource retrieval mechanism.

Several common input manipulation techniques were tested but did not produce the expected results.

Further investigation revealed that the application processed user-supplied input in a way that led to command execution opportunities.

Insert Screenshot: Internal Resource Fetcher

⸻

Obtaining the Flag

After identifying the vulnerability and successfully leveraging it, I was able to retrieve the flag and complete the room.

Insert Screenshot: Flag Retrieval

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

This write-up is intended for educational purposes and focuses on methodology rather than providing complete challenge solutions.
