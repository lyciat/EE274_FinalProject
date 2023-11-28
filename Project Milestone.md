# Project Milestone

> Pranavi Boyalakuntla and Lycia Tran
Autumn 2023
EE 274: Data Compression
> 

---

# Introduction

We are planning on optimizing the LZ77 implementation on SCL. LZ77 is the basis for some of the most widely used compressors (gzip, zstd), but the current implementation on SCL does not use repcodes and uses Huffman coding for entropy coding. We believe with the addition of a basic repcode and by changing the entropy coding to rANS, tANS, or Arithmetic coding we will see an improved data compression rate. The following variations of the LZ77 compressor will be implemented: original SCL implementation, variation with rANS coding, variation with tANS coding, variation with Arithmetic coding, variation with repcodes, variation with repcodes and rANS coding, variation with repcodes and tANS coding, and variation with repcodes and Arithmetic coding.  These variations will be tested on different datasets (text file, random ASCII, RGB array, and github user information) to see which variation performs the best for each dataset. 

# Previous Work/Literature Review

## Original LZ77 Implementation

The LZ77 algorithm works by replacing later iterations of a previously existing pattern by encoding information about how far back the pattern exists and how long the pattern was in a table. Unmatched patterns, or “new”, patters are included in the table as is. The specifics of how this is implemented in code as a part of the SCL library are discussed below. The steps to implement the LZ77 algorithm are:

- Keep track of your current position
- Find a match for your current point with something that exists previously
- Encode how far back that match was and how long the match exists for

### Encoder

This section will detail the variables and functions that make up the original LZ77 encoder and the order in which they are utilized.

This implementation utilizes a custom `LZ77Window` class provides utilities for a given byte array including retrieving the current window or getting a byte from a current window. This class stores information about the current window like the start and end pointers. 

This implementation also utilizes a custom match finder with options for two types: a base match finder, or a hash based match finder. The base match finder, `MatchFinderBase`, takes a lookahead buffer and a sliding window and finds the best possible match. The lookahead buffer is what we are trying to match with in the window. This function, `extend_match`, will look at a given candidate to match and will extend the match to the left and right. The position of the match in the lookahead buffer and the sliding window will be returned along with its length. 

The hash based match finder, `HashBasedMatchFinder`, builds off the `MatchFinderBase` to find the best match using a hash table. This hash table will store all the hash values and the previous occurances of that hash value. The best match is found using the function, `find_best_match` where all the starting positions in the lookahead buffer are hashed and the best match is found within a sliding window. If this match is less than the minimum length, the loop continues. 

The encoder begins by instantiating the first window with a provided window size, or a default of 1000000 bits. In order to encode a file, the file is broken down into blocks which are then encoded using the detailed LZ77 algorithm. The function `lz77_parse_and_generate_sequence` is used to encode a block. The function loops through the provided block and stores where the current position is throughout. Using the `HashBasedMatchFinder` a match is found for the provided lookahead buffer. If this match length is non-zero, then the offset is found and the current position is shifted. The bytes that are “done” are added onto the window so that they are included for future matches. If this match is zero, then the current position is shifted, but the bytes are saved as is and added to the window. This function returns the final lz77 table and the saved literals that did not have a pattern match.

![Untitled](Project%20Milestone%20423def15c9dc4e1db310a4b49fde68e2/Untitled.png)

### Decoder

The LZ77 decoder implementation loops through the LZ77 tables and decodes on a per block basis using the function `decode_block`which then calls the `execute_lz77_sequence` function which will go through the table to reconstruct the data. Since the table includes information about the literals, match offset, and match length, the table can be used to expands the compressed version. The position in the literals will need to be tracked while the decoding process is being completed because the literals are stored in their own list rather than as a list of lists. The length of the literal is stored however. The decoding is completed by, copying literals, copying over the matching string, and then appending any remaining literals. 

![Untitled](Project%20Milestone%20423def15c9dc4e1db310a4b49fde68e2/Untitled%201.png)

## Huffman Coding

Huffman codes are one of the optimal prefix free codes for a given distribution and uses a tree structure to encode and decode the data based on the frequency of each character. The Huffman tree is constructed by joining the two nodes with the smallest probabilities into a new node. The nodes will be either the characters to be encoded, or previously joined nodes. This procress is repeated until a single tree has been created. This approach is very greedy, but ends up being quite optimal with a compression rate close to entropy. However, there are some issues that occur with Huffman coding such as:

1. The data has to be split up into blocks
2. The codebook needs to be computed beforehand and it’s size increases exponentially wiht our block size
3. In practice, we cannot actually obtain a compression as close as 1/B bits/symbol to entropy H(X) with block size B. 

