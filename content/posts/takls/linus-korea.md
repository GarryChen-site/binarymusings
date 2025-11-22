---
title: "Linus Torvalds on AI, Linux, and Life in Korea"

description: "Linus Torvalds shares candid views on AI coding, Linux kernel maintenance, Rust in the kernel, Nvidia's GPU dominance, burnout and stress relief at OSS Summit Korea."

summary: "At OSS Summit Korea 2025, Linus Torvalds talks AI coding hype, Rust in the Linux kernel, Nvidia GPU power, maintainer burnout, and why failing at guitar pedals keeps him sane."

date: 2025-11-22
series: ["Talks"]
weight: 1
tags: ["linus-torvalds", "linux-kernel", "ai", "rust", "open-source"]
author: ["Garry Chen"]
cover:
  image: images/
  hiddenInList: true
  caption: ""

---

* **"For nearly the past 20 years, I haven't really been a programmer."**
* **"As for Git, which I created, I'm now just in an observer role."**
* **"I used to say my job was to say 'no' (to proposals), but now I find myself saying 'yes' or 'okay' to new things, sometimes against the objections of long-term maintainers."**
* **"'Vibe Coding' allows people to do things they couldn't before, but from a maintainer's perspective, maintaining the code it generates 'might be absolutely horrible'."**

These statements are not jokes or self-deprecation, but rather the sober admissions of Linus Torvalds, the father of Linux and creator of Git, in the face of technological waves. 

Earlier this month, Linus Torvalds had a conversation with Dirk Hohndel, Head of Open Source at Verizon, at the Linux Foundation's Open Source Summit held in Seoul, South Korea. He discussed his changing role, how AI is reshaping software development, his thoughts on the increasing reliance of hardware on Nvidia's proprietary GPUs and CUDA rather than open-source Linux, the conflicts Rust has sparked within the kernel team, the real-world predicament of kernel.org being severely disrupted by various AI crawler tools, and his daily pressures and how he copes with them.

Amid the AI boom that is almost rewriting the fate of developers, Torvalds admitted that he does not use AI to assist in writing code and hasn't even tried it out. "But I'm sure someone is already looking into whether it's applicable to the kernel codebase." When asked if AI would make programmers' jobs disappear, he simply stated: "AI is just another tool, like how compilers freed people from writing assembly by hand, greatly improving productivity, but it didn't make programmers disappear." 

Of course, if anyone disagrees with his view, they can email him. However, he said: "I almost guarantee I will read it, but I also almost guarantee I won't reply." He joked, "I rarely reply to emails. If you don't get an email from me, it means I'm fairly satisfied. I just don't let people know much. I apologize for that."

**Here is the complete content of that conversation:**

## "I'm not a programmer anymore; I don't do many things myself. I'm more watching Linux move forward."

Dirk Hohndel: My name is Dirk Hohndel, I'm responsible for open source at Verizon. I've been involved since the Linux Foundation was founded, and I've been involved with Linux for almost as long as the person on this stage – because you are...

Linus Torvalds: Yes, I'm Linus. We're doing this interview because I hate public speaking. In contrast, I have no idea what Dirk will ask me, but that feels much more relaxed. Over many years, we've had these chats once or twice a year. This format isn't new; compared to traditional speeches, this way makes me, someone who usually doesn't enjoy the 'public figure' role, feel more comfortable.

Dirk Hohndel: This is indeed our 28th time having such a conversation, which is quite interesting to think about. The last time we were here was exactly ten years ago, and I'm very happy to be back in Seoul. For me, every trip to Asia is fascinating. The way people here view open source and software development is different; it's a completely different world, and I'm captivated by it.
Linus, ten years ago you had just released Linux 4.8. 

Could you briefly summarize the biggest changes over these years?

Linus Torvalds: A lot of work has indeed been done. But I must first emphasize something I often repeat because it's important – the real work isn't done by me. For nearly the past twenty years, I haven't really been a programmer; I'm more the technical lead and maintainer of the system. 

That's true for Linux, and even more so for Git, where now I'm almost just an observer. 

I want to remind everyone that the real contributions are made by others, perhaps even by people sitting in the audience. Many people attribute all the credit to me because I've stayed with the Linux project. Actually, now I'm more 'watching' this kernel project move forward.

Dirk Hohndel: Looking back over the past decade, what has left the deepest impression on you in the evolution and development of Linux?

Linus Torvalds: What impresses me most is – I used to say that one day this project would be 'finished.' But that was a thought from a long, long time ago. I've been working on Linux for almost 35 years now, and I don't feel at all that there will be a point where we can say, 'Okay, that's it.' 

