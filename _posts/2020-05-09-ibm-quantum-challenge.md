---
layout:     post
title2:      Quantum Computing - Qiskit Quantum Challenge 2020 Writeup
title:     Qiskit Quantum Challenge 2020
date:       2020-5-09 08:00:00 -0400
summary:    Overview and walkthrough of the IBM Qiskit 4th anniversary Quantum Challenge excercises and my solutions.
categories: [quantum-computing]
thumbnail:  microchip
math: true
keywords:   quantum computing,qiskit,quantum,python,ibm,challenge,qubits,quantum gates,logic gates,unitary
thumbnail:  https://github.com/padraignix.png
canon:      https://blog.quantumlyconfused.com/quantum-computing/2020/05/09/ibm-quantum-challenge/
tags:
 - qiskit
 - quantum-computing
 - quantum-gates
 - unitary
 - error-correction
 - bb84
---

<h1>Introduction</h1>

<p>Four challenges to complete in four days - IBM sure knows how to tug at my heartstrings. Initially unaware of the challenge it was only ironically as I was preparing to deliver a Career Week Quantum Computing presentation with a colleague did <a href="https://quantum-computing.ibm.com/challenges/4anniversary">IBM's Quantum Challenge</a> come up in conversation. A quick look at the introductory page and I put on an extra pot of coffee as I knew I would be making up for loss time of the first two days.

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge-intro.PNG' | relative_url }}">
</p>

<p>The challenges were a perfect difficulty level for my current level of QC knowledge. Challenges 1 and 2 were introductory and helped get back in the groove of working with Qiskit, adding an element of error correction that I was not familiar with previously. Challenge 3 provided an opportunity to work with Quantum Cryptography, specifically a portion of BB84. I was already familiar with BB84 theory and this was more of a challenge on how to get it implemented practically. Lastly, challenge 4 is where we jumped off the deep end. Not only were the concepts involved things I had not heard of yet, the Qiskit specific functionality required to proceed further was also foreign to me at the start. This challenge pushed my initial boundaries and I'm happy to say with some determination and a lot of reading I was able to complete it with my own inneficient approach.</p> 

<p>Your <i>"final score"</i> being determined by how efficient your solution was also helped drive the community to not only solve the challenge, but push themselves to improve their solutions further. I took the opportunity and remaining time available to have a few conversations and learn what to look for as far as optimizations. I'm happy to say that while I did not manage to get near the lowest scores seens during the challenge I was able to further reduce the overall cost of my circuit.</p>

<p>Despite the challenge now being closed all material is available at <a href="https://github.com/qiskit-community/may4_challenge_exercises">Qiskit's community github repo</a>. Follow along as I go over the four challenges using some of the material covering how I ended up solving them.</p>

<h1>Challenge 1 - Basic Quantum Circuits</h1>

<p>Not so much a challenge as an introduction to using Qiskit, how quantum gates operate, and visualizing qubit states being run through these gates. The first exercise focused on interacting with a single qubit while the second exercise touched on multi-qubit interactions. As long as you followed the provided information, performed each portion step by step, and kept track of what each gate's functionality was this section was rather quick.</p>

<p>Each gate's functionality used throughout the exercise can be condensed down to:</p>

{% highlight python %}
qc.x(0)          # bit flip
qc.y(0)          # bit and phase flip
qc.z(0)          # phase flip
qc.h(0)          # superpostion
qc.s(0)          # quantum phase rotation by pi/2 (90 degrees)
qc.sdg(0)        # quantum phase rotation by -pi/2 (90 degrees)
qc.cx(c,t)       # controlled-X (= CNOT) gate with control qubit c and target qubit t
qc.cz(c,t)       # controlled-Z gate with control qubit c and target qubit t
qc.swap(a,b)     # SWAP gate that swaps the states of qubit a and qubit b
{% endhighlight %}

<p>Then if we take a look at an example of the type of questions were asked throughout the exercise:</p>

