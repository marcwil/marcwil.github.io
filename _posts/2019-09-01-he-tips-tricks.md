---
layout: post
comments: true
title:  "FHE2: Tips and tricks for practical homomorphic encryption"
date:   2019-09-01 11:25:15 +0100
categories: jekyll update
---

In the last post we had a look at the basics of Homomorphic Encryption.
In this post we cover a few advanced tricks that are useful in practice especially with respect to performance.
To that end, we have a look at the example of calculating the dot product of vectors.
Doing that, we learn about how to use different encodings and specific optimizations that can be used with the batch encoding.

# Case study: dot product of vectors 

As already mentioned this entire post will deal with the dot product of vectors.

Given two vectors `A=[a1,a2,...,aN]` and `B=[b1,b2,...,bN]`, the dot product `A*B` of `A` and `B` is defined as `A*B=a1*b1+a2*b2+...+aN*bN`.

Example: `[1,2,3]*[2,2,2] = 1*2 + 2*2 + 3*2 = 12`

The dot product is an important operation for neural networks, which makes this example relevant for many applications.

In Python the calculation can be done with NumPy as follows...
{% highlight python %}
import numpy as np

a = np.array([1,2,3])
b = np.array([2,2,2])

print(a.dot(b))
#=> prints 12
{% endhighlight %}
... or manually.
{% highlight python %}
def dot(A,B):
    return sum([a*b for (a,b) in zip(A,B)])

print(dot(a,b))
#=> prints 12
{% endhighlight %}

# Homomorphically encrypted dot product

Now assume that we still want to calculate `A*B` for some vectors `A` and `B`, but we want `A` to be encrypted.
This could be useful in a scenario where confidential data is fed into a neural network.

A naive approach would encrypt all numbers in `A` separately and perform one multiplication for each position in the vector. This could look like this:
{% highlight python %}
def encrypt_input(A):
    return [he.encryptInt(a) for a in A]

def encode_input(B):
    return [he.encodeInt(b) for b in B]

def encrypted_dotproduct(A, B):
    products = [he.multiply_plain(a, b, True) for (a,b) in zip(A,B)]
    result = products[0]
    for summand in products[1:]:
        he.add(result, summand)
    return result
{% endhighlight %}
We can verify this simple implementation:
{% highlight python %}
a_encrypted = encrypt_input([1,2,3])
b_encoded = encode_input([2,2,2])

result_encrypted = encrypted_dotproduct(a_encrypted, b_encoded)

print(he.decryptInt(result_encrypted))
#=> prints 12
{% endhighlight %}

## Performance
In order to analyze the performance of this naive approach we calculate the dot product with random inputs of different length.
We use the ipython magick `%timeit` for the performance measurement.
{% highlight python %}
def example_instance(length):
    a = np.random.randint(0, 100, (length))
    b = np.random.randint(0, 100, (length))
    a_enc = encrypt_input(a)
    b_enc = encode_input(b)
    return a_enc, b_enc

a_enc, b_enc = example_instance(100)
%timeit encrypted_dotproduct(a_enc, b_enc)
#=> 98.7 ms ± 17 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
a_enc, b_enc = example_instance(1000)
%timeit encrypted_dotproduct(a_enc, b_enc)
#=> 1.03 s ± 203 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
a_enc, b_enc = example_instance(4096)
%timeit encrypted_dotproduct(a_enc, b_enc)
#=> 3.52 s ± 44.6 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
{% endhighlight %}
Now let's compare that to the processing times with plaintext, once using the `dot` function from earlier and one using numpy.
{% highlight python %}
def plain_instance(length):
    a = np.random.randint(0, 100, (length))
    b = np.random.randint(0, 100, (length))
    return a,b

a, b = plain_instance(100)
%timeit dot(a,b)
#=> 38.3 µs ± 661 ns per loop (mean ± std. dev. of 7 runs, 10000 loops each)
%timeit np.dot(a,b)
#=> 1.25 µs ± 6.71 ns per loop (mean ± std. dev. of 7 runs, 1000000 loops each)
a, b = example_instance(4096)
%timeit dot(a, b)
#=> 1.46 ms ± 5.23 µs per loop (mean ± std. dev. of 7 runs, 1000 loops each)
%timeit np.dot(a, b)
#=> 4.09 µs ± 14.8 ns per loop (mean ± std. dev. of 7 runs, 100000 loops each)
{% endhighlight %}

We see that the performance of the encrypted dot product is about a factor of 2500 times slower than the plaintext operation in python.
Also, multiplying more numbers takes more time, which should not be too surprising.
In the next section we will improve this.

## Parallelization with the batch encoding