## Asymmetric Numeral Systems (ANS)

Asymmetrical Numerical Systems (ANS) are a set of compression algorithms that provide a good tradeoff between compression rate and speed. The two we will be implementing are range-ANS and table-ANS.

### range-ANS (rANS)

In rANS, we start with the probabilities of each symbol, `prob[s] = freq[s]/M`. The frequency of a symbol, `freq[s]`, is the number of times the symbol appears in the sequence, and `M` is the Range Factor, which the sum of frequencies of all the symbols. We also have the cumulative frequency, which is the sum of the frequencies for all the symbols before. ie: `culmul[0] = 0, cumul[1] = freq[0], cumul[2] = freq[0] + freq[1]`, etc. 

For example: if we were to encode the symbols `x = {1, 2, 3}`, we would get the following

```python
freq = [2, 3, 2]
M = 2 + 3 + 2 = 7
cumul = [0, 2, 5]
prob = [2/7, 3/7, 2/7]
```

To encode rANS, we will start at an initial state, `x = 0`. With each new symbol, we will calculate x_next and then set our state x to x_next. We will keep updating our state x until we have reached the end of our data. 

```python
x_next = (x//freq[s])*M + cumul[s] + x%freq[s]
```

To decode rANS, we want to recover our symbol values, s, and our previous state value, x_prev. We will follow the steps listed below to recover our original data. At the end of each iteration, we will update our state, x to be our recovered x_prev value, and repeat the following steps until `x_prev = 0`. 

```python
# step 1: finding the block_id and slot
block_id = x//M
slot = x%M

# step 2: find the symbol value s
s = find_bin(cumul, slot)

# step 3: find the previous state value, x_prev
x_prev = block_id*freq[s] + slot - cumul[s]
```

However, since our state, x, is increasing exponentially every iteration, we will actually store x as the `ceil(log2(x))`. If we limit x to a pre-defined range, `Interval[L,H]`, then we will at max transmit `ceil((log2(H))`. This leads us to the `streaming rANS` case where we write out both the new state value and additional bits after we encode each symbol. This means that our encoded bit array will not just be our state `x`, but the additional output bits, `out_bits`, followed by the binary value of x asserted into our `Interval[L,H]`, `x_shrunk`. `out_bits` and `x_shrunk` are calculated using the following method:

```python
x_shrunk = x
while x_shrunk not in Interval[L,H]:
	out_bits.append(x_shrunk%2)
	x_shrunk = x_shrunk//2
```

These bits will be prepended to our encoded bit array. The length of `x_shrunk` is also known as `````num_state_bits = ceil((log2(H)).` Therefore to recover this, we can grab the first `````num_state_bits````` in the encoded bit array and use the following method:

```python
x_shrunk = encoded_bitArr[:num_state_bits]
x = x_shrunk
num_out_bits = 0

while x_shrunk not in Interval[L,H]:
	x = x*2 + encoded_bitArr[num_out_bits]
	num_out_bits += 1
```

An example of this state shrinking and output bits can be seen below:

```python
# state value x
x = 10 = 1010b

# possible outputs using streaming rANS
x_shrunk = 1010b = 10, out_bits = ""
x_shrunk = 101b = 5, out_bits = "0"
x_shrunk = 10b = 2, out_bits = "10"
x_shrunk = 1b = 1, out_bits = "010"'
```

One thing to note is that as we expand the state `x` back out: `x = x*2 + encoded_bitArr[num_out_bits]`, we need to ensure that there is a unique `x` which lies in the `Inteval[L,H]`, otherwise we may not recover the correct `x` value. This means that the Interval[L,H] must meet the following constraint:

$$
L = Mt, H = 2Mt - 1 
$$

where M = sum of frequencies and t = an unsigned integer

In practice, we see that M is often chosen such that

$$
M = 2^r
$$

where r = integer (most commonly r = 16). This allows the `block_id` and `slot` calculation to perfrom much faster. In order to further improve the speed of our algorithm, we stream out 8, 16, or 32 bits when we shrink and expand our state instead of just 1 bit. This significantly reduces the speed of our rANS encoder and decoder. 

### table-ANS (tANS)

tANS is a variation of a cached rANS where we cache our rANS encoder and decoder. Caching is can be used to streamline repeated processes by doing the process once as a part of pre-processing and storing the results into a lookup table. This can be implemented into rANS, since we are repeating the same steps in our encoding and decoding process. 

