---
layout:     post
title2:     Quantum Computing - Qiskit Quantum Challenge Fall 2020
title:      Qiskit Quantum Challenge Fall 2020
date:       2020-12-10 08:02:00 -0400
summary:    Overview and walkthrough of IBM Qiskit Fall 2020 Quantum Final Challenge excercise with a focus on theoretically solving and circuit decomposition for reduced complexity cost. By the end of the improvement iterations the successful circuit resulted in a 12k complexity.
categories: [quantum-computing]
thumbnail:  microchip
math: true
keywords:   quantum computing,qiskit,quantum,python,ibm,challenge,qubits,quantum gates,logic gates,grover,qram
thumbnail:  https://blog.quantumlyconfused.com/assets/quantum-computing/quantum-challenge-fall2020/catbanner.jpg
canon:      https://blog.quantumlyconfused.com/quantum-computing/2020/12/10/ibm-quantum-challenge-fall2020/
tags:
 - qiskit
 - quantum-computing
 - quantum challenge
 - grover
 - qram
---

<h1>Introduction</h1>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/catbanner.jpg" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/catbanner.jpg' | relative_url }}"></a>
</p>

<p>For the final week of IBM's Quantum Challenge Fall 2020 we had to put the Grover and qRAM experience we had gained in the first two weeks to the test. Using the knowledge we had developed we were provided with a new "Asteroid Clearing" problem which ended up being similar to the classical <a href="https://en.wikipedia.org/wiki/Eight_queens_puzzle">n-queens</a> problem.</p>

<p>Apart from solving the problem solely using Quantum Circuits, a veritable feat in itself, the competitive nature of this challenge was in continuously optimizing the cost of your circuit to the lowest possible configuration while still ensuring the circuit solves the full problem space.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/week3-mainintro.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/week3-mainintro.png' | relative_url }}"></a>
</p>

<p>This post will cover a few different aspects. Initially it will cover my theoretical solution and corresponding circuit. Afterwards I will cover each improvement iteration, what I targetted and implemented in the improvement, and the subsequent lowered score. The final improvement section will also cover qSphere diagram flows to help elaborate on the theory leveraged.</p>

<p>This walkthrough is less about using Grover's in any specifically special manner, in fact I have some questions and doubts about the final implementation based on empirical evidence gathered through this challenge that I will highlight in the final section. The focus is really the journey from an initial solution, then how without structurally changing the overall flow repeating iterations until I reached a circuit complexity of only <code>10%</code> the original solution. All while still solving the same original problem space.</p>

<p>The Qiskit team's challenge notebook can be found on their <a href="https://github.com/qiskit-community/IBMQuantumChallenge2020/blob/main/exercises/week-3/final_en.ipynb">Github Page</a>. Without further delay, let's jump into the challenge!</p>

<h1>Challenge Problem Statement</h1>

<p>The problem statement started with our favourite Dr. Ryoko stuck in the multiverse with a quick description of what would end up being the challenge we need to solve.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/intro3.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/intro3.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/intro3-asteroids.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/intro3-asteroids.png' | relative_url }}"></a>
</p>

<p>We were provided an example that illustrates the problem space in terms of asteroids on a 4x4 board.</p>

<p>
<a width="75%" height="75%" href="/assets/quantum-computing/quantum-challenge-fall2020/week3-board1.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/week3-board1.png' | relative_url }}"></a>
</p>

<p>Subsequently we were also provided an example of how a board could be solvable with three laser beam shots.</p>

<p>
<a width="75%" height="75%" href="/assets/quantum-computing/quantum-challenge-fall2020/week3-board2.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/week3-board2.png' | relative_url }}"></a>
</p>

<p>Once the mechanics of the problem understood we are given the full problem statement and a set of rules that must be adhered to for our solutions to be valid. Ultimately the problem boiled down to being given 16 different 4x4 boards where we use Grover's algorithm to determine the single board that is not solvable (clearable) in three laser beam shots.</p>

<!--<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/week3-board3.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/week3-board3.png' | relative_url }}"></a>
</p> -->

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/week3-problemstatement.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/week3-problemstatement.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/week3-rules.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/week3-rules.png' | relative_url }}"></a>
</p>

<p>With the initial background information understood, let's start to solve the problem!</p>

<h1>Theoretical Solution</h1>

<p>Before writing a single line of code I spent time making sure I conceptually knew how to solve the problem. The first 36 hours of the final week was spent breaking down the various boards and attempting to find clever ways of determining the solution. Starting the overall qRAM structure from the previous week.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/week3-qram.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/week3-qram.png' | relative_url }}"></a>
</p>

<p>The initial thought was to approach the challenge similarly to the second week's lightsout problem - map the boards to the qram addresses, flesh out the 16 qubits representing the board, and codify the mechanics of a laser beam.</p>

<p>There were two inherent problems with this approach. Firstly the limitation on qubits we could leverage was 28. If we were to proceed with this approach we would need 16 for the board representation, 8 for the different laser beam shots (4 vertical, 4 horizontal), 4 for the qram addresses, and finally 1 for the oracle to help amplify winner states. This brings us up to 29 qubits, outside of our limitation. The second limitation in this approach is in the mechanics itself. In the second week's lightsout problem a button push was itself a reversible action. When a button was pushed, any adjoining lights that were on would be turned off and vice-versa. In this final asteroid problem the mechanics were of the nature of "if on, turn off, and stay off", representing destroying the asteroid. Unfortunately trying to leverage something like CNOT gates as we did previously does not work in this context and ultimately the approach would result in a non-reversible mechanic.</p>