While the naive approach offers very poor performance, we can use one of SEAL's more advanced encodings, the batch encoding, in order to multiply 4096 (or whatever our polynomial modulus is) numbers at once.
We can rewrite the dot product to use batch encoding as follows:
{% highlight python %}
def encrypt_input_batch(A):
    return he.encryptBatch(A)

def encode_input_batch(B):
    return he.encodeBatch(B)

def encrypted_dotproduct_batch(A, B):
    product = he.multiply_plain(A, B, True)
    result = product
    shift = 1
    while shift < he.getnSlots()/2:
        shifted = he.rotate(result, shift, True)
        result = he.add(result, shifted, True)
        shift *= 2
    return result
{% endhighlight %}
We now represent a whole vector of numbers in a single ciphertext and multiply all elements in `A` and `B` in one step.
After that we make clever use of the rotate function.
In order to sum up all entries of the vector, using only a logarithmic number of rotate operations, we rotate and add intermediate results.

If we call the intermediate result of the first loop `i1`, the one of the second loop `i2`, and so on, we can visualize the computation like this:
```
i1 = [product[1], product[2], ..., product[4095], product[4096]] +
     [product[2], product[3], ..., product[4096], product[1]]
   = [product[1]+product[2], product[2]+product[3], ...]
i2 = [i1[1], i1[2], ..., i1[4095], i1[4096]] +
     [i1[3], i1[4], ..., i1[1], i1[2]]
   = [product[1]+product[2]+product[3]+product[4], ...]
i3 = [i2[1], i2[2], ..., i2[4095], i2[4096]] +
     [i2[5], i2[6], ..., i2[3], i2[4]]
   = [product[1]+product[2]+...+product[8], ...]
...
```
This way the first entry in the resulting array contains the sum of all entries in `product`:
{% highlight python %}
a_enc = encrypt_input_batch([1,2,3])
b_enc = encode_input_batch([2,2,2])

res_enc = encrypted_dotproduct_batch(a_enc, b_enc)
print(he.decryptBatch(res_enc)[0])
#=> prints 12
{% endhighlight %}

We can now again test the performance the same way as before
{% highlight python %}
def example_instance_batch(length):
    a = np.random.randint(0, 100, (length))
    b = np.random.randint(0, 100, (length))
    a_enc = encrypt_input_batch(a)
    b_enc = encode_input_batch(b)
    return a_enc, b_enc

a_enc, b_enc = example_instance_batch(4096)
%timeit encrypted_dotproduct_batch(a_enc, b_enc)
#=> 57.7 ms ± 708 µs per loop (mean ± std. dev. of 7 runs, 10 loops each)
{% endhighlight %}
A dotproduct of vectors of length 4096 now takes 57.7ms! Compared to the 3.52s from before that is a massive improvement.
Compare that with the `dot` implementation in plain python, which took 1.46ms, and we see that the new encrypted dotproduct is less than 60 times slower!
A massive improvement.

## Batch encoding with floating point numbers

In the use case of deep learning, the dot product often has to be computed on vectors of floating point numbers.
As the batch encoding only supports vectors of integers, we cannot directly operate on floating point numbers.

Solutions to this are either using a framework that uses a crypto scheme supporting batches of floating point numbers (such as [CKKS](https://eprint.iacr.org/2016/421), implemented in SEAL 3), or treating the floating point numbers as fixed point numbers that can be scaled.

For example in order to have 3 digits accuracy, all values can be multiplied with 1000 and then rounded, before encryption.
The decrypted result of a multiplication then has to be divided by 1000000.

This of course means that the plaintext modulus `p` needs to be quite large, as the highest absolute value that can be stored in batch encoding is `p/2`.
This is not very desirable, as a larger plaintext modulus lead to less noise budget, unless the polynomial modulus is increased as well, which degrades performance.

One trick that sometimes helps is to use the [Chinese Remainder Theorem](https://en.wikipedia.org/wiki/Chinese_remainder_theorem).
Using the CRT we can create two PythonFHE objects with different plaintext moduli `p1, p2` and perform all computations with both PythonFHE objects.
The results modulo `p1` and `p2` can then be used to recover the result modulo `p1*p2`.
This way much bigger numbers can be stored.
In an individual use case however, it is most often necessary to manually determine adequate parameters based on the required precision, noise budget and performance.

# Conclusion

In this post we implemented a commonly used vector operation with homomorphic encryption.
We first tried a naive approach, which lead to very bad performance.
A second approach used the batch encoding to store complete vectors in a single ciphertext.
This was more difficult to implement, but improved the performance massively, as the number of multiplications was reduced.

Last but not at least, the last section discussed ways of storing floating point numbers with batch encoding and quickly described a way of using the Chinese Remainder Theorem in order to achieve smaller polynomial moduli.
