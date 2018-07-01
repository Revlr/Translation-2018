# LiveOverFlow - Episode 0x7

#### Uncrackable License check? 	Part 1/2

7월 1, 2018 willwayy 이정모



일단 먼저 예제파일을 다운받고 시작하자

```
git clone https://github.com/LiveOverflow/liveoverflow_youtube.git
```

> 그 중 0x05_simple_crackme_intro_assembler 이라는 디렉토리 안에 license_1 을 볼 것이다.



<img src="https://user-images.githubusercontent.com/24206298/42133816-eab1fd16-7d6a-11e8-8c01-3f6f72bb8743.png" style="zoom:100%" />

> 64bit 바이너리이고 ELF 파일이다. 
>
> 일단 여러가지 인자들을 주고 실행시켜봤지만 키값은 틀린 것 같다.



<img src="/Users/willwayy/Desktop/File/스크린샷/sc 2.png" style="zoom:0%" />

> 일단 정말 간단하게 시리얼 넘버가 하드코딩 되어있는 예제였다.



보안을 위해서라면 이렇게 중요한 키값이 코드 내부에 있으면 안된다. 그래서 우리는 키를 마구 섞는 알고리즘을 작성하여 키 값을 숨기는 작업을 수행해야 한다. 그런 다음 보안된 바이너리를 친구에게 한번 보여줘 해킹을 요청해 볼 수도 있다.

흔한 방법으로는 키의 ASCII 값을 비교하고 이를 숨겨진 의미의 값과 비교를 하는 방법이 있다. 

<img src="/Users/willwayy/Desktop/File/스크린샷/sc 3.png" style="zoom:50%"/>



먼저 오리지널 키값의 아스키합을 알아내기 위해 아래 코드와 같이 수정을 해준다.

```c
#include <string.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
        if(argc==2) {
		printf("Checking License: %s\n", argv[1]);
		int sum = 0;										; 추가한 부분
		for(int i = 0; i < strlen(argv[1]); i++) {			; 추가한 부분
			sum += (int)argv[1][i];							; 추가한 부분
		}													; 추가한 부분
		printf("Value: %d\n", sum);							; 추가한 부분
		if(strcmp(argv[1], "AAAA-Z10N-42-OK")==0) {
			printf("Access Granted!\n");
		} else {
			printf("WRONG!\n");
		}
	} else {
		printf("Usage: <key>\n");
	}
	return 0;
}
```

> 이렇게 키값에 대한 문자 값의 합계는 **916** 이 나오게 된다.



자 그럼 문자에 대한 키 합계를 알았으니 코드를 재 수정해보자.

```c
#include <string.h>
#include <stdio.h>

int main(int argc, char *argv[]) {
        if(argc==2) {
        printf("Checking License: %s\n", argv[1]);
        int sum = 0;
        for(int i = 0; i < strlen(argv[1]); i++){
            sum += (int)argv[1][i];
        }
        if(sum == 916) {									; 이전에는 키값이 하드코딩되어 있었다.
            printf("Access Granted!\n");
        } else {
            printf("WRONG!\n");
        }
    } else {
        printf("Usage: <key>\n");
    }
    return 0;
}
```



올바른 키값과 옳지 않은 키값을 집어넣어보자

<img src="/Users/willwayy/Desktop/File/스크린샷/sc 5.png" style="zoom:0%" />

> 잘 작동한다



자 그럼 이걸 어떻게 Crack 할 수 있을까? radare2 를 이용해 분석해보자

<img src="/Users/willwayy/Desktop/File/스크린샷/sc 6.png" style="zoom:0%" />

> 바이너리를 올리고 분석 후 메인 함수 표시



<img src="/Users/willwayy/Desktop/File/스크린샷/sc 7.png" style="zoom:00%" />

> 비교 구문을 하는 곳이 저 파트인 것 같으니 저 부분을 우회해주면 될 것 같다.



radare2 를 통해 rip를 바꿔주는 것은 상당히 쉽다.

1. 먼저 아무 키값이나 넣어준다.
2. 분기점인 0x00400642 에 breakpoint 를 걸어주고 실행한다.
3. 옳은 키값이 아니므로 다음 rip 에는 0x00400650 이 들어가 있겠지만 성공적인 분기점인 0x00400644 로 바꿔주면 된다.