<p>After spending some time trying to understand the nature of the problem and what differentiates a solvable and unsolvable board it started making sense when thought of using Linear Algebra. Conceptually a 4x4 board is not solvable in three or less laser beam shots if it has 4 asteroids on distinct rows and columns to each other. Another way of thinking of this is that the 4 asteroids must create a <a href="https://en.wikipedia.org/wiki/Rank_(linear_algebra)">Matrix Rank</a> of 4. Alternatively you can also think of it as these asteroids creating a board having a <a href="https://en.wikipedia.org/wiki/Determinant">Determinant</a> not equal to zero, or that it resembles an <a href="https://en.wikipedia.org/wiki/Identity_matrix">Identity Matrix</a>. In the context of our problem space as long as there were 4 asteroids that resembled the following, we knew that regardless of the additional two asteroids the board is not solvable in three laser beam shots.</p>

<p>
<a width="75%" height="75%" href="/assets/quantum-computing/quantum-challenge-fall2020/week3-board4.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/week3-board4.png' | relative_url }}"></a>
</p>

<p>Now that we have a direction we need to figure out how to leverage it. As stated in the rules we are not allowed to preprocess any of the input. We must take the board date and put it into the circuit before any processing is done. This makes it slightly more complicated since we could in theory use <a href="https://en.wikipedia.org/wiki/Determinant">classical algorithms</a> to calculate the Determinant. Unfortunately since we do not have the concept of multiplication in Quantum Computing we would need to define our own counters and functions. While not impossible, in fact we worked with adders and counters over the last two weeks, the overall complexity seemed rather high for our problem space. In hindsight since we were essentially working with only 1s and 0s this could have been achieved using a simplified classical function, however I decided to approach it from a different direction.</p>

<p>Two distinct eurka moments led to the first implementation in code. First, we do not "care" where the beams are firing. As long as a board fits a particular configuration we know by the nature of the board itself it is not solvable in three, regardless of where the beams are fired. The second relevation happened while iterating through the various configurations that a 4x4 matrix can have to resemble an Identity Matrix. I quickly realized there were only 24 different combinations, much less that I initially thought there would be. With only 24 permutations this puts us in a good position to leverage Qiskit's <a href="https://qiskit.org/textbook/ch-algorithms/grover.html#sudoku">Grover Sudoku clause approach</a> where each clause is one of the 24 permutations of a 4x4 Identity Matrix defining that the board has a rank 4 and non-zero determinant. An example clause for the board above would be <code>[0,5,10,15]</code> which also represents the Identity matrix itself. Time to start coding!</p>

<h1>119k - Initial Solution</h1>

<p>The theory was essentially broken down into the four parts of the qRAM diagram above.</p>

* qRAM the 16 board addresses (<code>0000...1111</code>) in a similar manner we did in the given examples
* Run through the various clauses - flipping the oracle if a board matches a configuration
* Uncompute the qRAM address
* Diffuse the board address, amplifying the non-solvable address

<p>For brevity sake I will try and not include the full code dump and focus on the appropriate snipped sections. My final notebook is available at my <a href="https://github.com/padraignix/ibm-quantum-challenge-fall2020">Github repo</a> for those interested.</p>

<h2>qRAM</h2>

<p>We were given the defined problem_set of boards.</p>

{% highlight python %}
problem_set = \
    [[['0', '2'], ['1', '0'], ['1', '2'], ['1', '3'], ['2', '0'], ['3', '3']],
    [['0', '0'], ['0', '1'], ['1', '2'], ['2', '2'], ['3', '0'], ['3', '3']],
    [['0', '0'], ['1', '1'], ['1', '3'], ['2', '0'], ['3', '2'], ['3', '3']],
    [['0', '0'], ['0', '1'], ['1', '1'], ['1', '3'], ['3', '2'], ['3', '3']],
    [['0', '2'], ['1', '0'], ['1', '3'], ['2', '0'], ['3', '2'], ['3', '3']],
    [['1', '1'], ['1', '2'], ['2', '0'], ['2', '1'], ['3', '1'], ['3', '3']],
    [['0', '2'], ['0', '3'], ['1', '2'], ['2', '0'], ['2', '1'], ['3', '3']],
    [['0', '0'], ['0', '3'], ['1', '2'], ['2', '2'], ['2', '3'], ['3', '0']],
    [['0', '3'], ['1', '1'], ['1', '2'], ['2', '0'], ['2', '1'], ['3', '3']],
    [['0', '0'], ['0', '1'], ['1', '3'], ['2', '1'], ['2', '3'], ['3', '0']],
    [['0', '1'], ['0', '3'], ['1', '2'], ['1', '3'], ['2', '0'], ['3', '2']],
    [['0', '0'], ['1', '3'], ['2', '0'], ['2', '1'], ['2', '3'], ['3', '1']],
    [['0', '1'], ['0', '2'], ['1', '0'], ['1', '2'], ['2', '2'], ['2', '3']],
    [['0', '3'], ['1', '0'], ['1', '3'], ['2', '1'], ['2', '2'], ['3', '0']],
    [['0', '2'], ['0', '3'], ['1', '2'], ['2', '3'], ['3', '0'], ['3', '1']],
    [['0', '1'], ['1', '0'], ['1', '2'], ['2', '2'], ['3', '0'], ['3', '1']]]
{% endhighlight %}

<p>At which point we can reference and load the appropriate board if the address matches.</p>

