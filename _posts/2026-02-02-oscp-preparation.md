---
layout: single
title: Beyond Try Harder: My OSCP Preparation Journey
excerpt: "A couple of weeks ago, I completed one of the most significant milestones of my professional life. I decided to put together this post to share my preparation strategy."
date: 2026-02-02
classes: wide
header:
  teaser: /assets/images/oscp-prep/logo.png
  teaser_home_page: true
categories:
  - OSCP
  - Certifications
  - OffSec
tags:
  - offsec
  - proving-grounds
  - oscp
  - preparation
  - study
---

# Index
- [Background](#background)
- [Preparation](#preparation)
  - [The Official OffSec Course (PEN-200)](#the-official-offsec-course-pen-200)
  - [Proving Grounds: The Real Game Changer](#proving-grounds-the-real-game-changer)
    - [Round One: Facing Reality](#round-one-facing-reality)
    - [Round Two: Refinement](#round-two-refinement)
    - [Round Three: Mastery](#round-three-mastery)
  - [OSCP Challenge Labs](#oscp-challenge-labs)
- [Exam Day](#exam-day)
- [The Results](#the-results)
- [Final Tips](#final-tips)

A couple of weeks ago, I completed one of the most significant milestones of my professional life. After years of putting it off, the opportunity finally arrived to take the most prestigious exam for many in the pentesting world: the **OSCP**.

This certification proves your ability for "out-of-the-box" reasoning and working under pressure. You aren't just fighting the technical challenges; you're fighting the clock (with 24 hours to complete the exam) and the psychological weight of being proctored via webcam. (Spoiler alert: You actually _do_ have enough time to eat, sleep, and function normally if you manage it well).

After several hours of testing different attacks and techniques, I passed the exam in 16 hours with a **perfect score of 100**. Because of this, I decided to put together this post to share my preparation strategy. It’s important to note that this isn't a "review" of the exam itself, but rather a breakdown of my action plan and my preparation process (which, in hindsight, was probably more than necessary).

# Background
About seven years ago, I discovered **HackTheBox (HTB)**. Back then, you had to complete a web "hacking" challenge just to create an account. That specific challenge was eventually retired but preserved for posterity as the initial access vector for the machine [TwoMillion](https://www.hackthebox.com/machines/twomillion), released to celebrate HTB's two million users.

During that time, I learned the ropes of CTF-style pentesting from content creators like [S4vitar](https://www.youtube.com/@s4vitar) and [IppSec](https://www.youtube.com/@ippsec). Both held the OSCP, and I set a goal for myself: one day, I would earn it too. After years of solving machines on platforms like HTB and TryHackMe (THM), and earning intermediate certifications like eJPT, eCPPT, and PNPT, I knew I was ready. The final push came when my employer provided me with a **Learn Enterprise** membership, which included an exam attempt, and I decided to make the most of it.

# Preparation
## The Official OffSec Course (PEN-200)
After years of learning initial access techniques, web app hacking, service enumeration, and privilege escalation for both Windows and Linux, I was somewhat skeptical about what the official course could offer me. Still, I decided to review it.

I followed the [Official 12-week Learning Plan](https://help.offsec.com/hc/en-us/articles/15541765522196-OffSec-PEN-200-Learning-Plan-12-Week), but upon reaching Topic 16, I decided to better focus on solving machines. Since I already had years of experience, the subsequent topics (PrivEsc, AD attacks, etc.) didn't offer much new information for my specific case.

> **Note:** If you are just starting out, I **highly recommend** completing the entire course. It teaches fundamental skills and specific "OffSec-style" methodologies that are vital for the exam.

## Proving Grounds: The Real Game Changer
I didn't feel comfortable going into the exam without specific practice, so I pivoted from the course material to grinding machines on **OffSec Proving Grounds (PG)**. OffSec themselves highlight a strong correlation between the number of PG machines solved and the OSCP pass rate.

![1]
_Source: [PEN-200 Onboarding - A Learner Introduction Guide to the OSCP+](https://help.offsec.com/hc/en-us/articles/4406841351316-PEN-200-Onboarding-A-Learner-Introduction-Guide-to-the-OSCP)_

Trust me, those numbers don't lie. During my prep, I solved **88 machines and 5 Challenge Labs**. In my opinion, this was the most valuable part of my journey. I combined the famous lists from [TJNull](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview) and [Lainkusanagi](https://docs.google.com/spreadsheets/d/18weuz_Eeynr6sXFQ87Cd5F0slOj9Z6rt/edit?gid=487240997#gid=487240997), focusing exclusively on Proving Grounds. Why? Because PG is owned by OffSec. If you want to pass an OffSec exam, study the machines built by the same people who design the exam.

### Round One: Facing Reality
I created a spreadsheet to track my progress, categorizing machines by OS, source list, and difficulty. Most importantly, I tracked my "Success Rate".

> **NOTE:** I have blurred my internal notes for each machine to avoid any spoilers regarding their resolution.

**"Achieved?" column:**
1. **Yes:** Solved without any help.
2. **More or Less:** Needed a small hint from Discord to get unstuck.
3. **No:** Needed a full write-up or multiple hints.

![2]

My goal was to solve at least two machines a day after work, treating them like a real exam (no outside help, only my notes and Google). If I couldn't progress after two hours, I'd look for a hint.

**Round One Results:**
- **Linux (63 machines):** ~60% success rate.
- **Windows (20 machines):** Slightly below 50%.
- **Active Directory (5 machines):** Only 1 out of 5 solved without help.

![3]

I’m showing you these "failures" for a reason. For a long time, I postponed this exam because I didn't feel "good enough." Even on "Easy" HTB machines, I often felt lost. If you are in that position now, don't give up. **Struggling during practice is where the real learning happens.**

### Round Two: Refinement
I went back and tackled every machine marked "More or Less" or "No." This included 23 Linux, 8 Windows, and 3 AD machines. I used the same strict rules as before.

>**Note:** Some machines were excluded from this round for various reasons, either they weren't working correctly or were considered significantly more difficult than the actual exam.

![4]

The results showed a massive improvement. My notes were becoming more robust, and my "intuition" for finding vulnerabilities was sharpening.

### Round Three: Mastery
Finally, I revisited the remaining handful of machines that still gave me trouble. By this round, 100% of the machines were solved without any external help.

## OSCP Challenge Labs
To wrap up, I tackled the official Challenge Labs: **Secura, Medtech, and OSCP A, B, and C.**

I completed **Secura** and **Medtech** normally, not as mock exams, but as environments to practice my skills. I only needed one hint for Medtech; the rest of the infrastructure was manageable.

On the other hand, I treated **OSCP A, B, and C** as "Mock Exams." I gave myself a 24-hour limit for each, using only my notes and Google. Based on the [official](https://help.offsec.com/hc/en-us/articles/360040165632-OSCP-Exam-Guide) 100-point scale (40 for AD, 20 for each Standalone), these were my mock scores:

- **OSCP A:** 70 points    
- **OSCP B:** 80 points
- **OSCP C:** 30 points

That 30 on the final lab was a reality check. It bruised my confidence, but it forced me to fill the gaps in my notes one last time.

# Exam Day
I scheduled my exam for 8:00 AM to maximize my daylight hours. I had a clean, updated Kali VM with snapshots ready and the following "battle kit" in my `/opt` directory:

- **Windows:** winPEAS, PowerUp.ps1, etc.
- **Linux:** linPEAS, linEnum, pspy.
- **Pivoting:** [Ligolo-ng](https://github.com/nicocha30/ligolo-ng/releases) (Since the Challenge Labs required pivoting into internal networks, I had my ligolo agent and proxy binaries ready).
- **Shells:** [Penelope](https://github.com/brightio/penelope) (for stable reverse shells).

Following advice from [William Chew](https://medium.com/@williamchew85), I also organized my workspaces to avoid confusion:

- **Desktop 1:** VPN, pivoting, and Python web servers.
- **Desktop 2:** Active Directory.
- **Desktop 3-5:** Standalone machines 1, 2, and 3.

![5]

I took this idea from his [Medium review for the OSCP](https://medium.com/@williamchew85/pwned-oscp-in-two-months-7d4d5c9527ce). He also mentions other interesting features for exam day that you might want to check out.

# The Results
I reached the passing threshold (70 points) within 7 hours. By the 10-hour mark, I had 80 points. The final machine was a beast, unlike anything I had seen. I spent 6 hours banging my head against it, testing technique after technique. Finally, at midnight, the root shell popped.

I spent the next hour ensuring my screenshots were perfect, thanked the proctor, and went to sleep with **100 points** in the bag. The next day, I wrote my report and submitted it. The "Pass" email arrived the following evening.

![6]

# Final Tips
- **Preparation is everything:** Don't just "do" machines; understand _why_ they work.
- **Build a God-tier Cheatsheet:** My notes are the result of 7 years of learning. Having every command indexed and ready was my biggest advantage.

![7]

- **Watch out for Tunnel Vision:** If you're stuck for two hours, walk away. Take a shower, eat, or anything that makes you feel happy. Fresh eyes find shells.
- **Trust the Process:** You know more than you think you do. The struggle you feel during prep is exactly what builds the skills to pass the exam.
- **The "Brain Dump" Technique:** If you think you've tried everything, take a breath, reset the machine, and **write down in your notes everything you have done so far.** Venting your ideas onto paper can help you see paths you missed or tests you skipped due to stress.

---

**Disclaimer:** OSCP® and OffSec® are registered trademarks of OffSec Services LLC. This blog post is not affiliated with, sponsored by, or endorsed by OffSec. All references to course names, certifications, and logos are used for informational and educational purposes only. The views and experiences expressed here are entirely my own.

[1]:/assets/images/oscp-prep/1.png
[2]:/assets/images/oscp-prep/2.png
[3]:/assets/images/oscp-prep/3.png
[4]:/assets/images/oscp-prep/4.png
[5]:/assets/images/oscp-prep/5.png
[6]:/assets/images/oscp-prep/6.png
[7]:/assets/images/oscp-prep/7.png