# wavelet-based-ecg-compression
  
This project implements ECG compression using wavelet transforms and Variable Run-Length Encoding. This code is written in Python for ease of understanding and usage, but the algorithm is efficient enough that it can be written in C and run on an embedded system with reasonable memory and CPU resources.  
  
The motivation of this algorithm is to solve the problem of saving or wirelessly transmitting ECG data, which tends to have a high sampling rate. By compressing the signal, more data can be stored or wirelessly transferred using less power.    

On the PhysioNet dataset used in this project, an average compression ratio of 12.8 was achieved with an average PRD of 0.385.
  
An overview of the algorithm is as follows: 
## 1. Pre-processing Raw Waveform
Reducing the amount of information in the signal prior to compression will intuitively improve the compression ratio. As a simple pre-processing step, detrend the signal to remove and drift or DC offset. This is generally not considered useful information so it will be lost going forward.  
  
Below is a excerpt of the dataset used in this project, both before and after detrending (note that the signal has very little drift or DC offset, so detrending effectively makes no difference):  
![](https://github.com/nerajbobra/wavelet-based-ecg-compression/blob/master/figs/detrending.png "Detrending") 
  
## 2. Wavelet Decomposition
In this project, the bior family of wavelet is utilized. Specifically, bior4.4 is used with 5 levels of decomposition. Below is an example of a wavelet decomposition of a 10 second block of pre-processed ECG data:
![](https://github.com/nerajbobra/wavelet-based-ecg-compression/blob/master/figs/wavelet_decomposition.png "Wavelet Decomposition")  
  
## 3. Coefficient Thresholding
Depending on the decomposition level, many of the wavelet coefficients can be very small (close to zero) while just a handful of coefficients represent the vast majority of the total energy. To more efficiently represent the coefficients, do coefficient thresholding to retain the following percentages:  
-approximate coefficients: 99.9%  
-detail level 5 coefficients: 97%  
-detail level 1 to 4 coefficients: 85%  
  
The exact thresholds, and algorithm for thresholding, is detailed in the paper *An Efficient Coding Algorithm for the Compression of ECG Signals Using the Wavelet Transform* (linked below).
  
After thresholding, a binary map array is created that indicates whether any given value in the thresholded coefficients array is either zero or nonzero. If the coefficient was set to zero, the value in the binary map at the index of the zeroed coefficient is zero. Otherwise, the value in the binary map at the index of the coefficient is one.

A visual explanation of the above process:    
<img src="https://github.com/nerajbobra/wavelet-based-ecg-compression/blob/master/figs/binary_map_example.jpg" alt="Binary Map Example" width="562" height="300">
  
Below is an example of before and after coefficient thresholding:
![](https://github.com/nerajbobra/wavelet-based-ecg-compression/blob/master/figs/wavelet_thresholding.png "Wavelet Thresholding")  
  
## 4. Scale Thresholded Wavelet Coefficients
Before the wavelet coefficients can be quantized, they need to be scaled to the [0,1] range. To do this, each of the decoposition levels is shifted by the minimum value, and then divided by the maximum value. Both these values must be stored as floats for later reconstruction. Assuming each float is 32 bits, this step costs 32x2x6 = 384 bits. 
  
Below is an example of the wavelet coefficients from step 3 scaled to [0,1]:
![](https://github.com/nerajbobra/wavelet-based-ecg-compression/blob/master/figs/wavelet_scaled.png "Wavelet Scaling")  

## 5. Calculate Minimum Number of Bits for Quantization
Each wavelet coefficient at this point is a float in the range [0,1]. To quantize the coefficients to N bits, one would multiply each float by 2^(N)-1 and then round to the nearest integer. The smaller the value of N, the higher the compression ratio will be. The question is, how do we determine what the lowest acceptable value for N is?  
  
The percentage root-mean-square difference is used as a metric to answer this question. The formula for the PRD is:  
![](https://github.com/nerajbobra/wavelet-based-ecg-compression/blob/master/figs/PRD_eqn.gif "PRD Equation")  
  
where Xi is the original signal and Xi bar is the reconstructed signal.
  
Starting at N=8, the signal is quantized, reconstructed, and the PRD associated with that value of N is calculated. N is then decremented by 1, and the process is repeated until the PRD crosses the defined threshold of 0.4 (which was determined through trial and error).
  
Below is a plot of the signal reconstructed with N = [8,7,6] and the associated PRD values:  
![](https://github.com/nerajbobra/wavelet-based-ecg-compression/blob/master/figs/PRD.png "PRD Equation")  
  
## 6. Quantization at N bits
In this step, simply quantize the signal for the value of N determined in the previous step.
  
For this particular 10 second segment, N=6. Below is a plot of the quantized signal:  
![](https://github.com/nerajbobra/wavelet-based-ecg-compression/blob/master/figs/wavelet_quantized.png "Wavelet Quantized")  
  
## 7. Combine Wavelet Coefficients/Binary Maps
Up until this step, the wavelet coefficients and their corresponding binary maps are all separated in their corresponding decomposition levels. This step simply combines them all into a single flat array. This will help increase the compression ratio.
  
## 8. Compress Wavelet Coefficients
For this particular block of data, N=6. Therefore, each quantized coefficient only has 6 bits of information. Yet, each coefficient is occupying 8 bits of space (ie, each coefficient is being stored as a byte). Simple bit packing is the compression algorithm for the wavelet coefficients.  
  
Below is a visual example of the bit packing process for an example where N=4:
<img src="https://github.com/nerajbobra/wavelet-based-ecg-compression/blob/master/figs/coefficient_quantization_example.jpg" alt="Coefficient Compression" width="464" height="500">
  
## 9. Compress Binary Map
The binary map consists of lots of repeated one's and zero's. One of the easiest (and quite effective) ways of compressing data with repetitive patterns is Variable Run-Length Encoding. The premise is to represent each number as a count. Since there is always going to be either a zero or a one in the binary map, and it will always switch, only the following two pieces of information need to be stored:  
-the initial value (either a zero or a one)  
-the run count, which is reset every time we switch from zero to one or vice versa

The diagram below explains the encoding scheme for the implementation of Variable Run-Length Encoding:
<img src="https://github.com/nerajbobra/wavelet-based-ecg-compression/blob/master/figs/RLE_format.jpg" alt="CRLE" width="464" height="400">


  
## 10. Calculate Compression Ratio
The compression ratio is calculated assuming the following information is being "transmitted" for reconstruction on the receiving end:  
-the compressed coefficients (represented as bytes)  
-the number of bits associated with the last byte of the compressed coefficients (1 byte)  
-the scaling factors for each wavelet decomposition (6 decomposition levels x 2 scaling coefficients each = 12 float)  
-the number of bits used to quantize the coefficients (1 byte)  
-the compressed binary map (represented as bytes)  
-the number of bits associated with the last byte of the compressed binary map (1 byte)  
-the initial state of the binary map (one byte)  
  
## 11. Decompress Binary Map
Simply repeat the compression step in reverse

## 12. Decompress Coefficients
Simply repeat the compression step in reverse
  
## 13. Remap Coefficients
Since the length of each of the decomposition levels are always constant, this is simply a process of re-separating the combined array into it's original 6 arrays.  
  
## 14. Unscale Coefficients
Using the scaling coefficients that were stored through the compression step, re-scale the coefficients back to their original scale.  
  
Below is a plot of the coefficients before scaling, and after unscaling:
![](https://github.com/nerajbobra/wavelet-based-ecg-compression/blob/master/figs/wavelet_rescaled.png "Unscaling Coefficients")  
  
## 15. Wavelet Reconstruction
Finally, do wavelet reconstruction of the uncompressed signal.
  
Below is a plot of the original signal and the final, uncompressed, reconstruced signal:  
![](https://github.com/nerajbobra/wavelet-based-ecg-compression/blob/master/figs/reconstructed.png "Wavelet Reconstruction")  
    
Note that the compression ratio for this particular block was 12.0  
  
## Other Notes
The ECG data used in this project was downloaded from the PhysioNet Apnea-ECG Database:  
https://www.physionet.org/physiobank/database/apnea-ecg/
  
The following papers were used as a reference for designing this algorithm:  
[An Efficient Coding Algorithm for the Compression of ECG Signals Using the Wavelet Transform](https://www.researchgate.net/profile/Bashar_Rajoub/publication/11423267_An_efficient_coding_algorithm_for_the_compression_of_ECG_signals_using_the_wavelet_transform/links/0912f5092ae7aeb7bf000000.pdf)  
[ECG Compression using Discrete Symmetric Wavelet Transform](https://ieeexplore.ieee.org/abstract/document/575053/)  
[A Wavelet Transform-Based ECG Compression Method Guaranteeing Desired Signal Quality](https://ieeexplore.ieee.org/document/730435/)  