{% highlight python %}
def week3_ans_func(problem_set):
    ##### build your quantum circuit here
    ##### In addition, please make it a function that can solve the problem even with different inputs (problem_set). We do validation with different inputs. 
    address = QuantumRegister(4, name='address')
    aux = QuantumRegister(4, name='aux')
    oracle = QuantumRegister(1, name='oracle')
    oracle2 = QuantumRegister(1, name='oracle2')
    tile_qubits = QuantumRegister(16, name='tile')
    address_cbit = ClassicalRegister(4,name='address-c')
    #classical2 = ClassicalRegister(16,name='tile-c')
    qc = QuantumCircuit(address,tile_qubits,aux,oracle,oracle2,address_cbit)

    ##init qubits
    qc.h(address)
    
    #############################
    ##### QRAM Init
    #############################
    # address 0 - 0000
    qc.x([address[0],address[1],address[2],address[3]])
    for asteroid in problem_set[0]:
                num = int(asteroid[0]) * 4
                num += int(asteroid[1])
                qc.mct(address,tile_qubits[num])
    qc.x([address[0],address[1],address[2],address[3]])
    # address 1 - 0001
    qc.x([address[0],address[1],address[2]])
    for asteroid in problem_set[1]:
                num = int(asteroid[0]) * 4
                num += int(asteroid[1])
                qc.mct(address,tile_qubits[num])
    qc.x([address[0],address[1],address[2]])
    # address 2 - 0010
    qc.x([address[0],address[1],address[3]])
    for asteroid in problem_set[2]:
                num = int(asteroid[0]) * 4
                num += int(asteroid[1])
                qc.mct(address,tile_qubits[num])
    qc.x([address[0],address[1],address[3]])
    # address 3 - 0011
    qc.x([address[0],address[1]])
    for asteroid in problem_set[3]:
                num = int(asteroid[0]) * 4
                num += int(asteroid[1])
                qc.mct(address,tile_qubits[num])
    qc.x([address[0],address[1]])
    # address 4 - 0100
... <snip>...
{% endhighlight %}

<p>Continuing down for all 16 addresses. This qRAM section is also used for the qRAM uncompute portion of the flow.</p>

<h2>Grover Oracle</h2>

<p>Once we had the board loaded we iterate through the 24 clauses and flip the oracle if set.</p>

{% highlight python %}
    for i in range(1):
    
        ################################
        ##### START of - Grover
        ################################
        
        ## Iterate through clause list of potentials - Crude version for determining rank4 matrix
        
        ################ 0's
        #0,5,10,15
        qc.mct([tile_qubits[0],tile_qubits[5],tile_qubits[10],tile_qubits[15]],aux[0]) 
        qc.cx(aux[0],oracle)
        qc.ccx(aux[0],oracle,oracle2)
        qc.mct([tile_qubits[0],tile_qubits[5],tile_qubits[10],tile_qubits[15]],aux[0]) 
        #0,5,11,14
        qc.mct([tile_qubits[0],tile_qubits[5],tile_qubits[11],tile_qubits[14]],aux[0])
        qc.cx(aux[0],oracle)
        qc.ccx(aux[0],oracle,oracle2)
        qc.mct([tile_qubits[0],tile_qubits[5],tile_qubits[11],tile_qubits[14]],aux[0])
        #0,6,9,15
        qc.mct([tile_qubits[0],tile_qubits[6],tile_qubits[9],tile_qubits[15]],aux[0])
        qc.cx(aux[0],oracle)
        qc.ccx(aux[0],oracle,oracle2)
        qc.mct([tile_qubits[0],tile_qubits[6],tile_qubits[9],tile_qubits[15]],aux[0])
        #0,6,11,13
        qc.mct([tile_qubits[0],tile_qubits[6],tile_qubits[11],tile_qubits[13]],aux[0])
        qc.cx(aux[0],oracle)
        qc.ccx(aux[0],oracle,oracle2)
        qc.mct([tile_qubits[0],tile_qubits[6],tile_qubits[11],tile_qubits[13]],aux[0])
        ...<snip>...
{% endhighlight %}

<p>Of note here is the double oracles. Since a clause is 4 asteroids and there are 6 asteroids on the board we need to take into consideration a board where two clauses are satisfied by the 6 asteroids. Example below where the board satisfies both <code>[0,5,10,15]</code> as well as <code>[1,4,10,15]</code>.</p>

<p>
<a width="75%" height="75%" href="/assets/quantum-computing/quantum-challenge-fall2020/week3-board3.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/week3-board3.png' | relative_url }}"></a>
</p>

<p>If we were to just flip the oracle when a clause is met, any board that satisfies two clauses would ultimately turn off the oracle. Since we know that <code>at most</code> two clauses can be satisfied by a single, 6-asteroid board, we only need to consider a double oracle situation, and not >2 potential clause satisfying boards.</p>

<h2>Diffusion</h2>

<p>Standard qRAM address diffusion we leveraged in the previous week's examples.</p>

{% highlight python %}
    qc.h(address)
    qc.x(address)
    qc.h(address[-1])
    qc.mct(address[:-1], address[-1])
    qc.h(address[-1])
    qc.x(address)
    qc.h(address)
{% endhighlight %}

<h2>Submission</h2>

<p>With the code setup I submitted the job and put on another pot of coffee while it processed. After about ~20 minutes I was happy to report that it successfully worked!</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/119ksol.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/119ksol.png' | relative_url }}"></a>
</p>

<p>This is just the start however, as now the improvement iterations can start!</p>

<h1>53k - Ancillary Qubits</h1>

<p>The first improvement iteration was conceptually simple. Originally we executed straight MCTs on the address/oracle portions. Let's add some ancillary qubits help bring that cost down since we have a few to work with within the 28 qubit limitation.</p>

{% highlight python %}
    # address 0 - 0000
    qc.x([address[0],address[1],address[2],address[3]])
    for asteroid in problem_set[0]:
                num = int(asteroid[0]) * 4
                num += int(asteroid[1])
                qc.mct(address,tile_qubits[num], anc, mode="basic")
    qc.x([address[0],address[1],address[2],address[3]])
{% endhighlight %}