In fact, I've gradually realized that for all truly long-lived projects, the core work is actually maintenance and ongoing support. Especially for the kernel, Greg (Greg Kroah-Hartman, Linux kernel developer) and I were discussing just yesterday that as long as new hardware keeps emerging, there will always be new work on the kernel side. But even excluding new hardware, what somewhat surprises me is: 35 years after the project started, we are still modifying core kernel code to make it cleaner, more maintainable, and more stable. 

At 3 AM this morning, due to jet lag, I was discussing with someone how to clean up some code. 

For a system like Linux, the real work is constant maintenance, keeping everything running smoothly, while also facing new challenges – whether from hardware or the ever-changing software ecosystem.

Dirk Hohndel: From a process perspective, the Linux kernel development model has been very stable for the past 15 years. However, that's 'too boring' for the media. They often focus only on the moments you raise your voice, or any time you reject a proposal. In your feeling, has the situation gotten better? Worse? Or stayed about the same? How often do you feel you have to step in now and say, 'We are not doing this'?

Linus Torvalds: One change is quite obvious: I used to say that my job was mainly about saying "no." People would propose all sorts of radical new ideas, which might be interesting, but sounded like maintenance nightmares, so I would say: "No. You put it in your own sandbox, implement it, prove me wrong with data, and then come back to me." I felt that was a large part of my job as a system maintainer.

But in the past few years, I've found that sometimes my job is actually about saying "yes." Because... you know, after being in this circle for so long, with hundreds of maintainers also having been around for decades, people tend to get stuck in their ways. Sometimes you want to break the deadlock and say: "Hey, let's try this new thing," and I'm the one who says, "Okay, let's do it."

Take the adoption of Rust as an example, even though we've been working on Rust for about five years now, it's not entirely new. But initially, I felt that the kernel shouldn't stagnate; we need to try new things and also attract new people.

This is one of the biggest changes for me: I now need to encourage other maintainers to be more open to new ideas.

## "Rust has become part of the Linux kernel, taking longer than I expected."

Dirk Hohndel: Rust is one of the examples I wanted to mention. I've noticed that although Rust has been around for five years, it only really started entering the kernel code about three years ago. It has indeed sparked quite a lot of discussion and controversy. 

Some people expressed their frustrations, others argued about code formatting issues, or there were disagreements during code reviews due to unfamiliarity with the language. There were even maintainers who stepped down because of it. Do you think it was all worth it? Was introducing new technology worth disrupting our development process?

Linus Torvalds: I think it was worth it. But I also think that Rust indeed attracted a lot of media attention, perhaps because it's quite visible within the kernel. Of course, there are other places with noticeable Rust code, but the fact is, we have disagreements in almost every area of the kernel because that's part of new development and bug discovery. People can get very passionate when defending their viewpoints, but in that sense, Rust isn't fundamentally different from other areas; it's just that it might make the news more easily. 

I think we've now reached a stage (Greg might elaborate more, he follows it more closely than I do) – Rust is truly becoming part of the kernel, no longer just experimental. 

Admittedly, it took longer than I expected, no doubt about that.

Dirk Hohndel: Actually, the more notable "heated controversy" earlier wasn't entirely related to Rust. The first time a component was removed from the kernel wasn't related to Rust either; it was actually entirely due to interpersonal issues.

Linus Torvalds: That's right, this year has been a bit turbulent. We've had many disagreements, even moving parts of the kernel's functionality out of the kernel to reduce friction. 

But to be fair, this isn't the first time something like this has happened. There have been modules in the kernel before that were no longer used or had serious issues and were removed. In 35 years, this has actually happened very rarely, and it's not pleasant, but I think we've handled it quite well. After all, it's a large project with thousands of participants every release, every two months. You'll have personal disagreements, professional disagreements, friction. That's all part of life. I think we are still largely a happy family.

Dirk Hohndel: I think I'd be more inclined to describe it as a group of very mature people who have found a way to coexist with each other. But I'll go along with your "happy family" description. Usually, this is the first thing I ask you, but today I'll put it at the end of the first part: Regarding the 6.18 RC4 version, is there anything you'd like to say?

Linus Torvalds: No. That's the current kernel version. I like "boring." For me, "boring" means no super exciting new features and no causing millions of machines worldwide to crash. 6.18 doesn't look like a problematic release. We had a series of test failures, but it turned out that those were largely failures of the tests themselves, not kernel failures. I was a bit worried a few weeks ago, but now it seems to be heading towards another incremental, boring – in the best way – release.

## The rise of Nvidia, AMD hardware, and its impact on Linux

Dirk Hohndel: Looking at major industry changes, I think one of the biggest changes is on the hardware side. For decades, everything revolved around the CPU; everyone talked about the CPU. Who had the fastest CPU, the best architecture. In the past few years, with the rise of companies like Nvidia and AMD, accelerated processing units (APUs) have become the focus. 

Interestingly, while these processors are relevant to Linux machines, Linux itself isn't actually running on these processors. What's your view on this trend of hardware focus gradually moving away from Linux?

