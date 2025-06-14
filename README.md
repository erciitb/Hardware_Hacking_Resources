# 🔓 Hardware Hacking — Where Circuits Spill Secrets

What if you could steal data without touching the code? What if a tiny delay or a dip in power was enough to reveal a password?

Welcome to hardware hacking—where we don’t break into software, we tap into the signals flowing through the wires.

This repo is your launchpad into the world of side-channel attacks. We’ll start with the basics: how signals behave, what serial communication really looks like, and how information leaks through power usage and timing quirks.

You won’t need expensive gear or a hardware lab—just curiosity and a few lines of code. Through simple simulations, you’ll see how hardware, even when it’s doing everything “right”, can still leak secrets.

Ready to listen in on what the hardware is trying to hide?

Let’s begin.

---
<br>

Before we dive into the fun stuff, here’s a banger of a **[book](./resources/ebin.pub_the-hardware-hacking-handbook-breaking-embedded-security-with-hardware-attacks-1nbsped-159327.pdf)** to kickstart your hardware hacking journey.  
Think of it as your sidekick while you spy on power traces and interrogate unsuspecting microcontrollers.  
We'll only be exploring certain concepts from the book but feel free to explore other chapters if you want!  
Now onto the fun stuff 😈.

---

## ⏱️ Timing Attacks

Timing attacks are sneakily simple ⏱️. You just watch how long a system takes to respond to different inputs—and voilà, you’re already gathering clues. Depending on the time taken, you might figure out whether the algorithm went down **path A** or **path B**. It’s like algorithmic detective work, but instead of a magnifying glass, you’re holding a stopwatch. Let’s see how this plays out with an example:
 

> Imagine a PIN checker that blinks a red LED when the password is wrong. Sounds innocent enough, right?  
> But if you measure how long it takes to respond, you might start noticing patterns—like which character it trips up on.  
> ⏰ A little timing can go a long way.  
>   
> Let’s check out a sample graph to see this in action:  
> ![PIN timing graph](./resources/timing_graph.png)
> <br>Take a look at the two graphs. In the first one, the error LED lights up *faster* than in the second. That means the system figured out something was wrong *sooner*.  
> 
> Let’s say the correct password is **WOMP**. The first graph might be from an input like **BOOP**, where the very first character is wrong—so the system bails out early. The second graph might be from something like **WONT**, where it gets through a few correct characters before tripping on the third one.  
>   
> Sneaky, right? Just by watching the clock, we’re already narrowing down the secret.
> <br>And here’s the real hack: instead of brute-forcing through ~5000 possible passwords, timing gives us clues that cut down the search space *big time*.  
> Now we’re looking at just around **40 guesses**. Way easier, way sneakier 😎.

---

Now let’s say someone got smart and added a little **random delay** before lighting up the error LED. The password check still works the same way under the hood, but now there’s a bit of noise in the timing.

That means the delay between pressing "verify" and seeing the error light no longer gives a clean hint about which character failed. Our trusty stopwatch? A little less trustworthy now.

So if timing alone can’t always give us the answers—especially with random delays muddying the waters—it’s time to shift our attention to something more subtle. Let’s see what the device might be revealing through its power consumption.

## ⚡ Simple Power Analysis (SPA)

Simple Power Analysis involves looking at how a device’s power consumption changes as it processes different inputs. It’s not just visual inspection—we might use basic operations like subtraction or filtering to highlight repeating patterns in the power trace.  
> **Power trace** is nothing but a graph of a device’s power consumption recorded over time while it performs a task.

These patterns can reveal things like which instructions are being run or even the data being handled, depending on how leaky the implementation is. It’s a surprisingly effective technique, especially when the device isn’t trying too hard to hide its secrets.  

Let’s look at how a sample power trace would appear for a microcontroller running a password checker algorithm. Here's the graph for a **single character**:

![Single Character Power Consumption](./resources/spa_trace_single.png)

Well okay… that doesn’t really tell us much, does it?

Now let’s try plotting it for **all possible characters** in the first position of the password:

<div style="display: inline-block;">
  <img src="resources/spa_trace_multiple.png" alt="Multiple Characters Power Consumption" width="500">
  <div style="text-align: center; width: 100%; font-size: 0.9em; color: gray;">
    <em>Power traces of Multiple Character Guesses in one plot</em>
  </div>
</div>

Wait, why does it look like there are only two graphs—**brown** and **gray**?