{% highlight python %}
        ## Iterate through clause list of potentials - Crude version for determining rank4 matrix
        ################ 0's
        #0,5,10,15
        qc.mct([tile_qubits[0],tile_qubits[5],tile_qubits[10],tile_qubits[15]],aux[0], anc, mode="basic") 
        qc.cx(aux[0],oracle)
        qc.ccx(aux[0],oracle,oracle2)
        qc.mct([tile_qubits[0],tile_qubits[5],tile_qubits[10],tile_qubits[15]],aux[0], anc, mode="basic")
{% endhighlight %}

<p>At this point I also want to introduce a nifty little code snippet. With a second notebook open I was able to leverage the below to do cost evaluations of the circuit without executing the full ~20 min runs. This helped immensely when I wasn't sure if the approach was worth persuing or not.</p>

{% highlight python %}
from qiskit.transpiler.passes import Unroller
from qiskit.transpiler import PassManager

pass_ = Unroller(['u3', 'cx'])
pm = PassManager(pass_)
new_circuit = pm.run(qc) 
odict = new_circuit.count_ops()
print('u3:'+str(odict['u3'])+' ''cx:'+str(odict['cx'])+' '+'total:'+str(odict['u3']+10*odict['cx']))    
{% endhighlight %}

<p>With this we can throw portions of circuit into this notebook an breakdown the cost of each. With our introduced ancillary qubits we get to a total complexity cost of 53k. Already less than half of the original solution.</p>

{% highlight python %}
u3:8307 cx:4502 total:53327
Qram - 20512
Oracle - 12144
Qram - 20512
Diffuse - 159
{% endhighlight %}

<h1>22k - MCMT/Gray Code</h1>

<p>This particular iteration is my favourite mostly of how it came to me. During the middle of the week I was going over the qRAM and oracle portions trying to identify where I could simplify the execution. During a morning shower I came to a sudden realization that for the qRAM portions I am running an MCT gate for EACH of the six asteroids. If we know the address is valid, we should be able to find a way cut down to a simpler comparison. I quickly mocked up the pseudocode on the shower door and made sure my logic made sense. I was then able to implement the code improvement later in the day.</p>

{% highlight python %}
    # address 0 - 0000
    qc.x([address[0],address[1],address[2],address[3]])
    qc.mct(address,aux[0], anc, mode="basic")
    for asteroid in problem_set[0]:
                num = int(asteroid[0]) * 4
                num += int(asteroid[1])
                qc.cx(aux[0],tile_qubits[num])
    qc.mct(address,aux[0], anc, mode="basic")          
    # address 1 - 0001
    qc.x(address[3])
    qc.mct(address,aux[0], anc, mode="basic")
    for asteroid in problem_set[1]:
                num = int(asteroid[0]) * 4
                num += int(asteroid[1])
                qc.cx(aux[0],tile_qubits[num])
    qc.mct(address,aux[0], anc, mode="basic")  
{% endhighlight %}

<p>By pulling out MCT gates from within the loop and using an additional aux qubit to determine when the address is triggered we can replace 6 MCT gates with 2 MCT and 6 CX gates.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/22ksol.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/22ksol.png' | relative_url }}"></a>
</p>

<p>The observant reader will also notice I changed the qRAM address method as well. Instead of running X gates before and after to reset we are able to leverage <a href="https://en.wikipedia.org/wiki/Gray_code">Gray Code</a> ordering to reduce a very small amount of gates for the qRAM address iteration.</p>

<p>Sadly, despite the eureka shower moment and finding a manual approach to reducing I soon found the <a href="https://qiskit.org/documentation/stubs/qiskit.circuit.library.AND.html?highlight=mcmt#qiskit.circuit.library.AND.mcmt">MCMT</a> (Multi-Control Multi-Target) gate. Using MCMTs ever so slightly increased efficiency, by a total of 12 cost per qRAM address, depite resulting in much uglier code.</p>

{% highlight python %}
    # address 0 - 0000
    qc.x([address[0],address[1],address[2],address[3]])    
    qc.mcmt(qc.cx,address,[tile_qubits[int(problem_set[0][0][0])*4+int(problem_set[0][0][1])],tile_qubits[int(problem_set[0][1][0])*4+int(problem_set[0][1][1])],tile_qubits[int(problem_set[0][2][0])*4+int(problem_set[0][2][1])],tile_qubits[int(problem_set[0][3][0])*4+int(problem_set[0][3][1])],tile_qubits[int(problem_set[0][4][0])*4+int(problem_set[0][4][1])],tile_qubits[int(problem_set[0][5][0])*4+int(problem_set[0][5][1])]],anc,mode='basic')
    # address 1 - 0001
    qc.x(address[3])
    qc.mcmt(qc.cx,address,[tile_qubits[int(problem_set[1][0][0])*4+int(problem_set[1][0][1])],tile_qubits[int(problem_set[1][1][0])*4+int(problem_set[1][1][1])],tile_qubits[int(problem_set[1][2][0])*4+int(problem_set[1][2][1])],tile_qubits[int(problem_set[1][3][0])*4+int(problem_set[1][3][1])],tile_qubits[int(problem_set[1][4][0])*4+int(problem_set[1][4][1])],tile_qubits[int(problem_set[1][5][0])*4+int(problem_set[1][5][1])]],anc,mode='basic')
{% endhighlight %}

<p>At the end of this iteration I was able to further reduce the complexity cost from 192 MCT gates, to 64 MCT + 192 CX, to 32 MCMT gates, and ultimately got the cost down to 22k.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/22ksol-3gray.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/22ksol-3gray.png' | relative_url }}"></a>
</p>

<h1>51k - MCT Decomposition</h1>

