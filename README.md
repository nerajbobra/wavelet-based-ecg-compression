# wavelet-based-ecg-compression

This project implements ECG compression using wavelet transforms and LZW lossless compression. This code is written in Python for ease of understanding and usage, but the algorithm is efficient enough that it can be written in C and run on an embedded system with reasonable memory and CPU resources.  

The motivation of this algorithm is to solve the problem of saving or wirelessly transmitting ECG data, which tends to have a high sampling rate. By compressing the signal, more data can be stored or wirelessly transferred using less power.    

An overview of the algorithm is as follows: 
1. Perform wavelet decomposition. The dB4 wavelet is used in this algorithm, with two levels of decomposition. 
2. Independently scale each of the wavelet coefficients to +/- 1 by looking for the absolute max in each of the coefficients, and dividing by that value. This a preprocessing step for the quantization step.
3. Quantize each of the scaled coefficients to between 8 bits and 2 bits. To quantize, multiply by 2^(num_bits-1) since these are signed values. Then round to the nearest integer. The stopping criteria is a 5% threshold on the percent root mean square difference (PRD) -- typically 4 bits is the lowest option to satisfy this requirement.
4. Combine all the wavelet coefficients from different decompositions into a single array. This will make the next step, the compression step, more straightforward. When combining all the coefficients together, this will result in a sparse array. To make compression more efficient, define a "binary map" which is an array of either zero or one. A zero indicates that there is a zero value at that index, while a one value indicates there is a non-zero value at that index. Then, remove all zero values from the combined array. Therefore, this steps yields two arrays: an array of length equal to the length of all the wavelet coefficients combined, with either a zero or a one value; and an array of the combined wavelet coefficients, with all zero values removed.
5. Convert the combined coefficients array and binary map array to a string of ascii characters. This step is to prepare the data for compression using the LZW compression algorithm. Since the coefficients may be negative, first add 2^(num_bits-1) to "bump" all the values up into the uint range.
6. Pass both ascii strings to the LZW compression algorithm. LZW compression is a relatively simple, but still quite effective, lossless compression algorithm. Any lossless compression algorithm can be used interchangeably in this step. 
7. Calculate the compression ratio by counting the number of bits in the compressed signal, and dividing by the number of bits in the original (uncompressed) signal. The number of bits in the input signal is calculated as the number of samples times the number of bits per sample. The number of bits in the compressed signal is calculated as the sum of the number of bits of each number in the compressed coefficients and the compressed binary map. Also, each of the scaling coefficients are added as 32 bit floating point numbers.
8. At this point, the compressed signal should be "saved" or "transmitted" and then "loaded" or "received" on another device. The remaining steps will be to reconstruct the signal from the compressed data. 
9. Decompress the coefficients array with zeros removed and binary map using LZW decompression. Then, form the original coefficient array with zeros.
10. Convert each ascii character back to an integer, and subtract by 2^(num_bits-1) to compensate for the addition of 2^(num_bits-1) in step 5.
11. Re-map the combined coefficients array back to separate decompositions. This should match the input to step 4.
12. Unscale the coefficients from +/- to their original range, based on the stored scaling factors. 
13. Do wavelet reconstruction on the unscaled coefficients. The output of this step is the final reconstructed signal.

The ECG data used in this project was downloaded from the PhysioNet Apnea-ECG Database:  
https://www.physionet.org/physiobank/database/apnea-ecg/

On the above PhysioNet dataset, an average compression ratio of 5.63 and PRD of 0.025 were achieved.  

The following figure shows the reconstruction at quantization of 8 through 4 bits for a specific ten second block of data:  
![](https://github.com/nerajbobra/wavelet-based-ecg-compression/blob/master/figs/PRD.png "Quantization")

To improve the results, several additional steps can be taken:
1. Try experimenting with different wavelets
2. Try using more levels of decomposition
3. Do wavelet coefficient thresholding prior to quantization
4. Allow a larger PRD threshold
5. Try different compression schemes such as Huffman Coding

The following papers were used as a reference for designing this algorithm:  
[An Efficient Coding Algorithm for the Compression of ECG Signals Using the Wavelet Transform](https://www.researchgate.net/profile/Bashar_Rajoub/publication/11423267_An_efficient_coding_algorithm_for_the_compression_of_ECG_signals_using_the_wavelet_transform/links/0912f5092ae7aeb7bf000000.pdf)  
[ECG Compression using Discrete Symmetric Wavelet Transform](https://ieeexplore.ieee.org/abstract/document/575053/)  
[A Wavelet Transform-Based ECG Compression Method Guaranteeing Desired Signal Quality](https://ieeexplore.ieee.org/document/730435/)  


