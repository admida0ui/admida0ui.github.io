---
title: Securinets NCSC IV CTF - Quantum Challenges Writeup
date: 2023-3-22 20:00:00
tags: Quantum
categories: Writeups
thumbnail: "https://i.imgur.com/NdX0RT0.jpg"
---

# QASM Writeup

## Description: 
>Can you make this quantum circuit work? Flag Format: Securinets{output}
>
>Author: mida0ui

Attachment: [circuit.qasm](https://github.com/RaniaMidaoui/CTF-writeups/blob/main/NCSC4.0-CTF/Quantum/QASM/circuit.qasm)

The file "circuit.qasm" was attached with the task, you can find it in this repo.

First of all, what is QASM?

QASM(quantum assembly language) is a simple text-format language for describing acyclic quantum circuits composed from single qubit, multiply controlled single-qubit gates, multiple-qubit, and multiple-qubit controlled multiple-qubit gates.
A QASM program declares the classical bits and qubits, describes the operations (gates) on those qubits and the measurements needed to obtain the classical result by inspecting the qubits. 

In this challenge, the only thing that needs to be done is to open the qasm file and run the circuit describes.
This can be done in various ways, there's no single intended solution, i will only state 2 solutions in this writeup. 

### Solution 1:
You can import the code in the QASM file in IBM Quantum composer: https://quantum-computing.ibm.com/composer/files/new
![2023-03-23_14h03_27](https://user-images.githubusercontent.com/68945305/227212904-65c59610-ca9d-47d8-9c10-77d5a778a901.png)

Yhe output of this circuit is always 01001 (with a 100% probability).
Flag: Securinets{01001}

### Solution 2:
You can import the QASM file code and run the circuit using qiskit.
In this case, to make the quantum circuit work, you only have to add measurements.

```
from qiskit import *

qasm_str = """OPENQASM 2.0;
include "qelib1.inc";

qreg q[5];
creg c[4];
y q[0];
y q[2];
x q[3];
y q[1];
id q[3];
id q[1];
z q[0];
y q[0];
ccx q[2], q[3], q[4];
swap q[0], q[1];
cx q[3], q[4];
x q[2];
x q[0];
cx q[2], q[3];
y q[0];
x q[2];
swap q[2], q[3];
cx q[3], q[4];
ccx q[0], q[1], q[2];
y q[4];
z q[3];
y q[2];
cx q[2], q[3];
measure q[0] -> c[0];
measure q[1] -> c[1];
measure q[2] -> c[2];
measure q[3] -> c[3];
"""

circuit = QuantumCircuit.from_qasm_str(qasm_str)

backend = Aer.get_backend('qasm_simulator')
job = execute(circuit, backend)
result = job.result()

print(result.get_counts(circuit))
```

The reuslt of this code is: 
![2023-03-23_15h35_19](https://user-images.githubusercontent.com/68945305/227237110-fde0799f-7c11-46f7-b979-5e875700b573.png)

But the circuit contains a quantum register of 5 qubits, we have to get the measurement result of them all: we notice that all the quantum gates described in the QASM file did not change the state of the fifth qubit, so its still in the default value.
The default state of a qubit, if not initialisaed is |0>.
So the overall output of the circuit is 01001 (always, with a 100% probability).

Flag: Securinets{01001}

# Secret Message 1

## Description

>Oh look! I managed to get this super secret message, but i cannot decode it :( The message is the output of this circuit.
>
>![226449081-4b7f8c73-8fe9-4a68-a515-fca0733f43df](https://user-images.githubusercontent.com/68945305/227240203-d0370ac5-878c-4283-8604-d0ca57514d69.png)
>
>The message is: òßÒïá×ÙßìâæÔ,í#ÙÞúÝïÙúî#.Ôúãï,Ù.ïÛúÎ,ì"/*ú#Ñ!Ó-Üë
>
>Can you help me ?
>
>Author: mida0ui

## Solution

A quantum gate is basic quantum circuit operating on a small number of qubits, they perform operations to change the state of the qubits, and unlike many classical logic gates, quantum logic gates are reversible.
Each quantum gate has its matrix, and since the qubit states are represented with vectors, the gates transform these vectors to get new states.
The state vectors of the states |0> and |1> are:

![2023-03-24_00h39_51](https://user-images.githubusercontent.com/68945305/227387829-286fd428-c826-4f30-8264-ea534bff4060.png)

First of all, lets explain the components of the given quantum circuit:
#### The X-Gate:
![2023-03-24_00h15_54](https://user-images.githubusercontent.com/68945305/227385150-7ef68a9d-5df0-44f1-8a15-3cd7ef30c3a1.png)

A single-qubit gate represented by the Pauli-X matrix:

![2023-03-24_00h18_49](https://user-images.githubusercontent.com/68945305/227385473-bec4a07f-9c72-4cdd-bcb3-1d74bf6f60d6.png)

the X-gate simply switches the amplitudes of the states |0> and |1>, you can see it like this: A qubit has a probability of measuring zero and a probability of measuring one, and an X gate swaps those probabilities. It perfroms a π rotation around the x-axis.

#### The Y-Gate:
![2023-03-24_00h16_38](https://user-images.githubusercontent.com/68945305/227386762-efc389e2-0fc7-40df-a1d4-ce875cc796b7.png)

Similar to the X gate, it's single-qubit gate represented by the Pauli-Y matrix:

![2023-03-24_00h30_41](https://user-images.githubusercontent.com/68945305/227386943-e500f1df-3336-46e4-9e04-77b4712c0a66.png)

It performs a π rotation around the y-axis in the Bloch sphere.

#### The S-Gate:
![2023-03-24_00h16_31](https://user-images.githubusercontent.com/68945305/227387496-b3b2db78-49ab-461e-aa20-6bc45a8a2175.png)

The Phase gate (or S gate) is a single-qubit operation represented by the matrix:

![2023-03-24_00h35_34](https://user-images.githubusercontent.com/68945305/227387322-15c87f80-6249-4b31-b330-320e1527e89f.png)

The S gate is also known as the phase gate or the Z90 gate, because it represents a 90-degree rotation around the z-axis.

#### The Controlled Not or CX Gate:
![2023-03-24_00h16_42](https://user-images.githubusercontent.com/68945305/227387580-b1f2e3e5-a042-4cd9-ab1a-85efe86e6f7e.png)

The CNOT gate iss a two-qubit quantum gate that performs a controlled NOT where one qubit is the control and the second qubit is the target. Its matrix is the following :

![2023-03-24_00h40_32](https://user-images.githubusercontent.com/68945305/227387914-12ac7b4b-7a8a-454b-9b72-7168e05c378a.png)

When applied:

![2023-03-24_00h40_54](https://user-images.githubusercontent.com/68945305/227387960-e482bfa2-1ef5-40dd-837d-ac00bb5b5a8d.png)

The mathematics behind these results are simple:

Lets take |10> and |11>:

![2023-03-24_00h41_31](https://user-images.githubusercontent.com/68945305/227388030-6a777d62-5c66-4930-ad04-8d80a4cc8209.png)

This ⊗ represents a tensor product, you can read about it here if you don't know what it is : https://www.math3ma.com/blog/the-tensor-product-demystified

The first qubit is usually referred to as the control qubit and the second qubit as the target qubit. Expressed in basis states, the CNOT gate: leaves the control qubit unchanged and performs a Pauli-X gate or NOT gate (which just flips the state of the qubit) on the target qubit when the control qubit is in state ∣1⟩; leaves the target qubit unchanged when the control qubit is in state ∣0⟩.

#### The SWAP Gate:
![2023-03-24_00h16_58](https://user-images.githubusercontent.com/68945305/227388440-e82427d9-a370-40cb-bbeb-57b4ed6e79f9.png)

The SWAP gate is two-qubit operation. Expressed in basis states, the SWAP gate swaps the state of the two qubits involved in the operation:

![2023-03-24_00h45_19](https://user-images.githubusercontent.com/68945305/227388502-2d56a22c-9f23-445f-a93f-ac1861661171.png)

#### Reversing the given circuit:

All the matrices of the gates stated bellow (and in genereal in quantum computing) are reversible, which means, the gate can be applied to reverse its own effet, so the only thing we have to do is to put them in a reversed order, and we get our flag.
We transform the output given in the description into a binary string to fit in the quantum circuit, the encoding used here is 'windows-1252' because the output string uses characters from the extended ASCII table, so we cannot use utf-8 for example. We create a quantum circuit with one quantum register containing 2 qubits and one classical register to measure the state of those qubits, put the gates in a reversed order and pass our string in a loop and decode the message slowly.

```
from qiskit import *

output = 'òßÒïá×ÙßìâæÔ,í#ÙÞúÝïÙúî#.Ôúãï,Ù.ïÛúÎ,ì"/*ú#Ñ!Ó-Üë'
output_array = ''.join(format(i, '08b') for i in bytearray(output, encoding ='windows-1252'))

bin_flag = ''

def state_vect(bit):
    if(bit == '0'):
        vect = [1, 0]
    else : 
        vect = [0 ,1]
    return vect

i = 0
while i<len(output_array):
    
    #Initialize the quantum circuit
    q = QuantumRegister(2, "q")
    c = ClassicalRegister(2, "c")
    
    qc = QuantumCircuit(q,c)

    vect0 = state_vect(output_array[i])
    vect1 = state_vect(output_array[i+1])

    qc.initialize(vect1,0)
    qc.initialize(vect0,1)
    
    qc.measure(q, c)

    qc.swap(q[0],q[1])
    qc.x(q[0])
    qc.cx(q[0],q[1])
    qc.s(q[0])
    qc.x(q[0])
    qc.y(q[1])

    qc.measure(q, c)
    
    qobj = assemble(qc, shots=1)
    job = execute(qc,Aer.get_backend('qasm_simulator'),shots=1)
    counts = job.result().get_counts()
    
    bin_flag += str(counts)[2:4]

    i=i+2
    
flag = ''.join(chr(int(bin_flag[i*8:i*8+8],2)) for i in range(len(bin_flag)//8))
print(flag)
```
Flag: Securinets{h4v1ng_fun_w17h_qu4n7um_G4t35?_1b2a6d}

# Secret Message 2

## Description
>
> I want to tell you a secret, but i'm afraid someone will intercept our communication. I know how good you are now with quantum encoding, so I will let you figure it out on your own. 
> I used the following code to get to encode the secret:
 ```
 qc.s(q[2])
 qc.ccx(q[0],q[1],q[2])
 qc.h(q[2])
 qc.swap(q[1],q[0])
 qc.y(q[0])
 ```
>The secret is: VMëÜÁÂÎèÜ
>
>Also, the secret contains two english words, it doesn't contain special characters nor numbers.
>
>Flag format: Securinets{decoded_message}
>
>BE CAREFUL, YOU ONLY HAVE 5 attempts
>
>Author: mida0ui

## Solution
From the description of this task, we can see that the CCNOT gate was used, along with other gates to encode the message, meaning that we need to use a quantum register containing 3 qubits to solve this task. The CCNOT gate is similar to the CNOT gate seen in the previous task, but it's a three-qubit gate, it is applied to one target qubit based on the state of two control qubits:

![2023-03-24_12h41_54](https://user-images.githubusercontent.com/68945305/227512203-39c1f52b-4c32-426e-8fb8-676bf2740cd8.png)

To encode the message, a very special gate waas used too, the Hadamard gate.
The Hadamard gate is a single-qubit operation that maps the basis state ∣0⟩ to ![2023-03-24_12h45_04](https://user-images.githubusercontent.com/68945305/227512818-c69dc67e-3b9a-4110-806a-68a1d8259f4f.png) = |+> and |1> to ![2023-03-24_12h45_39](https://user-images.githubusercontent.com/68945305/227512923-b68d7c97-22fb-4e3b-9ca3-59e835b93bf9.png) = |->, thus creating an equal superposition of the two basis states.
It has the matrix:

![2023-03-24_12h55_46](https://user-images.githubusercontent.com/68945305/227514820-95d36f4c-11ce-47bb-bdcf-ef68002436e0.png)


A qubit can exist in a superposition of its two "basis" states, which loosely means that it is in both states simultaneously. When measuring a qubit, the result is a probabilistic output of a classical bit, so for one single qubit, the measurement can give either |1> or |0> depending on the probabilistic state.
With that said, we cannot use one iteration to decode the message like in the previous task, we need to print all the possibilities (or almost) to see the different reults of measurements, and get the decoded message.
To make the task easier and less time consuming, the description specifies that the secret contains two english words and it doesn't contain special characters nor numbers. So we will filter the results based on these two facts.

```
import matplotlib
from qiskit import *
import re

output = 'VMëÜÁÂÎèÜ'
output_array = ''.join(format(i, '08b') for i in bytearray(output, encoding="windows-1252"))

bin_flag = ''
bins = ''
def state_vect(bit):
    if(bit == '0'):
        vect = [1, 0]
    else : 
        vect = [0 ,1]
    return vect

while(1):
    i = 0
    flag = ""
    while i<len(output_array):

        #Initialize the quantum circuit
        q = QuantumRegister(3, "q")
        c = ClassicalRegister(3, "c")

        qc = QuantumCircuit(q,c)

        #get the initial state from the output
        vect0 = state_vect(output_array[i])
        vect1 = state_vect(output_array[i+1])
        vect2 = state_vect(output_array[i+2])

        #in itialize the register
        qc.initialize(vect0,2)
        qc.initialize(vect1,1)
        qc.initialize(vect2,0)


        qc.measure(q, c)

        #Reverse the circuit
        qc.y(q[0])  
        qc.swap(q[1],q[0])     
        qc.h(q[2])
        qc.ccx(q[0],q[1],q[2])
        qc.s(q[2])


        qc.measure(q, c)

        #execute the circuit
        qobj = assemble(qc, shots=1)
        job = execute(qc,Aer.get_backend('qasm_simulator'),shots=1)
        counts = job.result().get_counts()

        bin_flag += str(counts)[2:5]
        #Increment the counter to get the next 2 qubits
        i=i+3

    flag = ''.join(chr(int(bin_flag[i*8:i*8+8],2)) for i in range(len(bin_flag)//8))
    if(bool(re.match('^[a-zA-Z]*$',flag))==True):
        print(flag)
    bin_flag = ''
    ```
Flag: Securinets{youwin}