<p>This iteration made the writeup not because it was more efficient. In fact it was worst than all but the original solution. It made this writeup however because of how it was implemented. While searching through various papers for academically inspired decompositions I ran into <a href="https://arxiv.org/pdf/1303.3557.pdf">Linear-Depth Quantum Circuits for n-qubit Toffoli gates with no Ancilla</a>. The interesting aspect of this paper was that it decomposed an MCT gate without ancillary qubits achieving a linear speed up to regular MCT gates. The approached was summarized in the following graph.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/phaseflipmct.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/phaseflipmct.png' | relative_url }}"></a>
</p>

<p>Feeling inspired I strived to implement it in my circuit and see how it fared in complexity cost.</p>

{% highlight python %}
qc.x([address[0],address[1],address[2],address[3]])
for asteroid in problem_set[0]:
num = int(asteroid[0]) * 4
num += int(asteroid[1])
#qc.mct(address,tile_qubits[num], anc, mode="basic")
qc.rxx(math.pi/2,address[0],address[1])
qc.rxx(math.pi/4,address[0],address[2])
qc.rxx(math.pi/8,address[0],address[3])
qc.rxx(math.pi/8,address[0],tile_qubits[num])
qc.rxx(math.pi/2,address[1],address[2])
qc.rxx(math.pi/4,address[1],address[3])
qc.rxx(math.pi/4,address[1],tile_qubits[num])
qc.rxx(math.pi/2,address[2],address[3])
qc.rxx(math.pi/2,address[2],tile_qubits[num])
qc.rxx(math.pi,address[3],tile_qubits[num])
qc.rxx(-math.pi/2,address[2],address[3])
qc.rxx(-math.pi/4,address[1],address[3])
qc.rxx(-math.pi/2,address[1],address[2])
qc.rxx(-math.pi/8,address[0],address[3])
qc.rxx(-math.pi/4,address[0],address[2])
qc.rxx(-math.pi/2,address[0],address[1])

qc.rxx(math.pi/2,address[1],address[2])
qc.rxx(math.pi/4,address[1],address[3])
qc.rxx(-math.pi/4,address[1],tile_qubits[num])
qc.rxx(math.pi/2,address[2],address[3])
qc.rxx(-math.pi/2,address[2],tile_qubits[num])
qc.rxx(-math.pi,address[3],tile_qubits[num])
qc.rxx(-math.pi/2,address[2],address[3])
qc.rxx(-math.pi/4,address[1],address[3])
qc.rxx(math.pi/2,address[1],address[2])
{% endhighlight %}

<p>I was able to get a result, however it was clearly not as efficient as what we had already achieved. Interesting nonetheless as it was the first time I used a pure academic paper and tried to implement it.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/51ksol.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/51ksol.png' | relative_url }}"></a>
</p>

<h1>20k - RCCX Oracle</h1>

<p>This next iteration leveraged the hints that the Qiskit team provided in week two - gate synthesis or equivalence transformation techniques. Specifically leveraging the <a href="https://qiskit.org/documentation/stubs/qiskit.circuit.library.RCCXGate.html#qiskit.circuit.library.RCCXGate">RCCX</a> gate example.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/week3-rccx.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/week3-rccx.png' | relative_url }}"></a>
</p>

<p>Since we were performing uncomputes throughout the circuit we should be able to break down MCT gates to their CCX/CX equivalents and then replace CCX with RCCX gates to save on cost. One additional improvement I realized at this point was related to the clause combination. Since the first two qubits of each clause pair (<code>[0,5,10,15]</code> and <code>[0,5,11,14]</code> as examples) are the same, we should be able to reduce the amount of overall comparisons done if we change only the 3rd and 4th qubits between pairs.</p>

{% highlight python %}
#0,5,10,15
#0,5,11,14
qc.rccx(tile_qubits[0],tile_qubits[5],aux[0])
qc.rccx(tile_qubits[10],tile_qubits[15],aux[1])
qc.swap(oracle,oracle2)
qc.rccx(aux[0],aux[1],oracle) #checking for 0,5,10,15
qc.swap(oracle,oracle2)
qc.rccx(tile_qubits[10],tile_qubits[15],aux[1])
qc.rccx(tile_qubits[11],tile_qubits[14],aux[1])
qc.swap(oracle,oracle2)
qc.rccx(aux[0],aux[1],oracle) #checking for 0,5,11,14
qc.swap(oracle,oracle2)
qc.rccx(tile_qubits[11],tile_qubits[14],aux[1])
qc.rccx(tile_qubits[0],tile_qubits[5],aux[0])
{% endhighlight %}

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/20ksol.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/20ksol.png' | relative_url }}"></a>
</p>

{% highlight python %}
QRAM - u3:886 cx:672 total:7606
Oracle - u3:576 cx:432 total:4896
QRAM - u3:890 cx:672 total:7606
Diffuse - u3:39 cx:12 total:159
{% endhighlight %}

<h1>13k - RCCX qRAM</h1>

<p>This particular iteration was taking the same methodology from the 20k variant and applying RCCX gates to the qRAM portions. I had to go back from the MCMT variants to MCT gates and break down the MCT to CCX/CX as above before transforming to RCCX gates.</p>

