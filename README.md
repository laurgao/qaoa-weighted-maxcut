# 11 Notebooks, 10 failures  
Time spent: 
* Feb 20 6AM-9AM
* Feb 20 12:30PM-2:30PM
* Feb 20 7PM-10PM
* Feb 21 6AM-9AM
* Feb 21 10AM-11:30AM
* Feb 21 1PM-5:40PM
* Feb 21 6:30PM-10:30PM  

Total time spent:  ~17 hours

*WAIT! Before I get into the story, some housekeeping: The final code I'm submitting is [11-pennylane-final.ipynb](/attempts/11-pennylane-final.ipynb). This is an implementation in Pennylane that I coded essentially from the ground up. The code in said notebook isn't perfect and the result is not always correct, but I choose to submit it anyway. I do, however, want to let you know that I have a perfectly-working implementation of QAOA weighted maxcut in the notebook [5-cirq-final.ipynb](/attempts/5-cirq-final.ipynb) - the code there outputs the correct result and is much more effecient than the code I wrote.*

## Problem statement

**Tl;dr: Generalize the QAOA code to solve the maxcut problems for *weighted* graphs.**

The [MaxCut problem](https://en.wikipedia.org/wiki/Maximum_cut) is a well-known optimization problem in which the nodes of a given undirected graph have to be divided in two sets (often referred as the set of “white” and “black” nodes) such that the number of edges connecting a white node with a black node are maximized. The MaxCut problem is a problem on which the QAOA algorithm has proved to be useful (for an explanation of the QAOA algorithm you can read [this blogpost](https://www.mustythoughts.com/quantum-approximate-optimization-algorithm-explained)).

At [this link](https://lucaman99.github.io/new_blog/2020/mar16.html) you can find an explicit implementation of the QAOA algorithm to solve the MaxCut problem for the simpler case of an unweighted graph. We ask you to generalize the above code to include also the solution for the case of weighted graphs. You can use the same code or you can also do an alternative implementation using, for example, qiskit. The important point is that you do not make use of any built-in QAOA functionalities.

## My QAOA maxcut journey 

My persoanl goal for this task was to "re-implement" Jack's code into Pennylane. I thought it would be easy: I already had working non-weighted maxcut Pennylane code from the demo on their website, as well as working weighted code from Cirq to base the circuit architecture off of. It would just be a quick tweak of a few lines of code, and all done, right?  

Wrong. It's been 6+ hours and my code still doesn't work 🤦‍♂️   

To understand why, let's look at where I was at 11:30 AM, Febrary 21.  

I had just figured out how to generalize an unweighted maxcut circuit into a weighted one. It was quite easy: I had to tweak Jack's code in what comes down to 3 locations:  
1. Change the cost function to take weights into account. I multiplied the unweighted cost by the weight.
2. Change the cost unitary matrix - multiply the exponent of the Cirq ZZPowGate by the weight
3. Change the input graph into a weighted one

(This is the code of [5-cirq-final.ipynb](/attempts/5-cirq-final.ipynb))


This was before I had really looked at anyone else's weighted implementations in practice, so I based it off of my understanding of the theory. By nature that we're trying to maximize the sum of the weights of the number of cut edges, it made sense to simply multiply the unweighted cost by the weight. An edge with a higher weight being cut would increase our "score" more than an edge with a lower weight. 

In Jack Ceroni's code, for the cost hamiltonian, one gate he uses is `cirq.ZZPowGate(exponent= -1*math.pi).on(qubits[i.start_node], qubits[i.end_node])`

The ZZPowGate didn't exist in Pennylane, so I looked at the Cirq docs to find out what it meant. What the gate does: it takes the tensor product (or also known as the kronecker product) of two Pauli-Z matrices, and raises it to the power of a parameter you input (in this case `exponent= -1*math.pi`). This is done because each edge has 2 vertices and each vertex is a qubit, therefore we need to take the expected value of 2 qubits in the Z basis, hence the tensor product of two Z matrices.

Then, I built the gate into Pennylane myself using `qml.Hermitian()`. `qml.Hermitian()` essentially is used to put any unitary matrix you defined as a gate onto your circuit.

Before ZZPowGate-ing, the OG Pennylane cost unitary was:

```python
def U_C(gamma):
    for edge in graph:
        wire1 = edge[0]
        wire2 = edge[1]
        qml.CNOT(wires=[wire1, wire2])
        qml.RZ(gamma, wires=wire2)
        qml.CNOT(wires=[wire1, wire2])
```

Now, the new cost unitary with the weights:

```python
def U_C(gamma):
    for edge in graph:
        start_node = edge[0]
        end_node = edge[1]
        weight = edge[2]
        
        # Define the Pauli-Z matrix. This is because we want to find the expected value of each edge pseudo-measured in the Z basis.
        pauli_z = [[1, 0], [0, -1]] 
        pauli_z_2 = np.kron(pauli_z, pauli_z) 
        
        # The unitary gate that will be applied to our circuit
        cost_unitary = scipy.linalg.fractional_matrix_power(pauli_z_2, -1*gamma/np.pi) 
        
        # Note: wires = qubits. Here we are applying the unitary matrix we defined onto the 2 vertices of our edge.
        qml.Hermitian(cost_unitary, wires=[start_node, end_node]) 
```

Here's where the first major problem came in. Jack's code's structure is set up like this: the quantum circuit outputs the measurement in the computational basis, then we fire 100 shots at the circuit to find the probability distribution of the measured states. In Pennylane, the root **circuit directly outputs the expected value**.

In order to use `qml.Hermitian()`, I had to change the Pennylane circuit to also output a computational basis measurement.

Then, I had to write extra fucntions to fire multiple shots and take the average. I couldn't re-use the Cirq functions here, because Pennylane's computational basis measurement outputs some weird tensor arrays, which is different from what Cirq's outputs. (FYI, I spent a good bulk of that 5 hours debugging this output.)

So... I spent a decent chunk of my Sunday afternoon on this task. I spent about 5 hours *just changing the circuit structure.*

Now, after talking to a friend Avneesh, I found out that I didn't actually have to ZZPowGate the Pennylane circuit. I could've just multiplied the gamma from the original cost hamiltonian by the weight 🤦‍♂️ 

Like this:

```python
def U_C(gamma):
    for edge in graph:
        wire1 = edge[0]
        wire2 = edge[1]
        weight = edge[2]
        qml.CNOT(wires=[wire1, wire2])
        qml.RZ(gamma*weight, wires=wire2)
        qml.CNOT(wires=[wire1, wire2])
```

5 hours re-engineering the structure of my code, writing functions to convert decimal strings into binary, were all useless. 

Anyway, it's there now. You can see it in [11-pennylane-final.ipynb](/attempts/11-pennylane-final.ipynb)

Now, armed with this new insight, I updated my code and re-trained the Pennylane circuit. 

How did it do?

![Pennylane output histogram](img/pennylane-output.png "I got 1/2 of the correct answer, at least! Similing through the pain")

I got 1/2 of the correct answer, at least! Similing through the pain


Future: debug this code and see what's up. I estimate that doing such a task would take at least an hour, likely more (remember the 5 hours switching a single cost unitary matrix? That was because I was debugging the output of `qml.Hermitian(np.diag(range(2 ** n_wires)), wires=wires)`. So instead, I'm using an hour to write up this account of my experience. 

---

As of 8pm, the notebooks right now are pretty messy. I started writing my own code starting from 8.ipynb. I spent a good first hour trying to make all my variables "politically correct", having a space for all user inputs in one place, wanting to explain how the algorithm works in every line of code. But now...

11 is filled with comments from debugging. 

10:27 PM: I have updated notebook 11.

Also, I realized that all the content from 8.ipynb got deleted somehow. I'm positive I spammed ctrl+s in the Jupyter Notebook editor 10 times before closing, but... oh well. There goes the notebook I spent 5 hours debugging and 2 hours making the comments and structure perfect.



## Some interesting insights

I used what I dubbed a "lazy cost function" for a good chunk of my QAOA time: `weighted_cost = unweighted_cost * weight`

It does not output the correct cost, but nevertheless it still works on Cirq... kind of. The lazy cost function outputs the correct answer but with less certainty: 

![Lazy cost function output](img/lazy-cost-function-cirq.png)

Compared to the legit cost function: 

![Legit cost function output](img/cirq-output.png)

I found this interesting. I'm curious to know why. I'm going to work through this math.
