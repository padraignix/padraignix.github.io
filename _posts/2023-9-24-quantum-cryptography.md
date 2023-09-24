---
layout:     post
title2:     Post-Quantum Cryptography - Shor's Algorithm in Action
title:      Post-Quantum Cryptography - Shor's Algorithm in Action
date:       2023-09-24 09:00:00 -0400
summary:    A post going over the theoretical implementation of Shor's algorithm as well as covering a previous Qiskit challenge that involved practically implementing Shor's to factor the number 35. This will lay the groundwork to further posts exploring the need for post-quantum cryptography and implementations so far.
categories: [quantum-computing]
thumbnail:  microchip
math: true
keywords:   quantum computing,pqc,cryptography,post quantum,qiskit
thumbnail:  https://blog.quantumlyconfused.com/assets/quantum-computing/pqc-shor/banner.jpg
canon:      https://blog.quantumlyconfused.com/quantum-computing/2022/09/21/quantum-cryptography/
tags:
 - quantum-computing
 - quantum challenge
 - quantum cryptography
 - post-quantum cryptography
---

<h1>Introduction</h1>

<p>
<img width="70%" height="70%" src="{{ '/assets/quantum-computing/pqc-shor/banner.jpg' | relative_url }}">
</p>

<p>Recently IBM released a free <a href="https://developer.ibm.com/blogs/quantum-introducing-course-post-quantum-cryptography">Practical introduction to quantum-safe cryptography</a> course that helps <i>"serves as a primer on the foundational concepts in quantum-safe cryptography"</i>. While the course itself I have been finding useful and do recommend, it reminded me of a previous Qiskit Event, Spring 2021, where I first practically experimented with Shor's algorithm. To my surprise, despite having the notebook uploaded to my <a href="https://github.com/padraignix/quantum-challenges/blob/main/ibm-iqc2021/ex2.ipynb">personal repo</a> I had not written up a summary of the event as I had done with previous ones.</p>

<p>This post will not only rectify the previous omission, but will serve as the kick off to a mini-series covering more in-depth Post-Quantum Cryptography. I want to start with this post, highlighting the fundamental reasons as to why we need PQC, explaining how Shor's algorithm threatens the underpinnings of our security premises today. A second post will cover the current efforts to introduce "Quantum-Safe Cryptography", covering loosely the mathematical concepts as well as standardization efforts currently underway with NIST. Lastly, I want to cover in a third post the experimentation and practical implementation of PQC in some of our day-to-day utilities. I  took a look an initial look in March 2022 at experimental branches of SSH developed by <a href="https://openquantumsafe.org/">Open Quantum Safe</a>, however at the time I ended up choosing an algorithm that did not end up being selected by NIST. With some of the latest announcements by companies such as <a href="https://arstechnica.com/security/2023/09/signal-preps-its-encryption-engine-for-the-quantum-doomsday-inevitability/">Signal</a> announcing the latest PQC integration, I want to revisit those initial experiments and implement an iteration using some of the approaches selected by NIST.</p>

<p>Alright, enough planning, let's jump into learning how Shor's algorithm can break our current security!</p>

<h1>Qiskit Event</h1>

As with previous Qiskit challenges, the Spring 2021 event was broken up into several notebooks with different focus areas. One of those, explored Shor's algorithm, ultimately leading individuals to implementing a prime-factorization of the number 35. While this won't directly allow us to break some of the modern day cryptography, since keys can be in the range of 2048-4096 bits, it demonstrates the inevitability that breaking the underpinning of our current security structures is no longer a theoretical possibility, it has moved to a problem of engineering. As quantum computers develop and get "better", while at the same time more research is done expanding on Shor's original implementation, it becomes a matter of "when", not "if" current cryptographic implementations can be broken.

Before we jump too far down the rabbit hole let us go over the basics - Shor's algorithm and covering the Qiskit notebook on how it was implemented practically.

<h1>Shor's Algorithm</h1>

As mentioned in the intro section, my corresponding notebook for the event can be found at my <a href="https://github.com/padraignix/quantum-challenges/blob/main/ibm-iqc2021/ex2.ipynb">personal repo</a>. What I would add on is that since the event some of the Qiskit documentation has changed with the site's restructuring. If you follow the notebook and try to go to the Qiskit documentation for Shor's it has changed to <a href="https://learn.qiskit.org/course/ch-algorithms/shors-algorithm">Shor's Algorithm</a>. 