{% highlight python %}
qc.x([address[0],address[1],address[2],address[3]])
#qc.mcmt(qc.cx,address,[tile_qubits[int(problem_set[0][0][0])*4+int(problem_set[0][0][1])],tile_qubits[int(problem_set[0][1][0])*4+int(problem_set[0][1][1])],tile_qubits[int(problem_set[0][2][0])*4+int(problem_set[0][2][1])],tile_qubits[int(problem_set[0][3][0])*4+int(problem_set[0][3][1])],tile_qubits[int(problem_set[0][4][0])*4+int(problem_set[0][4][1])],tile_qubits[int(problem_set[0][5][0])*4+int(problem_set[0][5][1])]],aux,mode='basic')
qc.rccx(address[0],address[1],aux[0])
qc.rccx(address[2],address[3],aux[1])
qc.rccx(aux[0],aux[1],aux[3])
qc.cx(aux[3],tile_qubits[int(problem_set[0][0][0])*4+int(problem_set[0][0][1])])
qc.cx(aux[3],tile_qubits[int(problem_set[0][1][0])*4+int(problem_set[0][1][1])])
qc.cx(aux[3],tile_qubits[int(problem_set[0][2][0])*4+int(problem_set[0][2][1])])
qc.cx(aux[3],tile_qubits[int(problem_set[0][3][0])*4+int(problem_set[0][3][1])])
qc.cx(aux[3],tile_qubits[int(problem_set[0][4][0])*4+int(problem_set[0][4][1])])
qc.cx(aux[3],tile_qubits[int(problem_set[0][5][0])*4+int(problem_set[0][5][1])])
qc.rccx(aux[0],aux[1],aux[3])
qc.rccx(address[2],address[3],aux[1])
qc.rccx(address[0],address[1],aux[0])
# address 1 - 0001
qc.x(address[3])
#qc.mcmt(qc.cx,address,[tile_qubits[int(problem_set[1][0][0])*4+int(problem_set[1][0][1])],tile_qubits[int(problem_set[1][1][0])*4+int(problem_set[1][1][1])],tile_qubits[int(problem_set[1][2][0])*4+int(problem_set[1][2][1])],tile_qubits[int(problem_set[1][3][0])*4+int(problem_set[1][3][1])],tile_qubits[int(problem_set[1][4][0])*4+int(problem_set[1][4][1])],tile_qubits[int(problem_set[1][5][0])*4+int(problem_set[1][5][1])]],aux,mode='basic')
qc.rccx(address[0],address[1],aux[0])
qc.rccx(address[2],address[3],aux[1])
qc.rccx(aux[0],aux[1],aux[3])
qc.cx(aux[3],tile_qubits[int(problem_set[1][0][0])*4+int(problem_set[1][0][1])])
qc.cx(aux[3],tile_qubits[int(problem_set[1][1][0])*4+int(problem_set[1][1][1])])
qc.cx(aux[3],tile_qubits[int(problem_set[1][2][0])*4+int(problem_set[1][2][1])])
qc.cx(aux[3],tile_qubits[int(problem_set[1][3][0])*4+int(problem_set[1][3][1])])
qc.cx(aux[3],tile_qubits[int(problem_set[1][4][0])*4+int(problem_set[1][4][1])])
qc.cx(aux[3],tile_qubits[int(problem_set[1][5][0])*4+int(problem_set[1][5][1])])
qc.rccx(aux[0],aux[1],aux[3])
qc.rccx(address[2],address[3],aux[1])
qc.rccx(address[0],address[1],aux[0])
{% endhighlight %}

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/13ksol.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/13ksol.png' | relative_url }}"></a>
</p>

{% highlight python %}
QRAM - u3:598 cx:384 total:4438
Oracle - u3:576 cx:432 total:4896
QRAM - u3:598 cx:384 total:4438
Diffuse - u3:39 cx:12 total:159
{% endhighlight %}

<p>With what seemed like the regular statement throughout this entire process I told myself I did not believe the cost could be lowered further without fundamentally changing the structure of my approach. There were other solutions submitted in the 7k range at this point and I was almost certain they were approaching the problem differently. Based on the RCCX/CCX/CX decompositions I thought I had done as much as I could. I was wrong...

<h1>12k - Here Be Dragons</h1>

<p>Everything I went through prior to this section can be thought of with classical circuits and makes logical sense more or less as you step through the code in 2 dimensional terms. This last iteration jumps off the deep end by leveraging quantum phases. I can honestly say that the jump from 13k to 12k is where I learned the most out of this entire journey.</p>

<p>As was mentioned above in the original rules of the challenge our solution could not include any runs of transpiler to simplify the circuit design. Out of interest I took my 13k circuit above and ran it through transpiler on a second notebook to see if the cost would go down. Sure enough it resulted in sub-10k complexity costs so I knew there was still work to do.</p>

<p>Although I couldn't directly use transpiler maybe I could tactically identify smaller components of my code, investigate what transpiler is doing to it in isolation, and see if I can adapt my overall circuit using that knowledge. The first part was to understand what is happening with a RCCX gate as this will form the basis of our work. Starting at this point I also used IBM's qSphere visualization to help understand in three dimensions what was going on. There were some limitations such as restricting the visual to 5 qubits or less that required me to be crafty in reusing "input" qubits, but I can honestly say I would probably not have gotten to where I ended up with without the visual aid.</p>

<p>
<!--<a href="/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-original.gif" data-lightbox="image2"><img style="float:left" margin-right="1%" width="47%" display="inline-block" src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-original.gif' | relative_url }}"></a>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-small-rotation.gif" data-lightbox="image3"><img style="float:left" margin-right="1%" width="47%" display="inline-block" src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-small-rotation.gif' | relative_url }}"></a>
</p>
<p style="clear: both;">-->
<a href="/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-small-rotation.gif" data-lightbox="image3"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-small-rotation.gif' | relative_url }}"></a>
<p>

<p>Which translates to the following code:</p>

{% highlight python %}
# rotate aux0 and use 1st and 2nd items to flip aux0 if "on"
qc.u(pi/2,pi/4,pi,aux[0])
qc.cx(tile_qubits[det_clause[i][1]],aux[0])
qc.u(0,0,-pi/4,aux[0])
qc.cx(tile_qubits[det_clause[i][0]],aux[0])
qc.u(0,0,pi/4,aux[0])
qc.cx(tile_qubits[det_clause[i][1]],aux[0])
qc.u(pi/2,0,3*pi/4,aux[0])
{% endhighlight %}