<p>
<codeblock><i><b>
I.i) Let us start by performing a bit flip. The goal is to reach the state |1⟩ starting from state |0⟩.
</b></i></codeblock>
</p>
<p>Most of the code was already provided for this question. The challenge was to conceptually understand what was being asked and apply the correct gate. In this case, if we applied a Hadamard gate to the qubit we would end up performing a bit flip.</p>

{% highlight python %}
def create_circuit():
    qc = QuantumCircuit(1)
    qc.x(0)
    return qc
# check solution
qc = create_circuit()
state = statevec(qc)
check1(state)
plot_state_qsphere(state.data, show_state_labels=True, show_state_angles=True) 
{% endhighlight %}

<p>Sure enough, visualizing the state with the provided code we see we were successful.</p>

<p>
<img width="50%" height="50%" src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge1-answer.PNG' | relative_url }}">
</p>

<h1>Challenge 2 - Error Correction</h1>

<p>The second challenge offered an opportunity to do a couple different things. Firstly, the entire premise that a physical implementation of QC would have inherent erorrs introduced requires us to interact with the physical hardware. Secondly, the challenge taught us how to introduce error correction to our specific data run and answer questions related to the data set.</p>

<p>The theory behind the error correction was explained well using the following:</p>

<p>
<i>We start by creating a set of circuits that prepare and measure each of the \(2^{n}\) basis states, where 𝑛 is the number of qubits. For example, 𝑛=2 qubits would prepare the states |00⟩, |01⟩, |10⟩, and |11⟩ individually and see the resulting outcomes. The outcome statistics are then captured by a matrix 𝑀, where the element \(𝑀_{𝑖𝑗}\) gives the probability to get output state |𝑖⟩ when state |𝑗⟩ was prepared.
</i></p>

<p>Some code snippets were provided to help select the least busy backend and run a calibration test for our baseline.</p>

{% highlight python %}
# find the least busy device that has at least 5 qubits
backend = least_busy(provider.backends(filters=lambda x: x.configuration().n_qubits >= num_qubits and 
                                   not x.configuration().simulator and x.status().operational==True))
{% endhighlight %}

<p>Then we leverage this backend to run our callibration matrix generation, while keeping an eye on the job status.</p>

{% highlight python %}
# run experiments on a real device
shots = 8192
experiments = transpile(meas_calibs, backend=backend, optimization_level=3)
job = backend.run(assemble(experiments, shots=shots))
print(job.job_id())
%qiskit_job_watcher
{% endhighlight %}

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge2-jobrun.PNG' | relative_url }}">
</p>

<p>With the job run complete, we can then visualize the latent errors seen in our backend. We will use this result to apply our upcoming corrections.</p>

{% highlight python %}
# get measurement filter
cal_results = job.result()
meas_fitter = CompleteMeasFitter(cal_results, state_labels, circlabel='mcal')
meas_filter = meas_fitter.filter
#print(meas_fitter.cal_matrix)
meas_fitter.plot_calibration()
{% endhighlight %}

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge2-errormapping.PNG' | relative_url }}">
</p>

<p>As mentioned in the Challenge 2 notebook <i>"The goal of this exercise is to create a calibration matrix 𝑀 that you can apply to noisy results (provided by us) to infer the noise-free results"</i>. With our callibration matrix created we can proceed through the questions. First we visualize the provided "noisy" data, apply our error correction, then finally answer the questions by choosing which graphs resembled our corrected data sets.</p>

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge2-application.PNG' | relative_url }}">
</p>

<h1>Challenge 3 - BB84</h1>

<p>Now that we covered QC basics with challenges 1 and 2, time to move on to some practical application! The challenge 3 notebook starts by going over the theory of the BB84 Quantum Key Distribution algorithm as we will be tasked with implementing parts of it as our challenge.</p>

<h3>Theory</h3>

<p>We can break down the BB84 algorithm into three parts for this challenge.</p>