Turns out, it’s actually a bunch of traces plotted together. But since the power consumption for many characters is identical, their traces get **superimposed**, forming that collective *brown* shade.

So what can we learn from this?

> Since the trace for **one character** stands out from the rest, it must be the **correct character**.  
> **BINGO!**

Wait—before we pop the confetti, there’s a small catch.

The graphs we saw earlier were just tiny snapshots of the device’s power usage while checking one character. But in reality? The full power trace looks more like this:

<div style="display: inline-block;">
  <img src="resources/spa_trace_full.png" alt="Full Trace SPA" width="500">
  <div style="text-align: center; width: 100%; font-size: 0.9em; color: gray;">
    <em>Full trace of two character guesses, orange and blue</em>
  </div>
</div>

Whoa. That looks nothing like the neat little plots we saw earlier—it’s on a completely different scale. So how do we find meaningful differences in this chaos?

Simple: **Subtraction.**

We subtract each power trace from a reference "wrong" power trace. This helps highlight the subtle differences in power consumption between correct and incorrect guesses.

But wait—what character can we always count on to be wrong?

Take a second to think about it...

<details>
  <summary><strong>Answer</strong></summary>

  ```0x00``` — the null byte. It's rarely ever the correct character in a password, so it’s a great baseline for comparison.

</details>

Once we subtract the reference wrong trace from all the others, we get a much clearer picture:

<div style="display: inline-block;">
  <img src="resources/spa_final.png" alt="Power trace subtracted" width="600">
  <div style="text-align: center; width: 100%; font-size: 0.9em; color: gray;">
    <em>Power traces after subtraction with reference "wrong" trace</em>
  </div>
</div>

> Now it's much easier to spot the outlier — the **gray** trace.  
> That’s the correct character for this position in the password. Let’s go. We’re finally cracking passwords with simple power analysis.
 
 ---

> But it's not just password cracking, we can also apply it on multiple encryption algorithms like **ECDSA**, **RSA** etc. You can learn more abouut them in *Chapter 8* of the [book](./resources/ebin.pub_the-hardware-hacking-handbook-breaking-embedded-security-with-hardware-attacks-1nbsped-159327.pdf).

But can we find a case where **Simple Power Analysis** doesn’t work?

Absolutely. In fact, there are several—but one of the most common is when we're dealing with encryption algorithms where the execution path stays the **same**, no matter what the input data is.

No branching, no obvious differences in power consumption.  
Tough break, right?

So how do we even begin to crack something like that?

We turn to a very important property of binary data: [**Hamming Weight**](https://en.wikipedia.org/wiki/Hamming_weight).  

Let’s take a look at the next type of power analysis that makes clever use of this idea—**Differential Power Analysis**.

## 📉 Differential Power Analysis

Sometimes the power traces don’t give away anything obvious—even though secrets are still leaking beneath the surface.  
This is where **Differential Power Analysis (DPA)** shines.  
Instead of relying on a single trace, DPA looks at a *bunch* of power traces—thousands, usually—and uses statistical techniques to find tiny differences between groups. These differences can leak information about the data being processed, even if the algorithm’s control flow doesn’t vary at all.  
It’s like picking up on a whisper in a crowd—not easy, but totally doable if you listen carefully (and run the right math).

Before we go on to the actual concepts of DPA, let's understand **Hamming Weight**.

---

### Hamming Weight

The **Hamming Weight** of a binary value is simply the number of bits that are set to 1. For example, the Hamming Weight of `0b1101` is 3.  
This concept becomes highly relevant in power analysis because many digital circuits—especially **CMOS-based** ones—consume power in proportion to the number of bits switching from 0 to 1 or vice versa.  
As a result, operations involving data with different Hamming Weights can leave behind subtle patterns in power consumption—patterns that, with the right approach, can reveal information about the data being processed.

> If you want to explore power consumption in CMOS circuits further, here is a great [resource](./resources/cmos_power_consumption.pdf).

---

### But what kind of algorithms does DPA actually work on?

One of the most widely studied—and vulnerable—algorithms is the **Advanced Encryption Standard**, or [AES](https://www.geeksforgeeks.org/computer-networks/advanced-encryption-standard-aes/).

It’s a symmetric encryption algorithm used practically everywhere: from securing Wi-Fi to encrypting files on your laptop.  
And yes, it's very much within the scope of DPA techniques.

So before we go further, let’s take a quick dive into how **AES encryption** works and where the potential leakage points are hiding.


### Advanced Encryption Standad (AES)