<p>Alright time to expand to a full "single oracle" iteration. Firstly let me mock up what I'm trying to solve.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-original.gif" data-lightbox="image3"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-original.gif' | relative_url }}"></a>
</p>

<p>Now working with the original RCCX/CCX example I can expand it to a full single oracle.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-single-oracle.gif" data-lightbox="image3"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-single-oracle.gif' | relative_url }}"></a>
</p>

<p>Where the code comes out to:</p>

{% highlight python %}
#rotate oracle for eventual matching check
qc.u(pi/2,pi/4,pi,oracle)

# rotate aux0 and use 1st and 2nd items to flip aux0 if "on"
qc.u(pi/2,pi/4,pi,aux[0])
qc.cx(tile_qubits[det_clause[i][1]],aux[0])
qc.u(0,0,-pi/4,aux[0])
qc.cx(tile_qubits[det_clause[i][0]],aux[0])
qc.u(0,0,pi/4,aux[0])
qc.cx(tile_qubits[det_clause[i][1]],aux[0])
qc.u(pi/2,0,3*pi/4,aux[0])

# rotate aux1 and use 3rd and 4th items to flip aux1 if "on"
qc.u(pi/2,pi/4,pi,aux[1])
qc.cx(tile_qubits[det_clause[i][3]],aux[1])
qc.u(0,0,-pi/4,aux[1])
qc.cx(tile_qubits[det_clause[i][2]],aux[1])
qc.u(0,0,pi/4,aux[1])
qc.cx(tile_qubits[det_clause[i][3]],aux[1])
qc.u(pi/2,0,3*pi/4,aux[1])

#At this point can be a winning clause if items 0,1,2,3 are on
#Which matches up to the first of two possibilities based on 0,1 items
# i.e. first det_clause = [0,5,10,15,11,14].
# If 0,5,10,15 are 1, it's a winning clause. At this point this is where it is checked

# check if both aux satisfy, then flip oracle
qc.cx(aux[1],oracle)
qc.u(0,0,-pi/4,oracle)
qc.cx(aux[0],oracle)
qc.u(0,0,pi/4,oracle)
qc.cx(aux[1],oracle)
{% endhighlight %}

<p>So far so good! I had to tweak the rotation angles slightly as I proceeded to try and get back to an original phase. This is where the qSphere visual definitely helped keep track of where things were.</p>

<p>The last part to take into consideration is the "double oracle" approach where we are using the same first two qubits of the clause. Again let me mock up what I am doing with CCX/CX gates. Before anyone calls me out I did forget to reapply the X gates to input 1/2 during the second uncompute... but it theoretically shows what I was trying to achieve.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-original-full.gif" data-lightbox="image3"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-original-full.gif' | relative_url }}"></a>
</p>

<p>Now taking the single oracle circuit we added the second clause check in qSphere. Again with some slight angle adjustment where the visual helped.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-full-oracle-step.gif" data-lightbox="image3"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/qsphere/qsphere-full-oracle-step.gif' | relative_url }}"></a>
</p>

<p>It may be easier for folks to follow the full circuit diagram below. Because of the qSpehere qubit limitations I had to omit above the oracle swap (double clause boards). I also added annotations to clarify a run through the first det_clause of <code>[0,5,10,15,11,14]</code>.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/week3-basisgate-oracle.png" data-lightbox="image3"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/week3-basisgate-oracle.png' | relative_url }}"></a>
</p>

<p>The final oracle code ends up being:</p>

{% highlight python %}
#Prep rotate oracle for eventual matching check
qc.u(pi/2,pi/4,pi,oracle)

# rotate aux0 and use 1st and 2nd items to flip aux0 if "on"
qc.u(pi/2,pi/4,pi,aux[0])
qc.cx(tile_qubits[det_clause[i][1]],aux[0])
qc.u(0,0,-pi/4,aux[0])
qc.cx(tile_qubits[det_clause[i][0]],aux[0])
qc.u(0,0,pi/4,aux[0])
qc.cx(tile_qubits[det_clause[i][1]],aux[0])
qc.u(pi/2,0,3*pi/4,aux[0])

# rotate aux1 and use 3rd and 4th items to flip aux1 if "on"
qc.u(pi/2,pi/4,pi,aux[1])
qc.cx(tile_qubits[det_clause[i][3]],aux[1])
qc.u(0,0,-pi/4,aux[1])
qc.cx(tile_qubits[det_clause[i][2]],aux[1])
qc.u(0,0,pi/4,aux[1])
qc.cx(tile_qubits[det_clause[i][3]],aux[1])
qc.u(pi/2,0,3*pi/4,aux[1])

#At this point can be a winning clause if items 0,1,2,3 are on
#Which matches up to the first of two possibilities based on 0,1 items
# i.e. first det_clause = [0,5,10,15,11,14].
# If 0,5,10,15 are 1, it's a winning clause. At this point this is where it is checked

# check if both aux satisfy, then flip oracle
qc.cx(aux[1],oracle)
qc.u(0,0,-pi/4,oracle)
qc.cx(aux[0],oracle)
qc.u(0,0,pi/4,oracle)
qc.cx(aux[1],oracle)
#### ^ this is the final oracle flip if the first quatuplet of the clause matches
#swap oracle - accounting for double winning clause
# In reality should be rotating oracle before swapping
# however for the purposes of this problem, just swapping DOES return proper answers
# for all edge cases (double clause within the same first tuplet in det_clause)
# As there is no possibility for a double clause winner where one winner is
# the second quatuplet of det_clause & first quatuplet of the next det_clause
# because they always differ by 3 qubits, which cannot be represented by 6 asteroids.
# This fact allows us to swap oracle between the quatuplet pairs only instead of start
# of loop
qc.u(pi/2,0,3*pi/4,oracle)
qc.cx(oracle,oracle2)
qc.cx(oracle2,oracle)
#qc.cx(oracle,oracle2) ## is not required as we would only ever set oracle2 once at most
qc.u(pi/2,pi/4,pi,oracle)

