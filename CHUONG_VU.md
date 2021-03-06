# Image to Sketch / Posterizing

## Chuong Vu
### April 28, 2017

# Overview

This set of code read all the pixels from an image, calculate all the values and create a new sketch image.

**Authorship note:** All of the code described here was written by myself.

# Libraries Used
The code uses four libraries:

```racket
(require images/flomap)
(require (except-in racket/draw make-pen make-color))
```

* The ```images/flomap``` library is used to read pixel from bitmap% and then convert back to bitmap%.
* The ```racket/draw ``` library is used convert image to bitmap% and convert bitmap% to image.

# Key Code Excerpts

The key of this is storing data (RGB) as 24 bits binarys into a single list. So, it save the memory and improve the reading time out of memory. Also, by using recursion to loop through the list that I have learned in UMass Lowell's COMP.3010 OPL class. I optimized the running time of the program to O(N).

Five examples are shown and they are individually numbered. 

## 1. Initialization Global Bitmap% Object

The ``imginput`` bitmap% object will be use through all the code. It is a object of pixel-based image.
 
```racket
;; Read image to bitmap% object
(define imginput (make-object bitmap% img-name))

;; allocate the memoery for pixels
(define pixels (make-bytes (* img-height img-width 4)))

;; read the argb value and store to pixels
(send imginput get-argb-pixels 0 0 img-width img-height pixels)

;; convert pixels to list for calcualte
(define PixelsList (bytes->list pixels))
 ```
 
 The pixels list itself after get-argb-pixels, the value will be something like ``#"\377\276\255\235\377\276\255\...``. So it is necessary to convert it to list as ``'(255 190 173 157 255 190 173 157 255 190...)`` which is store as (A R G B A R G B A R G B...) so this will be easy for read out the value to calcuate.


## 2. Recursive Interative Process Pixels List

By using the recursion function. I read RGB in the PixelsList. Passed it to join-value function which is used to convert to a 24 bits binary. I used the recursive interative process method that I learned from OPL class. This method take O(N) time to read 4 elements at time, convert and store it back to single list which take the space complexity is O(N/4).

```racket
;; This function read red/green/blue from PixelsList, then convert it to 24bits binays
;; then return a single list of 24bits integer
(define (RGBmap-iter result lst)
  (if (null? lst)
      result
      (RGBmap-iter
       (cons (join-value (get-r lst) (get-g lst) (get-b lst)) result)
       (remain-lst lst))))

(define RGBmap
  (RGBmap-iter '() PixelsList))
```

## 3. Multiple Local Variables in Function

The method I use to store three values RGB into a 24 bits integer is using bitwise and bitshift. To get understand deeply how I store it, let do an example:

+ Assume that I have three value Red = 190, Green = 173, and blue = 157.
    - Red Value: 190 & xFF (10111110 & 11111111 = 10111110)
    - Green Value: 173 & xFF (10101101 & 11111111 = 10101101). 10101101 >> 8 = 00000000 10101101
    - Blue Value: 157 & FF (10011101 & 11111111 = 10011101). 10011101 >> 16 = 00000000 00000000 10011101

- So now the final binary bits using logic or is: 
    10111110 | 00000000 10101101 | 00000000 00000000 10011101
  
==> 10111110 10101101 10011101 = 12496285

So, this is how I store 3 RGB values into a 24 bits integer.

```racket
;; this function store r/g/b to 24bits memory by using bitwise or/and with shift operators
(define (join-value red green blue)
  (let* ([redvalue (bitwise-and red #xFF)]
         [greenvalue (arithmetic-shift (bitwise-and green #xFF) 8)]
         [bluevalue (arithmetic-shift (bitwise-and blue #xFF) 16)])
    (bitwise-ior redvalue greenvalue bluevalue)))
```

The function ``(join-value red green blue)`` will be call all over other function when it need to convert back to single number. It use ``let*`` to create local multiple variables and then all those variables will be use to return the 24 bits binary number.

The interesting of this is I don't have to store each 3 values RGB separately and by convert to one single number, the length of the list is much more shorter and also it help me easy to troubleshoot the program.

## 4. Map for Gray-Scale to Inverted

The invert color will be use for Gaussian Blur method later on. For some people it sound really hard but this is just simply take each value in the list, extract the red/green/blue by using the bitwise-bit-field, subtract each value from 255 and then recombine back to 24bits integer by using ``join-value`` function. 

I used ``map`` to loop through each value in the list and passing value to ``lambda`` then return an inverted value.

```racket
;; Invert Colors from Gray Scale
(define (get-invert-value num)
  (let*
    [(red (bitwise-bit-field num 0 8))
     (green (bitwise-bit-field num 8 16))
     (blue (bitwise-bit-field num 16 24))]
    (join-value (- 255 red) (- 255 green) (- 255 blue))))

(define inverts-value
  (map (lambda (num) (get-invert-value num)) (reverse gray-scale)))
```


## 5. Tail Recursion Merge Two Images

Last step before convert all pixel back to bitmap%. The color dodge is use to combine the blur and gray-scale together to make a sketch image.

The method use it really simple, from each pixel of blur and gray-scale, extract the RGB value. For 3 RGB value, I use the same method is:

```
if blur_value == 255 return blur_value
else return (gray-scale_value * 256) / (255 - blur_value)
```

and this applied to each red, green, and blue color. Then recombine them by join-value functions as above. At this point, all the values in the new list Color-Dodge-Blend-Merge will be convert to bitmap% and it is sketch image. I use Tail Recursion to make sure the running time is O(N) based on the length of the GrayList.

```racket
(define (colordodge numblur numbw)
  (if (equal? 255 numblur)
      numblur
      (min 255 (round (/ (* numbw 256) (- 255 numblur)))))) 

(define (value-return blist glist)
  (join-value (colordodge (get-r blist) (get-r glist))
              (colordodge (get-g blist) (get-g glist))
              (colordodge (get-b blist) (get-b glist))
              ))

(define (return-dodge num1 num2)
  (local
    [(define rgb1 (extract-rgb num1))
     (define rgb2 (extract-rgb num2))]
    (value-return rgb1 rgb2)))


(define (Color-Dodge-Blend-Merge BlurList GrayList)
  (define (Color-Dodge-Blend-Merge-iter result blurlist bwlist)
    (if (null? bwlist)
        result
        (Color-Dodge-Blend-Merge-iter (cons (return-dodge (car blurlist) (car bwlist)) result)
                                      (cdr blurlist) (cdr bwlist))))
  (Color-Dodge-Blend-Merge-iter '() (reverse BlurList) GrayList))

```

This use to call the Color-Dodge-Blend-Merge function and convert it back to argb which is ready for display final result.

```racket
(define finallist
  (back-to-argb (Color-Dodge-Blend-Merge BlurValue gray-scale)))
```