<p>1). Alice chooses <i><b>k</b></i> bits that represent the actual information she wants to encode, <i><b>b</b></i> bases that will represent how she encodes her information bit.
<br><br>
For \(b_i=0\) (i.e., if the \(i^{th}\) bit is zero), Alice encodes the \(i^{th}\) qubit in the standard {|0⟩,|1⟩} basis, while for \(b_i=1\), she encodes it in the {|+⟩,|−⟩} basis, where:
$$
\begin{align*}
\left|+\right> := \frac{1}{\sqrt{2}}(\left|0\right> + \left|1\right>)
\end{align*}
$$
and
 $$
\begin{align*}
\left|-\right> := \frac{1}{\sqrt{2}}(\left|0\right> - \left|1\right>)
\end{align*}
$$ 
then sends these qubits to Bob.
</p>

<p>2). Bob choses his own set of <i><b>b</b></i> and measures the qubits sent by Alice with these bases, storing his own set of <i><b>k</b></i>.</p>

<p>3). Alice and Bob compare bases. For any \(i^{th}\) base that does not match, they will discard the corresponding <i><b>k</b></i>.</p>

<p>Once these three steps are completed, both Alice and Bob will have a shared set of information that will serve as their secret key.</p>

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge3-bb84.PNG' | relative_url }}">
</p>

<p>For this challenge parts 1 & 2 in the above graph has been provided to us. We will first need to implement our own function to measure the "prepared qubits" that Alice sends us. Secondly, given that Alice has provided her bases, iterate through our measurements and discard those that do not match. Lastly, with our secret key with Alice defined we need to use that key to decrypt a secret message she sent us.</p>

<h3>Solution</h3>

<p>For the first portion we are tasked with creating <i>bob_measure_qubit</i> in the below code portion.</p>

{% highlight python %}
%matplotlib inline

# Importing standard Qiskit libraries
import random
from qiskit import execute, Aer, IBMQ
from qiskit.tools.jupyter import *
from qiskit.visualization import *
from may4_challenge.ex3 import alice_prepare_qubit, check_bits, check_key, check_decrypted, show_message

# Configuring account
provider = IBMQ.load_account()
backend = provider.get_backend('ibmq_qasm_simulator')

# Initial setup
random.seed(84) # do not change this seed, otherwise you will get a different key

# This is your 'random' bit string that determines your bases
numqubits = 100
bob_bases = str('{0:0100b}'.format(random.getrandbits(numqubits)))

def bb84():
    print('Bob\'s bases:', bob_bases)

    # Now Alice will send her bits one by one...
    all_qubit_circuits = []
    for qubit_index in range(numqubits):

        # This is Alice creating the qubit
        thisqubit_circuit = alice_prepare_qubit(qubit_index)

        # This is Bob finishing the protocol below
        bob_measure_qubit(bob_bases, qubit_index, thisqubit_circuit)

        # We collect all these circuits and put them in an array
        all_qubit_circuits.append(thisqubit_circuit)
        
    # Now execute all the circuits for each qubit
    results = execute(all_qubit_circuits, backend=backend, shots=1).result()
    
    # And combine the results
    bits = ''
    for qubit_index in range(numqubits):
        bits += [measurement for measurement in results.get_counts(qubit_index)][0]
    return bits

# Here is your task
def bob_measure_qubit(bob_bases, qubit_index, qubit_circuit):              
    #### to be filled out ####

bits = bb84()
print('Bob\'s bits: ', bits)
check_bits(bits)
{% endhighlight %}

<p>To solve this portion we will need to iterate through Bob's bases, and when the base is 0, we are measuring the qubit in the Z axis so leave it as is, and if the base is 1, we are measuring the qubit in the X axis, and need to apply a Hadamard gate to the circuit before measuring.</p>

