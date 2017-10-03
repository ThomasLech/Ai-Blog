---
<!-- layout: post -->
layout: post
title: Handwritten character segmentation
date: '2017-06-09 22:21:03 +0200'
published: true
comments: true
author: thomaslech
use_math: true
---

While performing [OCR][ocr-link] on a photo, we typically extract each character by detecting its bounding rectangle. However, this technique does not apply to cursive scripts where characters are linked with one another.

Let's consider two cases:
![output](https://user-images.githubusercontent.com/22115481/30865366-a7f3af02-a2d6-11e7-9aa0-85a5183d4ea1.png)

In order to separate linked characters shown in **_illustration (b)_**, our algorithm needs to deal with:

- [Typographic ligatures][typographic-ligatures]
- Slightly overlapping characters

![illustration3](https://user-images.githubusercontent.com/22115481/30874297-22f15d94-a2f0-11e7-87d3-214b8384f6b8.png)

### Solution
We propose an algorithm that relies on **sliding window** and **pre-trained classifier**.

Pre-trained classifier is capable of classifying all sorts of data (e.g. images) into one (can be multiple) of classes that appeared in **training dataset**.  
Technically, it is an array of numbers that were trained (optimized) in order to minimize so called **loss-function**. Loss-function tells us how big the total error is after summing up all incorrect guesses made by classifier.

### Training dataset
For our research purposes, we consider dataset containing **30211** samples representing **handwritten lowercase letters**.

Data description:
- Characters are indicated by black color, whereas background is white.
- Characters are thinned (they are 1 pixel thick).
- Characters are centered within **32x32** square box.

Visualization:

![lowercase_letters](https://user-images.githubusercontent.com/22115481/31047990-fe245bec-a614-11e7-87a8-f28010f589d2.png)

### Algorithm
1. **Find Outer ROI**

    We get Outer Region Of Interest by finding the bounding rectangle of linked characters. Such bounding rectangle appears in **_illustration (b)_** as marked red.

2. **Process Outer ROI**

    Apply same processing techniques that were applied to training set samples.

    ![roi](https://user-images.githubusercontent.com/22115481/30885563-088a3a4c-a314-11e7-9f7e-5f7ebef2972d.png)

3. **Initialize sliding window**

    The initial height of the window is equal to Outer ROI's height.  
    The initial width is given by the following condition:  
        Let $W$ be the width of Outer ROI. Let $w$ be the initial width of sliding window.  
        Then $w < W$.

    Specify **stride** parameter, which tells how far to the right, we move the sliding window.

    ![sliding_window](https://user-images.githubusercontent.com/22115481/30885850-3127921e-a315-11e7-9d4c-a2f4c7a9c96d.png)

4. **Find Inner ROI**

    We get Inner ROI by finding the bounding rectangle of the **blob** found within the sliding window.  

    A Blob is a group of connected pixels in an image that share some common property (e.g. grayscale intensity value). In our case, all connected black pixels found within the sliding window are considered as a blob.

    ![inner_roi_found](https://user-images.githubusercontent.com/22115481/30938497-78e604c6-a3da-11e7-9ef6-b47d109db3c9.png)

5. **Process Inner ROI**

    Fit Inner ROI into a square box, whose sides are equally long to the **longer side** of the Inner ROI. Center it inside, leaving white space on both sides. Last but not least, downsize the square box to training sample size.

    Resulting image is considered as **"Standardized Inner ROI"**.

    ![roi_processing](https://user-images.githubusercontent.com/22115481/31050091-56538384-a641-11e7-9a25-a47ef2257317.png)

6. **Run classifier**

    We employ pre-trained classifier to detect characters within an image, no matter they are linked with one another or slightly overlapping.

    Measuring the confidence, the classifier outputs, we might reject sliding window contents as representing a character.

    If the **highest output confidence** is greater than the **confidence threshold** parameter, then we keep Inner ROI as a candidate for representing a valid letter. Specifically, we keep Inner ROI's properties, like its coordinates and dimensions.

    ![probabilities](https://user-images.githubusercontent.com/22115481/31059974-ca0700d4-a70a-11e7-9f9d-43e960e6dd48.png)

7. **Move the window**

    Move sliding window to the right by the number of pixels, specified by the **stride** parameter.

8. **Repeat the process**

    - If sliding window hasn't reached the end yet:

        Repeat Steps 4-7.

        ![next_step](https://user-images.githubusercontent.com/22115481/31127538-93914c26-a84f-11e7-9059-8fe798d50e9c.png)

        ![probabilities](https://user-images.githubusercontent.com/22115481/31127638-d644a1ee-a84f-11e7-99fc-d86ebe6003a8.png)

    - If sliding window has reached the end:

        Make sliding window wider, and repeat the process.
    - If sliding window's width is equal to Outer ROI's width:

        Extract characters.

### Extracting characters

Last but not least, we disqualify all candidates that are contained within other candidates.

If overlap of one candidate over the other is smaller than **overlap threshold** parameter, both candidates are qualified. Where overlap threshold parameter is an arbitrary number greater than 0.

<!-- - Contained within another candidate.

    Example: 'n' letter inside 'm' letter.
    ![double_n](https://user-images.githubusercontent.com/22115481/31135623-da24db10-a865-11e7-8309-f74dc6ae7b2e.png)

- Overlapping too much other candidates.

    Example: 'u' between 'a' and 'm' letters. -->

[ocr-link]: https://en.wikipedia.org/wiki/Optical_character_recognition
[typographic-ligatures]: https://en.wikipedia.org/wiki/Typographic_ligature
<!-- [extract-characters-by-its-bounding box]:  -->
