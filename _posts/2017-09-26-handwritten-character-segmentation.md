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

Pre-trained classifier is capable of classifying all sorts of data, like images into one (can be multiple) of all classes that appeared in **training dataset**.  
Technically, it is an array of numbers that were trained in order to minimize so called **loss-function**. Loss-function tells us how big the error is after summing up all incorrect guesses this classifier has made.

### Algorithm
* **Find ROI**  
  Region Of Interest can be found by finding the bounding rectangle of linked characters. Such bounding rectangle appears in **_illustration (b)_** as marked red.

* **Process ROI**  
  Apply all image processing techniques that are required by the classifier before fed.  
  The choice of techniques to be used depends on what techniques were applied to **training set**.

  ![roi](https://user-images.githubusercontent.com/22115481/30885563-088a3a4c-a314-11e7-9f7e-5f7ebef2972d.png)

* **Initialize sliding window**  
  The initial height of the window is equal to ROI's height.  
  The initial width is given by the following condition:  
    Let $W$ be the width of ROI mentioned in the previous step. Let $w$ be the initial width of sliding window.  
    Then $w < W$.

  ![sliding_window](https://user-images.githubusercontent.com/22115481/30885850-3127921e-a315-11e7-9d4c-a2f4c7a9c96d.png)

* **Find blob within the window**  
  ...

* **Standardize**  
  Fit sliding window contents into a square box. This square box always needs to have fixed size.
  ...

* **Run classifier**  
  ...

* **Move sliding window**  
  ...

_... more content ..._

[ocr-link]: https://en.wikipedia.org/wiki/Optical_character_recognition
[typographic-ligatures]: https://en.wikipedia.org/wiki/Typographic_ligature
<!-- [extract-characters-by-its-bounding box]:  -->
