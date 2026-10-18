# CEPARCO Integrating Project: ScanFast
Group 5

Dimaunahan, Chelsea Jei

Infante, Gabriel 

Maravilla, Sofia

Melanio, Marion Zane

## Link to Video
[Google Drive](https://drive.google.com/file/d/1YYYIeGDxYGFoMxkN7ufSHgIsvp9HiWfq/view?usp=sharing)

## Abstract
ScanFast aims to be a low-latency image processing engine written in C and vectorized with target-specific SIMD intrinsics with the intention of outperforming upstream acceleration filters for existing QR code decoding libraries like ZXing-C++ and Quirc. ScanFast processes raw camera frames through vector-parallelized grayscale conversion, adaptive integral-image binarization, and selective median filtering. By replacing standard sequential scalar loops with custom 256-bit vector intrinsics, the project targets a 4x to 8x acceleration in image binarization, enabling high-speed inventory automation and real-time processing of corrupted, unevenly lit printed materials.

## Description
### A. Scope and Limitations 
The project aims to use an existing decoder instead of making a QR recognition engine from scratch. It will not find, orient, or parse the bit-matrix data of QR codes. 
### B. Inputs
The inputs of the project would involve different image formats, including PNG, JPEG, raw RGB/RGBA arrays, and the like.

As part of testing, different image formats would be in different conditions, such as:
- Standard, clear QR codes.
- QR codes with salt-and-pepper noise.
- QR codes under uneven lighting conditions.

### C. Proposed Process
#### 1. Grayscale Conversion 
Images are basically made up of arranged pixels, and their size is represented by a matrix. They are functions f(x,y), taking up values in {0, 1, ..., 255}, where x and y are the coordinates of the pixels. (Cunibil, 2026)

Grayscale is an image made up of various shades of gray, from black to white, simplifying image data while still retaining and enhancing the important visual details through its luminance (white as brightest, black as darkest, and shades of gray as everything in between) (What Is Grayscale? Definition, Uses, and Applications in Digital Imaging | Lenovo US, 2023). 

According to Linkgit (2024), the mathematical conversion of images to grayscale would involve an algorithm of summing each pixel’s red, green, and blue values, with weights assigned based on the luminous intensity of a color as perceived by the human eye. Its general conversion is:

$$Grayscale = (0.299 * Red) + (0.587 * Green) + (0.114 * Blue)$$

Converting an image to grayscale for the project would involve using 8-bit grayscale as using 16-bit or floating point would consume more memory, increasing latency. 8-bit grayscale has 256 different kinds of gray, and can take values between 0 and 255. 16-bit grayscale has 65,536 different shades of gray, which offers increased levels of intensity and differentiation between pixels. Floating point grayscale has positive floating-point pixel intensity, which in theory, has no limitations to the amount of gray levels possible, and would give high accuracy of images (Linkgit, 2024).

Implementing this with SIMD:
- The conversion formula would be converted into integer arithmetic Y = (77R + 150G +29B) >> 8 for efficient packed integer operations.
- The RGB values will be in 256-bit YMM registers and expanded from 8-bit to 16-bit words.
- The RGB values will be multiplied in parallel by the fixed weights across 16 pixels, summed and shifted to the right by 8 bits to perform a faster division by 256, and then the processed 16-bit values will be packed back into 8-bit byte vectors to write out 32 grayscale output pixels per iteration using vector instructions.

#### 2. Denoising
Images often times contain noise. Processing these images without filtering would result with an unreliable output, can even lead to the QR decoder not scanning the image properly. To filter out the salt-and-pepper noise within images, the project would implement a Selective Median Filter inspired by an algorithm by Zhu and Huang (2012).

Noise typically does not take up 100% of the  image. The Standard Median Filter would have to go through the whole image, making it inefficient. Using this filter would also have the risk of blurring the edges, which is crucial for QR decoders.

Zhu and Huang’s algorithm doesn’t go through the whole image. Instead, it checks if a pixel is noise by looking at the 3x3 grid around it (the pixel itself plus 8 immediate neighbors), finding the average of the 9 pixels to get the average brightness, and subtracting the center or target pixel’s value from the average. If there is a big absolute difference between the average and the pixel’s value compared to the threshold (e.g. |120-0| = 120 > threshold, which we could put as 30 as an example), it would be considered noise. Otherwise, it would be considered clean and the algorithm skips it. If a pixel is considered as noise, the algorithm would find the median of the 3x3 grid and replace the value of the noisy pixel with the median. 

However, it is important to note that the “big/small absolute difference” would depend on the threshold set, and different values will be benchmarked throughout the project to see what threshold gives the best outcome.

Implementing this with SIMD:
- Using the horizontal addition operation can speed up the addition process of multiple pixels at once. 
- Vector subtraction will be used to compute the difference between local averages and center pixels.
- Vector Min/Max operations will be used for finding the median without the need for scalar loops or branching. 

#### 3. SIMD Parallel Adaptive Binarization
Binarization refers to the process of taking a grayscale image and segmenting it to its light and dark components, and producing an grayscale output of using only black and white. It is essential clarifying edge and object detection. With the intention of performing to be a QR decoder, it should be able to identify the edges of even and uneven printed codes in various lighting settings. 

Parallel adaptive binarization has a varying threshold to navigate between high-intensity and low-intensity settings, and identify when to invert a grayscale image in-case it is in high-intensity lighting through segmentation using integral images (Peuwnuan et al., 2016).

The process of outputting a binary grayscale image involves taking the grayscale image, performing segmentation using integral image (SUI), and performing modified adaptive thresholding (MAT). Classifying for SUI involves using the following equation:

$$R_s \le C \times (M + sd)$$ 

Where the sum of the rectangular area is used to calculate a local mean and compared with the mean value of the whole image. $$M$$ is the global mean and $$sd$$ is the global standard deviation. This determines whether the image is a low or high-intensity image for binarization. The equations below are used for binarization for high-intensity and low-intensity, respectively:

$$C \times i'(x, y) \ge R'_s \times \left(1 - \frac{T}{100}\right)$$ 
$$R_s \ge C \times i(x, y)$$ 

$$i’$$ is any point at $$(x,y)$$ on the complementary gray scale image and $$R’_s$$ is the $$R_s$$ at $(x,y)$. For a low-intensity image, all values are calculated from the local mean in a specified area ($$R$$). 

### D. Comparison with Existing Implementations
There are already existing libraries that can scan and decode QR codes, such as ZXing-C++ and Quirc. However, ScanFast focuses on improving the image preprocessing stage before the QR code is scanned. Instead of creating its own QR code decoder, it aims to make image processing faster using AVX2 SIMD instructions and improve the quality of images with noise and uneven lighting.

ZXing-C++ is a library that can detect and decode different types of barcodes, including QR codes. It processes the input image and converts it into black-and-white pixels using thresholding techniques. One of its methods, called Hybrid Binarizer, uses local areas of the image to determine which pixels should be black or white. This helps it handle images with uneven lighting. In comparison, ScanFast will use its own adaptive binarization method based on integral images and apply a selective median filter to reduce salt-and-pepper noise. It will also use AVX2 instructions to process multiple pixels at once. The project will test whether these changes can reduce processing time and improve QR code scanning success compared with the existing implementation.

Quirc is a lightweight QR code scanning library written in C. It accepts an 8-bit grayscale image and uses Otsu's thresholding method to convert the image into black-and-white pixels. It then looks for QR code patterns and decodes the detected codes. Unlike the proposed ScanFast pipeline, Quirc does not include the same combination of adaptive binarization and selective median filtering. ScanFast will preprocess the image before passing it to Quirc, allowing the project to test whether the processed image helps Quirc detect and decode QR codes more successfully, especially when the original image contains noise or uneven lighting.

ScanFast differs from these libraries because its main purpose is to improve image preprocessing rather than perform QR code detection and decoding itself. It will use a sequential C implementation as the baseline and compare it with an AVX2-optimized implementation. The preprocessing steps will include grayscale conversion, adaptive binarization, and selective median filtering. After preprocessing, the resulting image will be passed to an existing QR code decoder, with Quirc as the initial integration target.

To compare the implementations, the project will test QR code images under different conditions, including clear images, images with salt-and-pepper noise, and images with uneven lighting. The results will be evaluated based on processing time, CPU clock cycles per pixel, and QR code decoding success rate before and after applying ScanFast. This will help determine whether the proposed implementation can process images faster and improve QR code scanning results compared with the existing approaches.

### E. Outputs 
The project is expected to produce a working prototype of ScanFast, an image preprocessing program written in C and optimized using AVX2 SIMD instructions. It will process QR code images before passing them to an existing QR code decoder. The expected outputs are the following:

1. Preprocessed Image Matrix

The main output of ScanFast will be a black-and-white image matrix represented using either 1 bit per pixel or 8-bit binary values. This image will be used as input for QR code detection and decoding. Before producing the output, ScanFast will perform grayscale conversion, adaptive binarization, and selective median filtering to reduce noise and improve the visibility of QR code patterns.

The resulting image will be tested using clear QR codes, images with salt-and-pepper noise, and images with uneven lighting. These tests will help determine whether the preprocessing steps can preserve important QR code patterns and improve the quality of the input given to the decoder.

2. Performance Benchmark Report

ScanFast will generate a report comparing the performance of the sequential C implementation and the AVX2-optimized implementation. The report will include the following measurements:

Processing time (milliseconds per megapixel): Measures how long each implementation takes to process an image, adjusted for its size.

CPU clock cycles per pixel (CPK): Measures the average number of CPU clock cycles needed to process each pixel.

Speedup: Compares the processing time of the sequential C implementation with the AVX2 implementation to determine how much faster the optimized version is.

The tests will use the same images and conditions for both implementations to ensure a fair comparison. The results will show whether using AVX2 instructions improves the speed of image preprocessing.

3. QR Code Decoding Success Rate

The project will also produce a comparison of QR code decoding success rates before and after applying ScanFast's preprocessing. The tests will use the same degraded QR code dataset, particularly images affected by salt-and-pepper noise and uneven lighting.

The decoding success rate will be calculated using the following formula:

$$\text{Success Rate} = \frac{\text{Successfully Decoded QR Codes}}{\text{Total QR Codes Tested}} \times 100%$$

The results will show the percentage of QR codes successfully decoded using the original images and the preprocessed images. This will help determine whether ScanFast's preprocessing improves decoding success under different image conditions.

4. Integrated Prototype

The final output will be a working prototype that connects ScanFast to an existing QR code decoder, with Quirc as the initial integration target. It will demonstrate the complete process, from loading a QR code image and applying preprocessing to passing the resulting image to the decoder.

The prototype and its test results will be used to evaluate the performance of the sequential C and AVX2 implementations, as well as the effect of preprocessing on QR code decoding success. Overall, the project aims to demonstrate whether AVX2-based image preprocessing can reduce processing time while helping existing QR code decoders handle degraded images.

## Implementation
ScanFast will be developed using C and x86-64 Assembly language on a compatible computer system. C will handle image loading, memory management, execution-time measurement, and integration with an existing QR code decoding library. NASM will be used to assemble the x86-64 Assembly code, while GCC will compile the C source code and link the required components. The final development environment and libraries will be selected based on compatibility with the group's hardware and software setup.

The project will implement image preprocessing using two approaches: a sequential implementation in C and an optimized implementation using SIMD with AVX2 instructions. The sequential implementation will serve as the baseline for evaluating the performance of the parallelized version. Both implementations will perform the same image-processing operation to ensure a fair comparison.

The initial image-processing operations will be grayscale conversion, denoising, and adaptive image binarization. These operations will be applied to QR code images by AVX2 vectorization before passing them to an existing QR code decoding library.

For the AVX2 implementation, 256-bit vector registers will be used to process multiple pixel values in a single vector operation. For example, a 256-bit vector can contain 32 eight-bit values. The implementation will process groups of pixels using AVX2 instructions, while any remaining pixels that do not form a complete vector will be processed using a scalar loop. The program will also check whether the target processor supports AVX2 before executing the optimized implementation.

To evaluate the effectiveness of SIMD, both implementations will be tested using identical QR code images of different dimensions and visual conditions. The outputs will be checked for correctness, and execution time will be measured across multiple runs. The results will be used to calculate speedup using the following formula:

$$Speedup = Sequential Execution Time / AVX2 Execution Time$$

The processed images will also be evaluated using an existing QR code decoder to determine whether the QR content can be recovered. The project will report execution times, speedup, output correctness, and QR code decoding results. These measurements will help determine whether AVX2-based data-level parallelism improves image preprocessing performance compared with the sequential implementation.

The expected outcome is a working prototype that demonstrates the application of SIMD parallelism to QR code image preprocessing, together with a performance analysis comparing the sequential C and AVX2 Assembly implementations.
## Midproject Milestones 
### Milestone 1 — Baseline Infrastructure & Dataset Setup:
Build a test harness in C loading test images (clean, salt-and-pepper, uneven lighting).
Integrate baseline Quirc library and establish scalar execution performance baseline.
### Milestone 2 — SIMD Grayscale & Selective Denoising Kernel:
Implement vector-accelerated RGB-to-Grayscale conversion and selective median filtering using AVX2 intrinsics.
### Milestone 3 — SIMD Adaptive Local Binarization:
Implement vector-accelerated integral image thresholding.
### Milestone 4 — Integration and Benchmarking:
Connect ScanFast preprocessor output directly to Quirc search engine.
Generate benchmarking outputs.
## References
What is Grayscale? Definition, Uses, and Applications in Digital Imaging | Lenovo US. (2023, May 28). https://www.lenovo.com/us/en/glossary/grayscale/?orgRef=https%253A%252F%252Fwww.google.com%252F&srsltid=AU7gw4VwlSKMHXb5wpxaZlYoPk4ofiV7Azv_BR5A3UgkMDAu037RpcC7

Cunibil, C. (2026, September 30). An image as a matrix. Tangente Magazine. https://tangente-mag.com/en/articles/an-image-as-a-matrix 

K. Peuwnuan, K. Woraratpanya and K. Pasupa, "Modified adaptive thresholding using integral image," 2016 13th International Joint Conference on Computer Science and Software Engineering (JCSSE), Khon Kaen, Thailand, 2016, pp. 1-5, doi: 10.1109/JCSSE.2016.7748917. 

Linkgit. (2024, May 18). Why use Grayscale Conversion when Processing Images? Grayscale Images. https://www.grayscaleimage.com/why-use-grayscale-conversion-when-processing-images/ 

Quirc. QR decoder library. https://github.com/dlbeer/quirc

Zhu, Y., & Huang, C. (2012). An improved median filtering algorithm for image noise reduction. Physics Procedia, 25, 609–616. https://doi.org/10.1016/j.phpro.2012.03.133

ZXing-C++. ReadBarcode.cpp and ReaderOptions.h. https://github.com/zxing-cpp/zxing-cpp
