---
title: "Online Code Interviews"
date: 2024-04-29T20:01:18-03:00
tags:
  - career
  - startups
categories:
  - general
draft: false
summary: "How I will conduct code interviews from now on."
---

During the last 2 or 3 weeks, I've had to conduct many code interviews for [Wúru](https://www.wuru.ai). This interviews usually last 45 minutes and consist of a 15-minute introduction with a technical talk and a 30-minute live coding session.

Usually the transition from intro to live coding is a bit awkward, because, I have to explain the candidates that they should open their preferred text editor and be ready to compile a source file after the task is done. To avoid this roughness I devised this simple idea, that should fit my needs from now on.

I created a private **GitHub** repository, where I left a `README.md` file in the main branch. For each interview I conduct I will create a new branch with the date, and the name of the candidate (e.g. `20240924_javier-roberts`).

With the branch created I checked it out and left the assignment in a `.txt` file in the root dir. This saved me the hassle of explaining constraints, limits and the actual problem-to-be-solved.

The next step was opening a GitHub Codespace.

When it was open I installed [VS Code Live Share](https://marketplace.visualstudio.com/items?itemName=MS-vsliveshare.vsliveshare) extension and created a link for that particular session. Up to this point it was all pre-interview work. Some of which will not be repeated as the codespace will remain created with the extension installed.

When the time came I handed the link I created to the candidate and started the live coding session. The friction of starting coding was really low. The only thing to bear in mind is that if the candidate joins from their browser they have to decide wether logging in with Github (maybe they've done so) or joining anonymously.

An advantage I found on this method is that the user can join from their browser or their own VS Code, Itellij or Eclipse (I really wonder who uses Eclipse these days). Another upside is that they do not need to share their screen, making the interview more confortable for them.

Another updside is the fact that I get to keep a record of the people I interviewed and the code they wrote.

Downside is that I found it difficult to share the terminal, but given the nature of the interviews compiling is not a big issue. For now. Also my guess is that there should be a way to do so.

In conclusion, I will stick to this new method for now and continue to refine it. So far in the 2 interviews I tested it, it worked very well. The only remaining task is to find the right candidate for the job.
