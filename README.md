# 11 Notebooks, 10 failures  
Time spent: 
* Feb 20 6AM-9AM
* Feb 20 12:30PM-2:30PM
* Feb 20 7PM-10PM
* Feb 21 6AM-9AM
* Feb 21 10AM-11:30AM
* Feb 21 1PM-5:40PM
* Feb 21 6:30PM-10PM
Total time spent:  ~17 hours

*WAIT! Before I get into the story, some housekeeping: The final code I'm submitting is [5-cirq-final.ipynb](/attempts/5-cirq-final.ipynb). This is an implementation in Cirq. However, I spent the majority of time trying to build an implementation in Pennylane, in the notebook [11-pennylane-final.ipynb](/attempts/11-pennylane-final.ipynb). I ultimately failed, but you can check it out if you want. Even though the code there is incorrect, I have better explanations of the algorithm.*

### My QAOA maxcut journey 

My goal for this task was to "re-implement" Jack's code into Pennylane. I thought it would be easy: I already had a working non-weighted maxcut Pennylane code, as well as working weighted code from Cirq to base the circuit architecture off of. It would just be a quick tweak of a few lines of code, and all done, right?  
Wrong. It's been 6+ hours and my code still doesn't work :facepalm:  
To understand why, let's look at where I was at 11:30 AM, Febrary 21.  
I had just figured out how to generalize an unweighted maxcut circuit into a weighted one. It was quite easy: I had to tweak the code in what comes down to 3 locations:  
1. Change the cost function to take weights into account. I multiplied the unweighted cost by the weight.
2. Change the cost unitary matrix - multiply the exponent of the cirq ZZPowGate by the weight
3. Change the input graph into a weighted one


This was before I had really looked at anyone else's implementations in practice. I based it off of my understanding of theory. By nature that we're trying to maximize the sum of the weights of the number of cut edges, it made sense to simply multiply the unweighted cost by the weight. An edge with a higher weight being cut would increase our "score" more than an edge with a lower weight. 

In Jack Ceroni's code, for the cost hamiltonian, he uses `cirq.ZZPowGate(exponent= -1*gamma/math.pi).on(qubits[i.start_node], qubits[i.end_node])`

The ZZ powgate didn't exist in Pennylane, so I looked at the cirq docs to find out what it meant. Then, I built it into pennylane myself using qml.Hermitian() to put any gate onto the pennylane circuit.

Before ZZpowgating, the OG pennylane cost unitary was:

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
        pauli_z = [[1, 0], [0, -1]] # Define the Pauli-Z matrix. This is because we want to find the expected value of each edge pseudo-measured in the Z basis.
        pauli_z_2 = np.kron(pauli_z, pauli_z) # Takes the tensor product (or kronecker product) of 2 pauli Z matrices. This is because each edge has 2 vertices, each vertex is a qubit, so we need to take the expected value of 2 qubits in z basis, hence the tensor product of 2 z matrices.
        cost_unitary = scipy.linalg.fractional_matrix_power(pauli_z_2, -1*gamma/np.pi) # The unitary gate that will be applied to our circuit
        qml.Hermitian(cost_unitary, wires=[start_node, end_node]) # Note: wires = qubits. Here we are applying the unitary matrix we defined onto the 2 vertices of our edge.
```

Here's where the first major problem came in: ciq basically had it like the quantum circuit outputs the measurement in the computational basis, then we fire 100 shots at the circuit to find the probability distribution of the measured states. In pennylane, the root implementation **circuit directly outputs the expected value**.

In order to put qml.hermitian, I had to change the pennylane circuit to also output a computational basis measurement.

Then, I had to write extra fucntions to fire multiple shots and take the average. I couldn't re-use the cirq functions here, because pennylane has weird outputs.

So... I spent a decent chunk of my sunday afternoon on this task. I spent about 5 hours changing the circuit structure.

Now, after talking to Avneesh, I found out that I didn't actually have to ZZpowgate the pennylane circuit. I could've just multiplied the gamma from the original cost hamiltonian by the weight 🤦‍♂️ 

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

Anyway, it's there now. You can see it in 11.ipynb

Now, armed with this new insight, I updated my code and re-trained the Pennylane circuit. 

How did it do?

![Pennylane output histogram](img/pennylane-output.png "I got 1/2 of the correct answer, at least! Similing through the pain")

I got 1/2 of the correct answer, at least! Similing through the pain


Future: debug this code and see what's up. I estimate that doing such a task would take at least an hour, likely more (remember the 5 hours switching a single cost unitary matrix? That was because I was debugging the output of `qml.Hermitian(np.diag(range(2 ** n_wires)), wires=wires)`. So instead, I'm using an hour to write up this account of my experience. 

---

As of 8pm, the notebooks right now are pretty messy. I started writing my own code starting from 8.ipynb. I spent a good first hour trying to make all my variables "politically correct", having a space for all user inputs in one place, wanting to explain how the algorithm works in every line of code. But now...

11 is filled with comments from debugging. 

9:28 PM: I have updated notebook 11.

Also, I realized that all the content from 8.ipynb got deleted somehow. I'm positive I spammed ctrl+s in the Jupyter Notebook editor 10 times before closing, but... oh well. There goes the notebook I spent 5 hours debugging and 2 hours making the comments and structure perfect.


## Some interesting insights

I used what I dubbed a "lazy cost function" for a good chunk of my QAOA time: `weighted_cost = unweighted_cost * weight`

It does not output the correct cost, but nevertheless it still works on Cirq... kind of. I found this interesting. 

On Cirq, the lazy cost function outputs the correct answer but with less certainty. 

![Lazy cost function output](img/lazy-cost-function-cirq.png)

Compared to the legit cost function: 

![Legit cost function output](img/cirq-output.png)