Linus Torvalds: I don't see it that way. I still think the most interesting part is the general-purpose CPU. It might not make the news as often because it's been around for a long time, and people take it for granted. What Linux does is manage the system, boot the system, handle the UI, and all the things you expect the system to do. The AI part is the industry's new darling, and that's fine. However, it's not completely separate; it's a different environment that Linux helped foster and enable, and I don't feel the kernel necessarily has to be an extremely integral part of it.

For me, as a kernel maintainer, this is essentially no different from user space. While I personally love open source and don't want to participate in non-open source projects, open source has never been a religion to me. I do open source, Linux is also open source, but people have always run commercial applications on Linux, like large databases, cloud services, etc. This is very normal. 

For me, a GPU is just another form of the same thing; you run your AI workloads on top of the kernel. The fact that it has its own system for maintaining the GPU hardware is generally not something Linux needs to worry about excessively. We are actually involved in it to some extent. There are many things like resource management, virtual memory handling, etc., where the kernel is deeply involved. 

This is actually one of the benefits brought by AI; it has made Nvidia a good participant in the Linux kernel space. As everyone knows, this wasn't the case 20 years ago. Now, when Linux is so important for AI clouds, Nvidia suddenly cares very much about Linux, and we have many kernel maintainers in that area now too. So this is one of the positive aspects brought by the AI boom.

## "AI's application in the Linux kernel is experimental at best, I've never played with AI-assisted code"

Dirk Hohndel: I think whenever a vendor embraces what we do and participates, it's a very positive thing. That's great. Since you've mentioned AI so many times, I have to chat about this. 

Last year we talked about the potential of AI or generative AI for code review, code explanation. The Linux kernel community has done quite a bit of work around this. How is it progressing now?

Linus Torvalds: Well, it's not in place yet. There are indeed people doing a lot of work, some are trying to use AI to help maintainers handle patch streams, backport patches to stable versions, etc. Frankly, most of it is still experimental. The biggest problem we face is that AI causes significant interference with infrastructure. For example, AI crawlers scraping kernel.org source code everywhere, this causes huge trouble and isn't always pleasant. 

But there are also some good aspects. I look forward to the day when AI is no longer over-hyped and becomes more like an everyday reality that nobody talks about all the time. Clearly, that day is still a few years away. I think exciting new technologies are always something people want to talk about. Of course, with trillions of dollars being invested, people are even more filled with curiosity.

Dirk Hohndel: One thing that impressed me: at the Open Source Summit in Amsterdam, Daniel Stenberg of Libcurl mentioned that AI-generated low-quality security reports have almost become a "denial-of-service attack" against his project. Have you encountered anything similar on the kernel side?

Linus Torvalds: We have it on the kernel side too, but not as severe. But we do see some bug reports and security advisories that are clearly fabricated by someone misusing AI. This consumes maintainer resources. In some projects, this problem is worse than in the kernel.

Dirk Hohndel: Of course, another topic everyone most wants to discuss is AI-generated code. I often compare it to "enhanced autocorrect," because AI is indeed great at code completion, syntax checking, standard library usage. On the other hand, the much-discussed Agentic AI nowadays – basically you tell the AI: "Hey, Claude, I want you to develop this feature," and some even say, "With AI's help, I made a complete product within a week." Have you played with these things yourself?

Linus Torvalds: I haven't played with it at all. But I'm sure people are researching it, even wanting to apply it to the kernel codebase. However, I think the kernel is complex and special enough that although we've open-sourced a lot of code for AI to learn from, it's difficult to use it directly on the kernel. I suspect very few people would use Vibe Coding to write kernel code; it's more for their own small projects.

Actually, I think most of this is good. The way I got into computers as a kid was simple, typing out programs from magazines line by line. That's how I fell in love with computers back then. 

Computers are too complex now, and the requirements for programming are much higher, making it much harder to get started than in my day. Using Vibe Coding for a real product is probably a terrible idea from a maintenance perspective. But it is a great way to get new people involved, feel the joy of programming, and make computers do things that weren't possible before. So I'm generally positive about it.

Dirk Hohndel: I mean, that thrill obviously exists, entering a new programming language, a new environment, a new set of libraries, letting the tool do 90% of the work, that's exciting. But I've spent a lot of time on this; the tool can help you complete 90%, and it does it very well. But that remaining 10%...

Linus Torvalds: That remaining 10% is what has taken up 34 of my 35 years in this project.

Dirk Hohndel: Exactly. So there's a lot of opportunity here to create great things, but there's also a great need to actually land these things properly. But we are indeed seeing a lot of discussion about software developer layoffs, a real wave of unemployment in the US, with tens of thousands of people being laid off. The reason is often "Oh, AI makes us more efficient." If you think about students studying computer science today, do you think software development as a career will be significantly impacted?

