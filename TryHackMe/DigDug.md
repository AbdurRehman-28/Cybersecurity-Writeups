# DigDug — TryHackMe

**Focus:** DNS Enumeration
**Room Link:** [DigDug on TryHackMe](https://tryhackme.com)

## Reconnaissance

I started by scanning the target with Nmap:


**nmap 10.48.163.36**


Port **22 (SSH)** was open. However, the challenge description indicated that the machine was also a **DNS server**, so I focused on DNS enumeration.

## DNS Enumeration

The challenge provided the domain: **givemetheflag.com**

I queried the target machine as the DNS server using **dig**:


**dig @10.48.163.36 givemetheflag.com**


The response showed an *A* record in the question section:

**QUESTION SECTION:**
**givemetheflag.com.    IN    A**


Interestingly, the DNS server returned a **TXT record** in the answer section:


**ANSWER SECTION:**
**givemetheflag.com.    0    IN    text    "[REDACTED]"**


The text record contained the challenge goal.

## Takeaway

The key to the challenge was recognizing the DNS hint and using **dig** to query the target DNS server. The flag was retrieved from the returned text record.

## Disclaimer
 **This write-up is for educational purposes only. The flag and other sensitive challenge details have been redacted in accordance with TryHackMe's write-up guidelines**