<h2>Algorithm Summary</h2>

<p>Shor's algorithm ultimately solves the problem of period finding. This becomes important as factoring, the underpinning of current classical cryptography, can be turned into a period finding problem with polynomial time. This means if we can use Shor's to find an efficient way to find the period of $$f(x) = a^x\mod N$$ it can be used to find the prime factors used for encryption such as RSA.</p>

<p>What the Spring 2021 challenge had us implement was a quantum phase estimation of <i>U</i>, where the approach is $$U\left|y\right> = \left|ay\mod N\right>$$</p>

<h2>Creating the Unitary</h2>

<p>Since our challenge boils down to solve $$13y\mod 35$$ we already "know" how to solve this. We could mentally go through an iteration of multiples of 13 until we get back to our original state. In this case we can do it simply through classical confirmation, but the structure of how we setup this period is important, as we will be modelling it as a quantum unitary whose approach can be scaled where classically this becomes infeasible.</p>

<p>
<a href="/assets/quantum-computing/pqc-shor/unitary1.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/pqc-shor/unitary1.png' | relative_url }}"></a>
</p>

{% highlight python %}
from qiskit import QuantumCircuit
from qiskit import QuantumRegister, QuantumCircuit
c = QuantumRegister(1, 'control')
t = QuantumRegister(2, 'target')
cu = QuantumCircuit(c, t, name="Controlled 13^x mod 35")

# WRITE YOUR CODE BETWEEN THESE LINES - START

cu.ccx(c,t[0],t[1])
cu.cx(c,t[0])
{% endhighlight %}

<p>
<a href="/assets/quantum-computing/pqc-shor/unitary2.png" data-lightbox="image2"><img src="{{ '/assets/quantum-computing/pqc-shor/unitary2.png' | relative_url }}"></a>
</p>

<p>Now with <i>U</i> complete, we can build out the remainder of the circuits needed. In this case we already know the period (r) is 4. This can be considered "cheating" as in a real example we would not know this ahead of time, however for this implementation we can proceed with the assumption the information is known. As a quick side-note since this mentally toyed with me during the challenge, there are references in the notebook itself for implementations with no assumption, showing that this particular implementation can be more efficient since we have some of the details already, however it is not mandatory to solve.</p>

<p>
<a href="/assets/quantum-computing/pqc-shor/ux.png" data-lightbox="image3"><img src="{{ '/assets/quantum-computing/pqc-shor/ux.png' | relative_url }}"></a>
</p>

<h2>U2</h2>

<p>
<a href="/assets/quantum-computing/pqc-shor/2b.png" data-lightbox="image4"><img src="{{ '/assets/quantum-computing/pqc-shor/2b.png' | relative_url }}"></a>
</p>

{% highlight python %}
c = QuantumRegister(1, 'control')
t = QuantumRegister(2, 'target')
cu2 = QuantumCircuit(c, t)

# WRITE YOUR CODE BETWEEN THESE LINES - START

cu2.cx(c,t[1])
{% endhighlight %}

<p>
<a href="/assets/quantum-computing/pqc-shor/2b-circuit.png" data-lightbox="image5"><img src="{{ '/assets/quantum-computing/pqc-shor/2b-circuit.png' | relative_url }}"></a>
</p>

<h2>U4</h2>

<p>
<a href="/assets/quantum-computing/pqc-shor/2c.png" data-lightbox="image5"><img src="{{ '/assets/quantum-computing/pqc-shor/2c.png' | relative_url }}"></a>
</p>

Which in our case is a no-op, there is no change to the states therefore this unitary is "nothing".

<h2>Final Circuit</h2>

Putting all the components together, we get:

{% highlight python %}
# Code to combine your previous solutions into your final submission
cqr = QuantumRegister(3, 'control')
tqr = QuantumRegister(2, 'target')
cux = QuantumCircuit(cqr, tqr)
solutions = [cu, cu2, cu4]
for i in range(3):
    cux = cux.compose(solutions[i], [cqr[i], tqr[0], tqr[1])
cux.draw('mpl')
{% endhighlight %}

<p>
<a href="/assets/quantum-computing/pqc-shor/cux.png" data-lightbox="image6"><img src="{{ '/assets/quantum-computing/pqc-shor/cux.png' | relative_url }}"></a>
</p>

<h1>Finding r</h1>

<p>So we have a circuit. Great, what now? The total circuit will give us the output of <i>s/r</i> where <i>r</i> is the period of the original modulus function and <i>s</i> is a random integer between 0 and <i>r-1</i>.</p>

{% highlight python %}
from qiskit.circuit.library import QFT
from qiskit import ClassicalRegister
# Create the circuit object
cr = ClassicalRegister(3)
shor_circuit = QuantumCircuit(cqr, tqr, cr)

# Initialise the qubits
shor_circuit.h(cqr)

# Add your circuit
shor_circuit = shor_circuit.compose(cux)

# Perform the inverse QFT and extract the output
shor_circuit.append(QFT(3, inverse=True), cqr)
shor_circuit.measure(cqr, cr)
shor_circuit.draw('mpl')
{% endhighlight %}

<p>
<a href="/assets/quantum-computing/pqc-shor/fullcircuit.png" data-lightbox="image7"><img src="{{ '/assets/quantum-computing/pqc-shor/fullcircuit.png' | relative_url }}"></a>
</p>

<p>Then running the circuit we get the results of 0,2,4,8.</p>

<p>
<a href="/assets/quantum-computing/pqc-shor/fullresut.png" data-lightbox="image8"><img src="{{ '/assets/quantum-computing/pqc-shor/fullresult.png' | relative_url }}"></a>
</p>

<p>But wait, didn't you say it should be giving us <i>s/r</i>? That doesn't seem to be the right numbers. There's one last part we didn't account for. The circuit itself is actually giving us $$2^n \cdot \frac{s}{r}$$ so we need to adjust appropriately.</p>

<p>
<a href="/assets/quantum-computing/pqc-shor/r.png" data-lightbox="image9"><img src="{{ '/assets/quantum-computing/pqc-shor/r.png' | relative_url }}"></a>
</p>

And the 1/2 can be thought of as 2/4 here, giving us properly s/r with an r of 4. Now, again, we knew this ahead of time because we cheated by doing it mentally first. But this illustrates that the circuit can get us a proper answer, which is the main point of going through the steps to generate the circuit and confirm the output.

<h2>Getting the factors of 35</h2>

<p>So how do we get the factors from r? There is a high chance that at least one of the greatest common divisor between N (in our case 35) and either $$a^\frac{r}{2}-1$$ or $$a^\frac{r}{2}+1$$ is a factor of N. Since GCD is easily computed classically we can find that as:</p>

<p>
<a href="/assets/quantum-computing/pqc-shor/gcd.png" data-lightbox="image10"><img src="{{ '/assets/quantum-computing/pqc-shor/gcd.png' | relative_url }}"></a>
</p>

<p>In this case it's easy to see that both ended up being the factors of 35, however were only one of them a true factor, then finding the other would simply be dividing N by the factor found. Congratulations, you just factored a number using Shor's approach!

<h1>Summary</h1>

<p>I was debating including the following "simplified" approach to Shor's earlier in the post, but I felt that including this now, after you've gone through the practical implementation should allow for deeper understanding. The mathematical explanation comes from the latest Developer course on Post-Quantum Cryptography listed in the introduction, and I highly suggest going through that to reinforce the concepts seen here as well as provide different implementation overviews.</p>

<p>
<a href="/assets/quantum-computing/pqc-shor/shor.png" data-lightbox="image11"><img src="{{ '/assets/quantum-computing/pqc-shor/shor.png' | relative_url }}"></a>
</p>

<p>Hopefully this walkthrough helped cut through the underlying explanation of how Shor's algorithm provides a threat to the current classical approach to our security processes. What we will cover in a next post is the answer to Shor's - how the industry is evolving to address security in a "quantum-safe" approach, and the standardization efforts around it. We are the latest precipice of security and computing paradigms and learning more about both the threats and future approaches is fundamental to ensure we are keeping our environments safe. It is never too early to start learning and not only be ready for what comes next, but to be able to help shape it as well.</p>

<p>Thanks folks, until next time!</p>