```tex
[0x7fb9fb44ec30]> db 0x00400642							; db: 브레이크 포인트
[0x7f9714e4f748]> ood AAA-WRONG-KEY						; ood: 디버깅 모드 
[0x7f78fde4ac30]> dc									; 실행
Checking License: AAA-WRONG-KEY
hit breakpoint at: 400642
[0x00400642]> dr										; 현재 레지스터 상태보기
rsp = 0x7ffd1eb54210
rbp = 0x7ffd1eb54240
rip = 0x00400642
[0x00400642]> dr rip=0x00400644							; <register> 변경
0x00400642 ->0x00400644
[0x00400642]> dr
rsp = 0x7ffd1eb54210
rbp = 0x7ffd1eb54240
rip = 0x00400644
[0x00400642]> dc
Access Granted!
```

> 단순히 성공적인 메세지를 뿌리는 곳으로 바꿔줬을 뿐이다. (매우 간단한 방법)



하지만 이런 점프문 패치는 지루하고 별로 의미가 없다. 그렇다면 keygen을 만들어보는건 어떨까? 한번 해보자.

<img src="/Users/willwayy/Desktop/File/스크린샷/sc 9.png" style="zoom:100%" />

> **1:**
>
> - rbx 에는 루프 조건 i 가 저장되고 rax 에는 문자의 개수를 저장한다. (AAA-WRONG-KEY : rax ➔ D (13))
>
> **2:**
>
> - [rax] 에서 키값을 넣은 문자들을 하나씩 16진수 형태로 가져온다. 그리고 eax 에 (int)형으로 저장하고 i 값 1 증가



<img src="/Users/willwayy/Desktop/File/스크린샷/sc 14.png" style="zoom:0%" />

> **MOVZX**
>
> \- 부호없는 산술 값에 대해서 사용됨
>
> \- 바이트나 워드를 워드나 더블워드 목적지에 전송
>
> \- 0비트로 목적지 피연산자의 왼쪽 비트들을 채운다.
>
> ```assembly
> MOV bx,0A69Bh
> 
> MOVZX EAX, BX                  ; EAX = 0000A69Bh         //16비트 BX의 값이 이동
> 
> MOVZX EDX, BL                  ; EDX = 0000009Bh         //8비트 BL(BX의 하위 8비트)값이 이동
> 
> MOVZX CX, BL                   ; CX = 009Bh              //8비트 BL값이 이동
> ```
>
> **MOVSX**
>
> \- 부호있는 산술 값에 대해서 사용
>
> \- 바이트나 워드를 워드나 더블워드의 목적지에 전송
>
> \- 부호비트(원시 피연산자의 맨 왼쪽비트)로 목적지 피연산자의 왼쪽 비트들을 채운다.
>
> ```asm
> MOV BX, 0A69Bh
> 
> MOVSX EAX, BX                  ; EAX = FFFFA69Bh
> 
> MOVSX EDX, BL                  ; EDX = FFFFFF9Bh
> 
> MOVSX CX, BL                   ; CX = FF9Bh
> ```
>
> 



보기 쉽게 예를 들어 파라미터를 ABCD 로 주고 rax 에 어떤식으로 저장되는지 보면,

<img src="/Users/willwayy/Desktop/File/스크린샷/sc 15.png" style="zoom:00%" />

> 이렇게 레지스터에 저장되는것을 볼 수 있다.



이런식의 알고리즘으로 진행되고 마지막으로 `sum == 916` 이 맞는지를 검사하는 절차이니깐 꼭 키값이 원래의 키값 1개만 있을 수 있는것이 아니라 916이 될 수 있는 경우의 수는 상당히 많다! 이러한 경우를 생각해보며 파이썬 키젠 코드를 작성해보자



```python
import sys
import random

def check_key(key):
	char_sum = 0;
	for c in key:
		char_sum += ord(c)
	sys.stdout.write("{0:3} | {1}		\r".format(char_sum, key))
	sys.stdout.flush()
	return char_sum

key = ""
while True:
	key += random.choice("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890_")
	s = check_key(key)
	if s > 916:
		key = ""
	elif s == 916:
		print "Found valid key: {0}".format(key)
```

> 





<img src="/Users/willwayy/Desktop/File/스크린샷/sc 11.png" style="zoom:0%" />

> 완★성



7월 1, 2018 willwayy 이정모