Linus Torvalds: Honestly, I don't know. This is one of those questions where I'd say, "Hey, let's wait a few years and see what the real answer is," because I think it's a complex issue. 

My personal guess is that you'll find you need just as many maintainers to keep the project actually running. AI is just another tool, just like compilers freed people from writing assembly code by hand and greatly improved productivity, but didn't make programmers disappear. 

I think AI will ultimately be the same. It's another tool that lets you avoid dealing with all the nitty-gritty details, but it won't make real programmers disappear. That's my intuition. If anything, it might make people more efficient, but it also opens up entirely new areas of development, so you might actually end up needing more software programmers. 

Dirk Hohndel: That's exactly what I was thinking. If you get these productivity boosts, you can do a couple of things. You can say, "I'll do the same thing with fewer people," or "I'll do more things with the existing people." To me, one of the biggest opportunities with generative AI is that we can do things that were impossible in the past because the initial barrier to creating a demo prototype was too high. So from my perspective, for newcomers in computer science today, being able to express ideas, create demos, or prototypes using modern tools is as important as writing a bubble sort was 20 or 30 years ago. 

This is interesting because it really changes the work content of software engineers and how you interact with the system. I find your comparison to assembly language and machine code very apt. Or the transition from C (which some people still use) to object-oriented languages was similar.

## Making your own guitar pedals is to relieve stress

Dirk Hohndel: We've talked so much about software, let's talk about hardware. Some people have really strange hobbies. For example, some people make their own pedals for string instruments. Can you talk about your experience playing with guitar effects pedals? 

Linus Torvalds: The background of this very strange hobby is: Last Christmas, I started making guitar pedals for fun. It makes no sense because I have no musical talent, have never touched an electric guitar in my life, but I wanted to learn electronics. So I started making guitar pedals, first with kits, then designing my own. They all turned out terribly. I don't really want to encourage others to do this because it's pointless. After all, all modern guitar pedals are digital now.

But the reason I do it is because I think – and this is what I encourage everyone to do – when you have a high-stress, high-stakes job and you feel the need to do something else to relax, you should find a hobby where failure is not only expected but is actually fun.

It doesn't have to be guitar pedals, it can be anything, anything at all. For me, the point of interest happened to be soldering and making hardware. I know I'm completely incompetent at it, but I really enjoy it. Some people think failure is a bad thing, and I happen to be the kind of person who likes doing things I'm not good at because that's how you learn. You have to accept that you will fail. I've been playing for a year and haven't fully learned it yet (laughs).

Dirk Hohndel: I disagree, I have a few pedals you made, and they are getting better. 

Linus Torvalds: This is something I would encourage anyone in this industry to do, because this line of work can be quite stressful at times. Especially... if you do open source, at least for me, the most stressful part is often people. I don't find the technology stressful. But sometimes when you have disagreements, and you really want to say, "I want to take a break, I need to do something completely different," that's when you need a hobby or something, where you can say, "Hey, this has nothing to do with my job, and it's okay to mess up." For me, that's electronics. 

Dirk Hohndel: I think what's interesting is that the electronic hardware you make is simple, while the Linux open source project you're responsible for is the most complex thing in the world. This stark contrast fascinates me. 

Linus Torvalds: Yeah, my electronics hobby is actually becoming more and more "regressive." I started with slightly fancy integrated circuits, then I started regressing, and now I'm playing with and really understanding how individual transistors work. My day job is dealing with hundreds of billions of transistors, and my personal hobby is dealing with three transistors. So these are my two extremes in hardware.

## Linus's Daily Routine: Reading emails, but rarely replying to emails 

Dirk Hohndel: You mentioned earlier that you don't write software anymore, you're a manager. Now we know you usually play with relatively simple hardware. So, what exactly do you do more of in your daily routine? 

Linus Torvalds: The reality is, I sit in front of the computer every day reading emails. I almost never reply to emails. If you send me an email, I can almost guarantee I'll read it, but I also almost guarantee I won't reply. The occasions where I reply to emails are very rare. 

Actually... I kind of want to apologize. Not only to everyone who emails me, but also to those developers who only see the complaining side of me. People think I'm an angry, mean old man because the types of emails I reply to are often about various problems that occur. And when everything is going well – which is actually the vast majority of the time – I don't send an email saying, "Thank you, this was done really well." So if you don't get an email from me, it means I'm quite satisfied. I just don't let people know. I apologize for that. 

Dirk Hohndel: I think this is a good point to end on. This information shows that Linus is actually a very nice person, just keeping the goodwill hidden in his heart. 

Linus Torvalds: In my heart, I'm very happy. It's just that my outward expression isn't always like that, and I deeply apologize for that.

## Video Source

[Watch the full conversation on YouTube](https://www.youtube.com/watch?v=tWx769t1JKg)