{% highlight python %}
def bob_measure_qubit(bob_bases, qubit_index, qubit_circuit):              
    if bob_bases[qubit_index] == '0': #measuring in Z
        qubit_circuit.measure(0,0)
    if bob_bases[qubit_index] == '1': #measuring in X
        qubit_circuit.h(0)
        qubit_circuit.measure(0,0)
{% endhighlight %}

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge3-measure.PNG' | relative_url }}">
</p>

<p><i>Side note</i>: I lost several hours at this point and exemplified my ingrained classical vs. quantum thinking. I had the BB84 theory right and was applying the gate properly, however when I ran the code my bits would return all 0s. It took me entirely too longer to realize I was forgetting to measure the circuit as part of the iteration. Harsh lesson.</p>

<p>Next, we need create a function to compare bases and keep only the measured bits where they match.</p>

{% highlight python %}
key = ''

for basis_index in range(numqubits):
    print("Bob: ",bob_bases[basis_index])
    print("Alice: ",alice_bases[basis_index])
    if bob_bases[basis_index] == alice_bases[basis_index]:
        key += bits[basis_index]
        print(key)
    print("==========")
    
check_key(key)
{% endhighlight %}

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge3-key.PNG' | relative_url }}">
</p>

<p>With our key established we are provided Alice's encrypted message. While iterating through the message and our key I decided to take an "easier" approach and keep the logic using strings rather than covert them to digits. I wrote my own simple variant of XOR while iterating through our strings.</p>

{% highlight python %}
m = '0011011010100011101000001100010000001000011000101110110111100111111110001111100011100101011010111010111010001'\
    '1101010010111111100101000011010011011011011101111010111000101111111001010101001100101111011' # encrypted message

decrypted = ''

for index in range(len(m)):
    if m[index] == "0":
        if key[index % 50] == "0":
            decrypted += "0"
        if key[index % 50] == "1":
            decrypted += "1"
    if m[index] == "1":
        if key[index % 50] == "0":
            decrypted += "1"
        if key[index % 50] == "1":
            decrypted += "0"

print(decrypted)

check_decrypted(decrypted)
{% endhighlight %}

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge3-decrypt.PNG' | relative_url }}">
</p>

<p>The final part of this challenge was to understand the decrypted message. The challenge notebook let us know this was morse code encrypted and provided a conversion chart to get the string readable. I'll be perfectly honest and after the several hours lost forgetting to measure the circuit I did not want to strain my brain further trying to write up a snippet of code to do the conversion - so since it was a small amount of text I did it by hand in 5 minutes.</p>

{% highlight shell %}
r   .-.     10110100
e   .       100
d   -..     11010100
d   -..     11010100
i   ..      10100
t   -       1100
.   .-.-.-  1011010110101100
c   -.-.    11010110100
o   ---     1101101100
m   --      1101100
/   -..-.   1101010110100
r   .-.     10110100
/   -..-.   1101010110100
m   --      1101100
a   .-      101100
y   -.--    110101101100
4   ....-   101010101100
q   --.-    110110101100
u   ..-     10101100
a   .-      101100
n   -.      110100
t   -       1100
u   ..-     10101100
m   --      11011
{% endhighlight %}

<p>Which brings us to our answer for challenge 3 - <i><b>reddit.com/r/may4quantum</b></i></p>

<h1>Challenge 4 - Circuit Decomposition</h1>

<p>Here is where the piano was dropped on our collective heads. Reading through the brief a first time I reached the coding section and stopped a moment to scratch my head - I wasn't entirely sure what I was reading let alone what they were asking us to accomplish. This is where the community came to the rescue and I was able to boil down the challenge and a few related details that were not provided initially in the notebook. The following is a conversation seen in the slack channel during the challenge.</p>

<blockquote>
do we take the U gate and remake it only using single qubit rotations and cnots?<br><br>
then whats the V and what does it mean to have a cost smaller than 1600?<br><br>
The gate you make is called might be an approximation of U. Since it's not identical, we give it a different name: V.<br><br>
U is the reference Unitary and V is the approximation, cost is calculated on the number of u3 and cx gates<br><br>
CX gates cost 10, and U3s cost 1<br><br>
so U gives these numbers, and we need to figure out a V which gives (very closely) the same numbers?<br><br>
yes, that's the goal
</blockquote>