The problem with cached rANS or tANS is that retrieving a value from a lookup table also takes time and the lookup table must be stored, so we have to consider if the saved computation time is worth the added retrieval time and the additional memory needed to store the table. As a general rule of thumb, the caching based methods, like tANS, should be used when the corresponding lookup tables are small, since larger tables will require accessing data from the RAM which is much slower. 

In order to cache the decode step, we have to cache the `rans_decode_step` and the `expand_state` step into individual lookup tables. `rans_decode_step` cached will map `x` to `x_shrunk, s`, thus making the lookup table size `M`. `expand_state` will be modified to the function below, so that it can be turned into a lookup table.

```python
def expand_state(x_shrunk, enc_bitarray):
    num_bits_step = (r+1) - ceil(log2(x_shrunk+1))

    # get the new state:
    x = (x_shrunk << num_bits_step) + to_uint(enc_bitarray[:num_bits_step])

    return x, num_bits_step

```

We can also cache the encode step by again turning `rans_encode_step` and the `shrink_state` into individual lookup tables. rans_encode_step will map `x_shrunk, s` to `x`, thus making it a the lookup table size `M`.  `shrink_state` will be modified similarly to `expand_state`, so that it can be turned into a lookup table as well. 

```python
def shrink_state(x,s):
    # calculate the power of 2 lying in [freq[s], 2freq[s] - 1]
    y = ceil(log2(freq[s]))
    num_out_bits = r+1 - y
    thresh = freq[s] << (num_out_bits+1)
    if x >= thresh:
        num_out_bits += 1
    
    x_shrunk = x >> num_out_bits
    out_bits = to_binary(x)[-num_out_bits:]
             
    return x_shrunk, out_bits
```

The size of our lookup tables for our decoding steps will be:

- rans_decode_table = `M` rows
- shrink_state_table = `M` rows
- expand_state_num_bits = `M` rows

So the total size of the tables is `3M`

The size of our lookup tables for our encoding steps will be:

- rans_encode_table = `M` rows
- shrink_state_out_bits = `A` rows
- shrink_state_threshold = `A` rows

A is our alphabet size, so our total size of the tables is `2A+M`

## Arithmetic Coding

Arithmetic coding is another form of entropy coding that solve some of the issues that occurs in block-based Huffman coding, since

1. The entire data is encoded as a single block
2. The codewords are computed on the fly, so no codebook is necessary
3. The compression efficiency is much closer to entropy, H(X) since we only have ~2 bits of overhead for the whole sequence. 

The Arithmetic encoding follows the steps below:

- First we have to find an interval or range `[L, H)` that corresponds to the entire sequence `x`.
    - Step 1: Start with some starting interval [L, H), which will be the [0, 1) and divide the interval based on the probability we see each symbol
    
    ```python
    alphabet = [A, B, C]
    prob_array = [0.3, 0.5, 0.2]
    cumul_array = [0.0, 0.3, 0.8, 1.0]
    
    [L,H) = [0, 1)
    A = [0, 0.3), B = [0.3, 0.8), C = [0.8, 1)
    ```
    
    - Step 2: Our new interval will be interval of the symbol we see
    
    ```python
    symbol = B
    [L,H) = [0.3, 0.8)
    ```
    
    - Step 3: We then subdivide our new interval based on our symbol probabilities again
    
    ```python
    [L,H) = [0.3, 0.8)
    BA = [0.3, 0.45), BB = [0.45, 0.7), BC = [0.7, 0.8)
    ```
    
    - Step 4: We will repeat Step 2 and 3 until we have encoded our entire sequence `x`
    
    ```python
    x = BACB
    # final interval
    x_interval = [0.429, 0.444)
    ```
    
- Now we have to communicate the interval `[L,H)`
    - Step 1: We pick the `Z` to be the midpoint of the interval `[L,H)`
    
    $$
    Z = \frac{L+H}{2}, Z = 0.5365
    $$
    
    - Step 2: We truncate `Z` to k bits (`Z_hat`) such that 1. `Z_hat` is still in the interval [L,H) and 2. that any extention of `Z_hat` in binary is also in the interval [L,H)
    
    ```python
    Z = 0.4365 = b0.01101111101...
    Z_hat = b0.011011111 ~ 0.4296
    ```
    

Therefore our final encoding is `encoded_bitArr = 011011111`. We must also encode how many symbols were encoded as `n`

The Arithmetic decoding follows the steps below:

