---
title: eCPPT Review - 2023
date: 2023-12-28
categories: [CertificationReviews]
tags: [eCPPT, Certifications, INE, PenetrationTesting]
---

Recently, I took on a personal challenge by pursuing my first practical penetration testing certification. I am excited to share that I successfully passed the Certified Professional Penetration Tester (eCPPTV2) certification exam offered by INE Security.

To achieve this certification, participants are required to submit a professional penetration test report within a 14-day timeframe, with the first 7 days dedicated to hacking and another 7 days for report writing. I managed to pwn all machines of the exam on the 3rd/4th day and took my precious time in writing the report. 


## About Me

I work full time as a security consultant with 3 years of experience mostly in web application security. I had decent experience in doing CTFs and boot2root machines prior to taking this certification.

## Preparation

I took the exam without subscribing to the course provided by INE. Here are some of alternative labs and course materials that I practiced on before taking the exam:

**TryHackMe**:
- Offensive Pentesting learning path (Feel free to skip AD section)
- Wreath

**HackTheBox Academy**:
- Pivoting, Port Forwarding, and Tunneling module
- Stack-Based Buffer Overflows on Windows x86 module
- Documentation & Reporting module

**TCM Security**:
- Practical Ethical Hacking course

The materials above may be overkill, but it's better to come in overprepared than not.

**Buffer Overflow Prep**:
- I prepared an x86 Windows 7 VM with ASLR and DEP disabled.
- I also installed ImmunityDebugger and mona.py to the said VM
- I prepared scripts in advance that sends the shell code and checks for bad characters.

**Report Writing**:
- I prepared a template report hosted on GoogleDocs. Having it on cloud was very useful to me as I was switching between using my PC and laptop.
- Greenshot can be very useful in taking nice screenshots.
- I placed my screenshots and notes via Microsoft OneNote

  ![](https://raw.githubusercontent.com/jmrcsnchz/jmrcsnchz.github.io/main/assets/2023-12-29%2003_38_12-.png)

## Exam Experience

Since I work full time, I had limited time in doing the exam. I started the exam 10pm on a thursday so the timeline will be very messy lol, please bear with me

### Day 1
I started the exam on a Thursday night. I had initial access and root privileges to the web server in under a few minutes. I was able to pivot into the internal network but was not able to move further. I called it a day and went to sleep.

### Day 2
Friday morning. With a fresh mind and good sleep, I took another chance to the exam before I went to work. I managed to pwn another machine in 20 minutes and was very excited. I gained access to more useful information that was vital to the exam. I went to work very motivated and energized lol.

After work, I managed to pwn my 3rd machine. The solution is very simple but it requires you to be a little bit creative. 

### Day 3
Saturday morning. No work! I ordered pizza because I knew that I won't be able to leave my table during that day. I needed food that is easy to consume but is filling. 

- **12:00 pm:** I had problems running the vulnerable binary in my 32bit windows 7 VM
- **1:30 am:** Managed to make the binary work
- **2:00 pm:** Managed to write a working POC and made it work on my VM. Unexpectedly, It did not work on the exam machine.
- **4:30 pm:** After two hours and a half of banging my head and rabbit hole-ing, I managed to pwn the buffer overflow machine. At that point, I pwn'd 4 machines already and all that was left was the machine/s in the DMZ
- **8:30 pm:**  I discovered the IP address of the last machine in the DMZ in which I needed to pwn in order to achieve the exam objective. I felt the instability of the lab environment during these times. This was maybe due to the traffic running through two pivot machines as I was attempting to enumerate another subnet.

After discovering the last machine, I rabbit holed A LOT. I was puzzling together pieces of post-exploitation data but I struggled to find the kill-path for the last machine. At that point, I was already doubting my capabilities.  I took a break and chugged a lot of soda as I felt very thirsty for reasons I do not know why. 

- **11:00 pm:** I gathered my self and re-enumerated again. 
- **11:50 pm:** I gained foothold to the last machine and all that was left was the root access.

### Day 4
- **2:00 am:** I found the privilege escalation vector and gained root access to the last server. I think that the vector is pretty stupid and unrealistic, but hey, a win's a win! I celebrated a little and started outlining my report. Since the exam requires a full penetration test, I also looked for other vulnerabilities and gathered my screenshots.

   ![](https://raw.githubusercontent.com/jmrcsnchz/jmrcsnchz.github.io/main/assets/2023-12-29%2003_28_37-Clipboard.png)

## Exam Tips & Tricks
- Practice pivoting with Metasploit and Proxychains.
- Practice Metasploit post-exploitation modules.
- Know when to use reverse shells vs bind shells.
- Take your time. The time given were more than enough.
- Trust your instincts! If something should work, try different payloads or a different approach.
- Create a cheat sheet of common commands that you can easily copy and paste.
- Taking breaks helped me overcome rabbit holes.
- Screenshot everything.
- Don't treat the exam as a CTF; there are no flags to capture, and you have to document every vulnerability you come across.

## Overall Thoughts About the Exam
Overall, the exam tests your methodologies and creative thinking alongside your technical competence. If you are already in the offensive security field right now, you may find the exam to be outdated. I took the certification as it is now the only exam with BOF (after the revision of OSCP) and I wanted to experience it before they phase it out in the future.


I don't believe this certification holds great value for HR, but I still recommend it to those wanting to challenge themselves or to those aiming to expand their skill set.

![](https://raw.githubusercontent.com/jmrcsnchz/jmrcsnchz.github.io/main/assets/ecppt-1.png)

If you plan on taking the eCPPT, I wish you luck and I hope this review somehow helps :)