<p>Thank you random stranger for asking the question before me! Now at least there is a clear understanding of _what_ we are trying to accomplish. With that, let us start looking at the details.</p>

<h3>Initial Investigation</h3>

<p>We are given a unitary U:</p>

{% highlight python %}
from may4_challenge.ex4 import get_unitary

U = get_unitary()

print("U has shape", U.shape)
#print(U)
{% endhighlight %}

<p>The shape of U is shown as (16,16) and if we print out U directly it gives us a 16x16 matrix with various values. I didn't think of this originally, however it would have been smart to plot U and notice the symmetry.</p>

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge4-chart.PNG' | relative_url }}">
</p>

<p>There are two important Qiskit functionalities that we will need to use for my approaches - isometry and transpile.</p>

<p>Firstly <a href="https://qiskit.org/documentation/stubs/qiskit.circuit.QuantumCircuit.iso.html">QuantumCircuit.iso</a> attaches an arbitrary isometry from m to n qubits to a circuit. In particular, this allows to attach arbitrary unitaries on n qubits (m=n) or to prepare any state on n qubits (m=0). In our particular situation we can set our unitary U to our quantum circuit qc.</p>

<p>Secondly <a href="https://qiskit.org/documentation/stubs/qiskit.compiler.transpile.html">transpile</a> allows us to transpile, or break down, one or more circuits into a desired target. Since our ultimate goal with this challenge is to create a circuit using only u3 and cx gates we can use transpile to reduce our target circuit to those specific gates.</p>

<p>With the information we have we can now code up a small sample and see where we are at.</p>

{% highlight python %}
qc.iso(U, [0,1,2,3], [])
V = transpile(qc,basis_gates=['u3','cx'],optimization_level=3)
{% endhighlight %}

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge4-1676.PNG' | relative_url }}">
</p>

<p>At least our theory works and we have something to work with. It looks like even with the highest optimization level set on transpile we still have too high of a complexity cost. We will need to see what we can do to make that more efficient.</p>

<h3>Inefficient Solution</h3>

<p>While I covered the solution for challenge 3 above, some might be wondering why a subreddit? If we follow the trail of crumbs we are pointed to the solution of challenge 1 as a hint for this final challenge - which came out as "Hadamard".</p>

<p>There was a lot of reading and attempts at understanding the theory at this point but my (potentially fallible) logic was that by multiplying a 16x16 Hadamard matrix H to the unitary U, we would have a circuit section denoted HU. The inverse of a Hadamard gate is closely related to itself, in this case \(HH^{T} = HH = I\), or the Identity matrix, so if we were to add another H circuit this should create a circuit of \(HHU = IU\). In theory, we should have a reduced circuit that approximates U.</p>

{% highlight python %}
qc = QuantumCircuit(4)
qc1 = QuantumCircuit(4)

# Create 16x16 Hadamard
H = scipy.linalg.hadamard(16)/4

# HU
qc.iso(np.matmul(U,H), [0,1,2,3], [])
V=transpile(qc,basis_gates=['u3','cx'],optimization_level=3)

# H
qc1.iso(H, [0,1,2,3], [])
V1=transpile(qc1,basis_gates=['u3','cx'],optimization_level=3)

# HHU -> IU
V2 = V1 + V

V2.draw()
{% endhighlight %}

<p>Our circuit ends up looking like the following.</p>

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge4-hhu.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge4-hhu-eval.PNG' | relative_url }}">
</p>

<p>Excellent, our theory ended up working! This isn't the most efficient solution as the lowest cost recorded was 45, however I'm proud that I was able to get an initial solution.</p>

<h3>Optmization</h3>