- We start with our `Z_hat` value and the probabilities of our symbols as well as our initial interval
    - Step 1: find which interval `Z_hat` lies in and that will be the first symbol of `x`
    
    ```python
    Z_hat = 0.4296
    alphabet = [A, B, C]
    prob_array = [0.3, 0.5, 0.2]
    cumul_array = [0.0, 0.3, 0.8, 1.0]
    
    [L,H) = [0, 1)
    A = [0, 0.3), B = [0.3, 0.8), C = [0.8, 1)
    
    # Z lies in interval for B, so
    x = B
    [L,H) = [0.3, 0.8)
    ```
    
    - Step 2: divide the interval with the probabilities of our symbols again
    - Step 3: repeat steps 1 and 2 until we `length(x) = n`
    
    ```python
    [L,H) = [0.3, 0.8)
    BA = [0.3, 0.45), BB = [0.45, 0.7), BC = [0.7, 0.8)
    
    # Z lies in interval for BA, so
    x = BA
    [L,H) = [0.3, 0.45)
    
    ```
    

With Arithmetic coding we often see a better compression rate than Huffman coding, but the encoding and decoding speed is often worse than for Huffman coding. 

## Repcodes

The purpose of repcodes are to help compress structured data more efficiently and handle the scenario where the same offset is used frequently, or where it could be used frequency in the LZ77 algorithm. useful in the match finding process. Repcodes prove The repcode addition follows the following steps:

- Start `rep1`, `rep2`, and `rep3` with 1, 4, and 8 respectively.
- Continually update the three repcodes based on the last 3 used match offsets.

You can use repcodes to cheaply determine matches by having a “seed” for the most likely matches given by the `rep1`, `rep2`, and `rep3` values. The original repcode modeling algorithm changes the three repcode values based on the last used offsets. For the purposes of this project, we can store a dictionary of the offset numbers and maintain the repcodes to be the top three most used repcodes. Alternatively, we can test the efficiency improvement by matching the repcode implementation in zstd. 

# Methods

## Test Options

We will test a variety of different compression algorithms with changes to the original LZ77 algorithm. The algorithms we will test are detailed below.

- Original Huffman LZ77
- rANS LZ77
- tANS LZ77
- Arithmetic coding LZ77
- Original huffman LZ77 + repcodes
- rANS LZ77 + repcodes
- tANS LZ77 + repcodes
- Arithmetic coding LZ77 + repcodes

## Dataset Choices

We will test our algorithms on several different datasets including a text dataset of a book, random text, an array of RGB values, and the Github user information folder included as a part of our homeworks for this course. the book and random text will prove useful in testing the algorithms performance on English text with some repeating patterns but not many. The randomized ascii will remove the predictability of the English language and test the compressor on less predictable and repeatable data. The RGB values will imitate compression of an image. Lastly, the Github user information folder has a lot of repeats across files, so we will test which compressor performs the best across a dataset with repeats across a larger span.

# Progress Report

- We have spent time understanding the currently LZ77 algorithm currently implemented on SCL. This required us to look at the code in the scl library and read through line by line to understand how the LZ77 algorithm is currently implemented and find the parts of the algorithm we will be modifying. We also spent time understanding the alternative entropy coding algorithms we plan to implment: rANS, tANS, and Arithmetic coding. This required us to read through the code available in the SCL library to understand how it was implemented, as well as thoroughly read through and understand the class notes on these algorithms. (Note: the examples and explanations of these algorithms follow what was written in the class notes as that was the main source we use to understand these algorithms. These notes will be linked at the bottom of this report) Additionally, we learned and repcodes markdown file in the zstd github given to us by Shubham (linked at the bottom of this report). From there we discussed what data might have a better compression ratio with the implementation of repcodes. Finally, we discussed and chose data types that would be useful to test for all variations of our algorithm.
- The plan for the remaining weeks is to modify the LZ77 algorithm implemented on SCL. After we finish implementing the various versions of our algorithm, we will test each version of our algorithm on each of our chosen data types to see which performs the best for a given data type. We expect to spend the next week adapting the LZ77 algorithm and testing them, and the days leading up to the presentation creating our presentation.

## Our Predicted Results

- We expect the tANS, rANS, and Arithmetic coding to perform better than the original Huffman LZ77 in most if not all the test data. tANS, rANS, and Arithemetic coding should have a better compression rate, but the tradeoff will be a longer computation time. The addition of repcodes could be useful for book text and random ASCII characters would probably not help much, since there is little repitition in these files.
- Repcodes would be useful for structured repeating data where the offsets are more predictable. For example, with the Github data, it may be structured by the exact offset will be determined by the length of the username, which would change the offset for repeats between files. Images with repeating colors could benefit from the use of repcodes, so we would expect to see better compression performance with the versions of our algorithm that use repcodes.