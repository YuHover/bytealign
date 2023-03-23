# Does data alignment matter today?

"Data alignment" seems to be an important rule in programming, I have often heard people mention that "data alignment" or "memory alignment"  greatly affects the program's performance. There are also many exercises on "data alignment", such as calculating the number of bytes occupied by a specific struct type.

Thanks to compilers, I have never considered "data alignment" in practical programming. But as we all know, no all-round knowledge can be acquired merely by relying on hearsay.

Inspired by [Data alignment: Straighten up and fly right](https://developer.ibm.com/articles/pa-dalign/), [Data alignment for speed: myth or reality?](https://lemire.me/blog/2012/05/31/data-alignment-for-speed-myth-or-reality/), [Aligned vs. unaligned memory access](http://www.alexonlinux.com/aligned-vs-unaligned-memory-access) and many other blogs, I conduct two experiments to see "Does data alignment matter today?".

## Problem definition

Simply put, "data alignment" refers to storing data at an address that is an integer multiple of its size (the real situation is not so simple, such as the alignment of struct type).

In computing, a word is the natural unit of data used by a particular processor design. When the processor reads from the memory subsystem into a register or writes a register's value to memory, the amount of data transferred is often a word.<sup>[1]</sup> In most circumstances, the word size of a 32-bit CPU is 32 bits, and the word size of a 64-bit CPU will be 64 bits.

Accessing at a misaligned address often causes the CPU has to read mutiple words to fetch data we require. Please see Figure 1 and Figure 2 below. When we read an 8-byte data (may be a 64-bit integer, like `long long` type in c++ or `int64` type in go) at address 0, the CPU (assuming that the word size of this CPU is 8 bytes) only needs to read one word to obtain the data we need, but has to get an extra word when the 8-byte data is stored at address 4. Obviously, the CPU also has to shift out unwanted bytes and merge the left bytes of two words. Figure 3 and Figure 4 present the the scenarios of reading 4-byte data at the aligned and misaligned address.

<div align='center'>
    <img src=".\images\read_8_bytes_data_at_aligned_address.png" style="zoom: 80%;" />
    <div align='center'>
        <font size='2'>Figure1. read 8-byte data at an aligned address</font>
    </div>
</div>
<br/>
<br/>
<div align='center'>
    <img src=".\images\read_8_bytes_data_at_misaligned_address.png" style="zoom: 80%;" />
    <div align='center'>
        <font size='2'>Figure2. read 8-byte data at a misaligned address</font>
    </div>
</div>
<br/>
<br/>
<div align='center'>
    <img src=".\images\read_4_bytes_data_at_aligned_address.png" style="zoom: 80%;" />
    <div align='center'>
        <font size='2'>Figure3. read 4-byte data at an aligned address</font>
    </div>
</div>
<br/>
<br/>
<div align='center'>
    <img src=".\images\read_4_bytes_data_at_misaligned_address.png" style="zoom: 80%;" />
    <div align='center'>
        <font size='2'>Figure4. read 4-byte data at a misaligned address</font>
    </div>
</div>

## Environment

- os: Windows 11
- CPU: Intel(R) Core(TM) i5-10600KF CPU @ 4.10GHz
- programing languages: go 1.18 and Python 3.8.11 (for visualizing experimental results)

## Experiment 1

In a large enough buffer, read `sizeof(T)` bytes at `addr`, `addr+256`, `addr+256+256` respectively, and so on until a total of `N` bytes data is read.

- The first address `addr` is aligned to 0, 1, 2, 3, ... 255, and so are following addresses `addr+256`, `addr+256+256`, etc. addr≡(addr+256)(mod 256).
- Type `T` refers to `int8`, `int16`, `int32` and `int64` .
- `N` is set to 16M.

- Skip 256 bytes per read, which ensures the data we will read is not in cache yet. (My CPU is Intel core i5-10600KF whose cache line size is 64 Bytes). For more information, see [How do cache lines work?](https://stackoverflow.com/questions/3928995/how-do-cache-lines-work).

Figure 5 shows an example of this experiment. Type `T` is selected as `int32` which occupies 4 bytes space in go. Firstly, the `addr` is aligned to 2, that is to say "addr % 256 == 2". Fetching a `int32` type of data at `addr` (or `addr+256`, etc), the CPU solely need to read one word. We denote the time it takes to read 16MB in this case as `t1`. But when the `addr` is aligned to 5, something bad happens. The CPU has to do more work because a data type of `int32` crosses two words. The time it takes to read 16MB of data is denoted as `t2` in this case. According to our analysis `t2` should be greater than `t1`, but is it really? Let's test our conjecture with experimental results.

<div align='center'>
    <img src=".\images\experiment_1_general_view1.png" style="zoom: 80%;" />
    <div align='center'>
        <font size='2'>Figure5. an example of experiment 1 </font>
    </div>
</div>
<br/>
<br/>

Figure 6 depicts the trend of time taken by the program to read all 16MB of `int32` data when `addr` is aligned to 0 to 255. The vertical dashed lines marks the alignments which cause data of type `int32` to cross two words. Figure 7 is a close-up of a part of Figure 6. This two figures clearly show that when `addr` is aligned to the addresses marked by vertical dashed lines, the program takes more time to read all data than when `addr` is aligned to other addresses.

It is also worth noting that when the alignment is 61, 62, 63 (nearby 64), 125, 126, 127 (nearby 128), 189, 190, 191 (nearby 192), 253, 254, 255 (nearby 256), the program takes the most time. The same phenomenon can be seen in the results of type `int16` and `int64` below. In short, a combination of data misalignment and the cpu cache mechanism causes this behavior. [This blog](http://www.alexonlinux.com/aligned-vs-unaligned-memory-access) explains this behavior well. Details will also be given at the end of this section.

<div align='center'>
    <img src=".\images\avoid_cache_four.png" />
    <div align='center'>
        <font size='2'>Figure6. experiment results of int32, the clearer picture is <a href=".\images\avoid_cache_four.png">here</a></font>
    </div>
</div>
<br/>
<div align='center'>
    <img src=".\images\avoid_cache_four_close_up.png" />
    <div align='center'>
        <font size='2'>Figure7. a close-up of Figure6 </font>
    </div>
</div>
<br/>
<br/>

As can be seen from Figure8 and Figure 9 which show the results of `int16`, the program takes more time to read data when `addr` satisfies "addr % 8 == 7", namely the data of type `int16` at `addr` will cross two words.

<div align='center'>
    <img src=".\images\avoid_cache_two.png" />
    <div align='center'>
        <font size='2'>Figure8. experiment results of int16, the clearer picture is <a href=".\images\avoid_cache_two.png">here</a></font>
    </div>
</div>
<br/>
<div align='center'>
    <img src=".\images\avoid_cache_two_close_up.png" />
    <div align='center'>
        <font size='2'>Figure9. a close-up of Figure8 </font>
    </div>
</div>
<br/>
<br/>

The aligments for `int64` which are marked by vertical dashed lines in Figure 10 are what the cpu is willing to meet. Accessing the data of `int64` type which is aligned with these alignments, CPU can work like Figure 1 in **Problem definition** section. The rest of the alignments will result in the scene presented in Figure 2.
<div align='center'>
    <img src=".\images\avoid_cache_eight.png" />
    <div align='center'>
        <font size='2'>Figure10. experiment results of int64, the clearer picture is <a href=".\images\avoid_cache_eight.png">here</a></font>
    </div>
</div>
<br/><br/>