<p>While our HHU attempt worked because we added two separate circuits together, we should be able to reduce the complexity cost by doing the work prior to invoking isometry - at least that is my assumption in theory. I used <a href="https://docs.scipy.org/doc/scipy-0.14.0/reference/generated/scipy.linalg.hadamard.html">scipy.linalg.hadamard</a> in my initial solution, however I quickly found out it did not work well with this second approach. I needed to switch from generating a 16x16 Hadamard matrix (which by documentation should also be using <a href="https://en.wikipedia.org/wiki/Hadamard_matrix#Sylvester's_construction">Sylverster's construction</a>) to using <a href="https://docs.scipy.org/doc/numpy/reference/generated/numpy.kron.html">numpy.kron</a> to compute my own 16x16 Kronecker product. If we apply this approach we start off with.</p>

{% highlight python %}
h = np.matrix([[1,1], [1,-1]]) / np.sqrt(2)
h2x2 = np.kron(h,h)
h4x4 = np.kron(h2x2, h2x2)
huh = h4x4 * U * h4x4

qc.iso(huh, [0,1,2,3], [])

qc = transpile(qc,basis_gates=['u3','cx'],optimization_level=3)
{% endhighlight %}

<p>This ended up working as far as creating a more optimized circuit, however when we go to check our solution we see something off.</p>

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge4-kron-fail.PNG' | relative_url }}">
</p>

<p>Hum... so our cost was drastically reduced, but the solution did not produce the right approximation of U.</p>

<p>Once again the community came to the rescue at this point. I did not fully understand it originally, and to be perfectly honest I am still spending time reading through the theory, however I came to learn that since we modified our unitary to be in Hadamard space, we would need to apply H gates on our qubits before and after the circuit to represent our original unitary. This brings our solution to the following.</p>

{% highlight python %}
h = np.matrix([[1,1], [1,-1]]) / np.sqrt(2)
h2x2 = np.kron(h,h)
h4x4 = np.kron(h2x2, h2x2)
huh = h4x4 * U * h4x4

qc.h([0,1,2,3])
qc.iso(huh, [0,1,2,3], [])
qc.h([0,1,2,3])

qc = transpile(qc,basis_gates=['u3','cx'],optimization_level=3)
{% endhighlight %}

<p>Then if we draw this circuit out and evaluate it.</p>

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge4-kron-circuit.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge4-kron-eval.PNG' | relative_url }}">
</p>

<p>Success! Much happier with this result and although I would love to spend more time trying to get further optimization, I believe this is a good stopping point. There are quite a few concepts and theory I need to bring myself up to speed with before I can truly understand how to start approaching further improvements.</p>

<h1>Conclusion & Next Steps</h1>

<p>This Quantum Challenge was a great continuation of my Quantum journey. As I mentioned in my previous <a href="{{ site.baseurl }}{% link _posts/2020-02-09-quantum-computing-helloworld.md %}">QC Helloworld</a> post I was already planning on diving into BB84 implementation. Challenge 3 offered me that chance and I definitely want to continue with implementing the algorithm more completely.</p>

<p>Within a day of the challenge closing IBM issued the digital badge for completing the four exercises. This is a nice addition to the overall challenge in addition to the knowledge gained.</p>

<p>
<a href="https://www.youracclaim.com/badges/bdfce883-83fa-41ca-84f1-9d50a527ee0c">
<img src="{{ '/assets/quantum-computing/quantum-challenge-2020/quantum-challenge-badge.PNG' | relative_url }}">
</a>
</p>

<p>Additionally, not that I needed further reinforcement, but this challenge highlighted just how much I don't know in the space. While I have a basic Linear Algebra background and a Comp Sci mindset, while reading some of community's the most efficient approaches to challenge 4 there were quite a few concepts I did not have enough understanding to apply myself. These will be topics of further reading in the near future.</p>

<p>Hopefully you all learned something while reading along with my writeup. As always thanks folks, until next time!</p>