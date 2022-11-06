---
title: memset과 bzero의 차이점
date: 2022-10-24 21:20:04 +0900
categories: [Development, C/C++]
tags: [TAG]
---
## 1. bzero

```
#include 
void bzero(void *s, size_t n);
```

-   이름에서도 나타나듯이, zero의 값을 덮어씀
-   0x00의 값을 `s` 영역에 `n` 크기만큼 쓰며, 오직 0x00만의 값만 쓸 수 있다
-   쓰지 않기를 권장하는(deprecated된) 함수. 실제로 `man bzero`를 실행하면 다음 문구를 확인 가능

> 4.3BSD. This function is deprecated (marked as LEGACY in  
> POSIX.1-2001): use memset(3) in new programs. POSIX.1-2008  
> removes the specification of bzero().

## 2. memset

```
#include 
void *memset(void *s, int c, size_t n);
```

-   `c`로 지정된 값을 `s`의 영역에 **_byte 단위로_** `n` 크기만큼 채운다
    -   초기화에 사용될 값을 별도로 지정 가능
-   반환값 : 메모리 영역 s에 대한 시작 포인터

````c++
#include <stdio.h>
#include <string.h>

int main(void) {
    char arr[10]; 
    printf("%p\n", &arr[0]); // 0x7ffc715b8ac0
    printf("%p\n", memset(arr,0x00,sizeof(arr))); // 0x7ffc715b8ac0
    printf("%p\n", &arr[1]); // 0x7ffc715b8ac1
}
````

-   byte 단위를 강조한 이유는?
    -   다음 코드를 실행해볼 것

```c++
#include <stdio.h>
#include <string.h>

int main(void) {
    int arr[10]; 
    memset(arr, 5, sizeof(arr)); 
    printf("%d\n", arr[0]); // 5가 출력될까?
}
```

* 5가 출력되지 않는 이유는?
  * `memset sets each byte of the destination buffer to the specified value. On your system, an int is four bytes, each of which is 5 after the call to memset. Thus, grid\[0\] has the value 0x05050505 (hexadecimal), which is 84215045 in decimal.`
  * memset은 byte 단위로 값을 채우는데, 위 코드에서 출력 대상으로 지정된 memset 대상으로 지정된 arr[0]은 int, 즉 <b>4byte</b>이다
    * 따라서 0x05가 4번 채워지게 되므로, arr[0]의 내부 값은 0x05050505로 초기화된다.  
    * 0x05050505 = 84215045(10)