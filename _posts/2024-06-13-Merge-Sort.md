---
layout: post
title:  "Merge Sort Java Implementation"
date:   2024-06-13 09:44:32 +0800
categories: jekyll update
tags: Algorithm
---
{% for tag in site.tags %}
    <div style="cursor: pointer; display: inline-block; width: 100px;">
        {{ tag }}
    </div>
{% endfor %}
```java
import java.time.Duration;
import java.time.Instant;

class Solution {
    /**
     * 1. 當前數組長度大於二則切分兩半
     * 2. 直到當前數組長度<=2
     * 3. 若長度為1則直接返回；否則比較兩者大小後返回
     * 4. 對返回的兩個數組做比大小；注意此步驟只需對比頭元素即可(因為已排序)
     * 5. 比較並合併為原本的數組大小
     */
    public int[] mergeSort(int start, int end, int[] arr) {
        int half = (int)((end - start) * 0.5);
        if (end - start < 2) {
            return cmp(start, end, arr);
        }
        return conquer(mergeSort(start, start + half, arr),
            mergeSort(start + half + 1, end, arr));
    }
    
    public int[] cmp(int start, int end, int[] arr) {
        if (end - start == 0) return new int[]{arr[start]};
        if (arr[end] < arr[start]) return new int[]{arr[end], arr[start]};
        return new int[]{arr[start], arr[end]};
    }

    public int[] conquer(int[] a, int[] b){
        int pta = 0, ptb = 0;
        int[] result = new int[a.length + b.length];
        while (pta < a.length && ptb < b.length) {
            if (a[pta] <= b[ptb]) {
                result[pta + ptb] = a[pta];
                pta++;
            }else{
                result[pta + ptb] = b[ptb];
                ptb++;
            }
        }
        if (pta == a.length) {
            for (int i=ptb; ptb<b.length; ptb++) {
                result[a.length+ptb]=b[ptb];
            }
        }else{
            for (int i=pta; pta<a.length; pta++) {
                result[b.length+pta]=a[pta];
            }
        }
        return result;
    }

    public static void main(String[] args) {
        int arr[] = new int[80000];
        for(int i = 0; i < 80000; i++) {
            arr[i] = (int)(Math.random() * 80000);
        }
        
        int temp[] = new int[arr.length];
        Instant startTime = Instant.now();
        mergeSort(arr, 0, arr.length - 1, temp);
        Instant endTime = Instant.now();
        System.out.println("Time Cost：" + Duration.between(startTime, endTime).toMillis() + " ms");
    }
}
```