## uncompute the 3rd and 4th clause winners to test the 5th and 6th (2nd quatuplet)
## unprepate aux[1] for the second set of checks. aux[0] doesn't change as it remains the same
qc.u(pi/2,pi/4,pi,aux[1])
# uncompute aux1 using 3rd/4th clause items
qc.cx(tile_qubits[det_clause[i][3]],aux[1])
qc.u(0,0,-pi/4,aux[1])
qc.cx(tile_qubits[det_clause[i][2]],aux[1])
qc.u(0,0,pi/4,aux[1])
qc.cx(tile_qubits[det_clause[i][3]],aux[1])
### start of second quatuplet check. compute aux1 if 5th/6th items are on
qc.cx(tile_qubits[det_clause[i][5]],aux[1])
qc.u(0,0,-pi/4,aux[1])
qc.cx(tile_qubits[det_clause[i][4]],aux[1])
qc.u(0,0,pi/4,aux[1])
qc.cx(tile_qubits[det_clause[i][5]],aux[1])
# prep aux1 and bring it back in line with aux0 for the check
qc.u(pi/2,0,3*pi/4,aux[1])

# Second oracle check. Now using the new aux[1]
# If both aux0,1 are on based on new phi angle, flip oracle
qc.cx(aux[1],oracle)
qc.u(0,0,-pi/4,oracle)
qc.cx(aux[0],oracle)
qc.u(0,0,pi/4,oracle)
qc.cx(aux[1],oracle)
# Now that both oracle checks are done, rotate oracle back, preserving "flip" if so
qc.u(pi/2,0,3*pi/4,oracle)

### time to clean up and uncompute previous steps - for both aux this time
# Start with aux0 
qc.u(pi/2,pi/4,pi,aux[0])
qc.cx(tile_qubits[det_clause[i][1]],aux[0])
qc.u(0,0,-pi/4,aux[0])
qc.cx(tile_qubits[det_clause[i][0]],aux[0])
qc.u(0,0,pi/4,aux[0])
qc.cx(tile_qubits[det_clause[i][1]],aux[0])
qc.u(pi/2,0,3*pi/4,aux[0])
# Uncompute aux1
qc.u(pi/2,pi/4,pi,aux[1])
qc.cx(tile_qubits[det_clause[i][5]],aux[1])
qc.u(0,0,-pi/4,aux[1])
qc.cx(tile_qubits[det_clause[i][4]],aux[1])
qc.u(0,0,pi/4,aux[1])
qc.cx(tile_qubits[det_clause[i][5]],aux[1])
qc.u(pi/2,0,3*pi/4,aux[1])

#### At this point both aux qubits are completely uncomputed and oracle is flipped
#### if either of the clauses are on. Preserved through rotation and oracle swap at the start
{% endhighlight %}

<p>A submission later and I managed to get down to 12k as a result!</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/12ksol.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/12ksol.png' | relative_url }}"></a>
</p>

<p>With that I was able to crack the top 25 submissions by close of the competition.</p>

<p>
<a href="/assets/quantum-computing/quantum-challenge-fall2020/ibmresults.png" data-lightbox="image1"><img src="{{ '/assets/quantum-computing/quantum-challenge-fall2020/ibmresults.png' | relative_url }}"></a>
</p>

<h1>Conclusion & Recap</h1>

<p>What a ride. When comparing this to <a href="{{ site.baseurl }}{% link _posts/2020-05-09-ibm-quantum-challenge.md %}">IBM's May event</a> it was almost a night and day difference in difficulty. With that came added time to truly deep dive and I appreciated the opportunity to spend time (trying) to understand more fundamentally than "make it work". With that said there were a few things that bothered me throughout the final challenge.</p>

<p>Firstly, I'm still not entirely sure if this was a coincidence or if there is a more fundamental reason as to why, but the careful observer might have noticed that I never initially set the negative phase to the two oracles. Regardless of if I set/unset the oracles phases (by using X/H gates) or did not, with a very small tweak I ended with same correct result, with different levels of amplification. I'd like to dig a bit deeper and understand if this was just a fluke, or if there some deeper meaning.</p>

<p>Secondly the issues and restrictions with the grader were frustrating. If the grader wasn't working properly that is one thing, technical issues happen. The more frustrating aspect was having to constantly doubt myself that my approach was "valid". I still don't fully know either! The rules were not necessarily clear, and the restrictions on what we could do and not do limited my willingness to be more creative in certain aspects.</p>

<p>Overall I really enjoyed the event. I learned a ridiculous amount over the three weeks! Even after the event closed I was still looking at my code and having discussions with other participants to continue that knowledge momentum. At the end of the day this is what it is all about - improving ourselves and pushing the envelop forward. Back in May I barely understood what was happening and managed to hobble through to completion. This time around, only a few months later, I understood enough to start diving into academic work and trying to apply it practically. I'm looking forward to dive deeper into certain topics I barely scraped the surface of and applying the new knowledge when the next Qiskit event rolls around!</p>

<p>For anyone who is interested I put up my challenge notebooks on my <a href="https://github.com/padraignix/ibm-quantum-challenge-fall2020">Github Repo</a>. 

<p>Thanks folks, until next time!</p>