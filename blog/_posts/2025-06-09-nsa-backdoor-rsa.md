---
layout: post
title: How the NSA Probably Backdoored RSA
description: >
  A somewhat technical and ethical commentary on the NSA backdooring of RSA via Dual_EC_DRBG.
sitemap: false
hide_last_modified: true
---

> "Even if you're not doing anything wrong, you are being watched and recorded." --Edward J. Snowden

In the annals of modern cryptography, one of the rather disturbing episodes is the NSA’s attempt to backdoor a cryptographic standard using `Dual_EC_DRBG`, a pseudorandom number generator (PRNG) that, under certain parameters, could function less like a secure random number generator and more like a master key (held, conveniently, by its creators).

This isn’t a theoretical issue. In fact, it’s confirmed. In 2013, internal documents leaked by Edward Snowden revealed that the NSA had deliberately influenced the design of `Dual_EC_DRBG` (Dual Elliptic Curve Deterministic Random Bit Generator) to make it suspectable to backdoor access, then actively pushed for it to be adopted as a standard by NIST (National Institute of Standards and Technology) and RSA Security.

## How Dual_EC_DRBG Works

At its core, `Dual_EC_DRBG` is a deterministic random bit generator based on the mathematics of elliptic curves. While elliptic curve cryptography (ECC) is widely used and considered secure (think ECDSA, ECDH), it was quite an odd choice from the start.

From a high-level:

- It operates using public elliptic curve and two points on the curve, call them $$P$$ and $$Q$$.

- The secret internal state is updated by multiplying it by $$P$$ (as in scalar multiplication over the elliptic curve), then the output is derived via multiplication with $$Q$$.

- If the relationship between $$P$$ and $$Q$$ is known (i.e., there exists a scalar $$dd$$ such that $$Q=dPQ=dP$$), then anyone with knowledge of $$dd$$ can recover the internal state from the outputs.

Think about that for a second. If you control the generation of $$Q$$ (as the NSA did), and if you knew the relationship between $$Q$$ and $$P$$, then you could very easily rewind the generator. What should be a one-way, irreversible process becomes a surveillance engineer’s wet dream.

Cloudflare actually provides an excellent breakdown showing how, with partial output and knowledge of this secret scalar $$d$$, one could recover the internal state and predict future outputs — effectively compromising all cryptographic operations that depend on this PRNG, including TLS sessions and VPN tunnels (which generally aren’t meant explicitly for security, but case in point).

## A Bit Lower

The digits of $$\pi$$ look random, but they’re predictable. That’s why we don’t use them (or any predictable sequence) as a source of randomness. Anyone who knows the algorithm can replicate it. That predictability undermines any cryptographic system relying on that randomness.

As such, PRNGs must be designed with significant caution. Most PRNGs start with some secret seed and generate outputs through some internal transformation. If the seed and internal state remain secret, the output appears random to an observer; if they’re exposed, the entire output stream can be reconstructed.

Linux’s `/dev/random` uses an internal entropy pool as its state. When a program requests random data, Linux runs that data through `SHA-1`, a one-way cryptographic hash. The idea is simple:

- Apply a one-way function (`SHA-1`) to the internal state.

- Mix the output back into the state.

- Hash unpredictable system events (e.g., mouse clicks, keypresses) and mix that in as well.

Diagrammatically:

```console
State --F--> Output
State <--G-- Output
```

Where:

- F = `SHA-1`

- G = `SHA-1` + XOR + remix

So long as `SHA-1` is one-way, and the entropy is high, the design is robust. Even if you see the output, the internal state stays hidden.

With elliptic curve cryptography (ECC), we hinge on the hardness of the elliptic curve discrete logarithm problem (ECDLP).

Given a point $$P_1$$​ and a number $$n$$, we can compute:
$$Q = n \cdot P_1$$, which is repeated elliptic curve “dotting.”

Getting $$Q$$ is easy. Going backwards to find $$n$$ given $$P_1$$ and $$Q$$ is hard -- this is the ECDLP. Thus, we have a natural one-way function:

- Input: a number $$n$$

- Output: the $$x$$-coordinate of $$ Q = n \cdot P_1$$

Here’s an analogy. Let’s say you’re playing billiards. You shoot the ball ($$P_1$$) $$n$$ times in a special elliptic curve table. The final position $$Q$$ is easy to see, but reverse-engineering the number of shots ($$n$$) is quite hard.

## Introducing a Backdoor

Let's say we want to design a PRNG using ECC. Pick two points $$P_1$$ and $$P_2$$ on an elliptic curve. Construct two one-way functions:

- $$F(n) = x \text{-coordinate of } (n \cdot P_1)$$ &nbsp;

- $$G(n) = x \text{-coordinate of } (n \cdot P_2)$$ &nbsp;

If $$P_1$$ and $$P_2$$ are independent, this looks secure. But here's the trick:

- Choose $$P_2 = s \cdot P_1$$, where $$s$$ is secret.

- Now $$F$$ and $$G$$ are *not* independent. Rather, they're related through some unknown multiplier $$s$$.

So when the PRNG does:

- Output: $$Q = n \cdot P_1$$

- Update state: $$S = n \cdot P_2 = n \cdot (s \cdot P_1) = s \cdot (n \cdot P_1) = s \cdot Q$$

If you know $$s$$, and you observe $$Q$$, you can compute the new state $$S$$ directly. The entire internal state is exposed. That breaks everything. This is exactly how `Dual_EC_DRBG` works.

### Troubling

`Dual_EC_DRBG` was published by NIST (with contributions from the NSA) and recommended as a PRNG standard. It uses two fixed elliptic curve points $$P_1$$ and $$P_2$$. However:

1. The values of $$P_1$$ and $$P_2$$ were never justified.
2. There was no verifiable process (e.g., hashing known seeds) used to generate them.
3. If $$P_2 = s \cdot P_1$$, and $$s$$ is known only to the designer, then `Dual_EC_DRBG` contains a backdoor.

This was implemented in 2013 using OpenSSL. Funnily enough, a patent describing this exact technique as a method of **key escrow** was filed in 2006!

## RSA

The story takes an even more troubling turn when we consider how this became widely adopted. In 2006, NIST (under NSA influence) standardized `Dual_EC_DRBG`, despite **known** criticisms in the academic community by cryptographers about its suspicious structure and performance inefficiencies.

Here's the real kicker: RSA Security allegedly accepted a $10 million contract from the NSA to make `Dual_EC_DRBG` the default PRNG in its flagship cryptographic library, BSAFE.

RSA obviously denied wrongdoing, stating they were simply following NIST standard. But the damage was done. NIST eventually withdrew the standard in 2014.


## Surveillance Creep
This is most definitely *not* an isolated incident. It's significantly broader pattern of deliberate sabotage and blatant manipulation by the NSA:

- Program BULLRUN: Leaked documents detailed efforts to insert vulnerabilities into commercial encryption systems, undermine standards, and tamper with the design of international cryptographic protocols.

- Compromised Hardware: Reports that the NSA intercepted shipments of Cisco routers to implant surveillance tools. It's a complete nation-state supply chain poisoning.

- And surveillance en masse under PRISM and XKeyScore: Warrantless access to user data from major tech companies, dragnet-style collections of emails, search history, and real-time communication (often without *any* meaningful oversight).

So, is this an agency tasked with defending encryption or breaking it? Really makes you wonder

## Why Should You Care?

Mass surveillance and the NSA's frankly egregious desire to inject Americans with it is a very, very clear invasion of basic human rights in addition to a plethora of American laws (including the very constitution).

It doesn't matter whether you're a criminal or not, the government is recording you (note the initial quote). The politicians say they need more tools for surveillance so they can catch terrorists, and yet there's been *no* substantial proof that this has actually helped in any way. In fact, the NSA, FBI, and other three letter agencies have been denounced for substantial failures to catch terrorists despite the massive breach of privacy operation they run. In fact, the FBI has been known to actively find and manipulate unstable individuals, encourage them to commit acts of terrorism (and given them the materials to do so), just to promptly arrest them and say, "Look! We got someone!"

As for our beloved constitution, it simply doesn't mean *anything* to the NSA. It's clear violation of our First, Fourth, AND Fifth Amendment rights. 
- Freedom of Speech and Association? Hell no! Here's a chilling effect though... And your journalists, activists, and religious organizations? Yeah, we're prosecuting and censoring them.

- Protection from Unreasonable Searches and Seizures? You'd think you would have to have a warrant to conduct this surveillance... that's a utterly stupid idea! Why would we need warrants to collection every single bit of information about you without probable cause? In fact, this practice has been ruled *unconstitutional* by multiple courts.

- Due Process? You didn't need that anyway. There is **no chance** for individuals to challenge their inclusion in watchlists or databases. Programs like no-fly lists, built from secret intelligence, deny individuals liberty with minimal resources.

Some people say, "Well, if you've done nothing wrong, you've got nothing to hide." The government is granting itself overwhelming power to peer into any individual's life whenever it wants. They don't discriminate. Ever heard of Nazi Germany and how they subverted society to push their agenda and normalize repression? Mass surveillance wasn't and isn't used to protect people, it's used to control them. You might recall Pastor Martin Niemöller's poem, "First They Came":
> First they came for the Communists <br>
> And I did not speak out<br>
> Because I was not a Communist<br>
> Then they came for the Socialists<br>
> And I did not speak out<br>
> Because I was not a Socialist<br>
> Then they came for the trade unionists<br>
> And I did not speak out<br>
> Because I was not a trade unionist<br>
> Then they came for the Jews<br>
> And I did not speak out<br>
> Because I was not a Jew<br>
> Then they came for me<br>
> And there was no one left<br>
> To speak out for me<br>

The Nazis were able to prosecute and execute millions of people simply because they normalized it, used surveillance to silence opposition, and nobody lifted a finger. It's the same as saying, "I shouldn't have to worry because I'm not doing anything wrong." You're allowing the government to do whatever they want, whenever they want, and you expect them not to slowly start smothering your rights &mdash; because they most definitely well. Sure, some minority group is suddenly deported or some activist is put in prison for opposing state policy, but that won't matter to you because they're not you, right? ... Until the day they are, and by then, it's too late. 

Rights don't only exist for the guilty, they exist to protect the innocent from unchecked power. Once you surrender those rights, history shows that the governments rarely give them back.