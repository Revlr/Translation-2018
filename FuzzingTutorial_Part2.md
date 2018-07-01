# Tutorial - Beginner's Guide to Fuzzing

### Part 2 : Find more Bugs with Address Sanitizer

 [원문](https://fuzzing-project.org/tutorial2.html) 번역 -  6. 27, 2018 willwayy 이정모



퍼징으로 발견할 수 있는 가장 일반적인 버그는 메모리 엑세스 버그이다. 간단히 말하자면 이는 응용 프로그램이 읽지 않거나 쓰지 않아야하는 메모리를 읽거나 쓰는 것을 의미한다. 그러나 모든 메모리 액세스 오류로 인해 crash 가 발생하는 것은 아니다.



C 코드를 예로 들어 설명하겠다, test.c 로 저장하자 :

```c
int main() {
    int a[2] = {1, 0};
    int b=a[2];
}
```

> 이 코드는 클래식한 off-by-one-error 이다.



두 요소가있는 배열 **a**가 정의되고 배열 요소는 0으로 시작하는 인덱스로 액세스되므로 이 배열은 요소 **a**[0]과 **a**[1]로 구성된다. 그리고 **b**에 **a**[2] 값을 저장한다. 그러나 a[2]는 유효하지 않은 값이며 배열의 일부가 아니다. 결국 변수 **b**는 결국 임의의 값을 포함하게된다. 스택에서 "일부" 메모리를 읽는 것만으로 그 메모리에 들어있는 내용을 알 수 없다. 그러나 다른 메모리 액세스 버그와 달리 crash 가 발생하지 않는다. 좋은 툴인 `valgrind`조차도 우리에게 뭔가 잘못되었다고 말해주지 않는다.



최신 버전의 컴파일러 인 `llvm`과 `gcc`는 이러한 메모리 액세스 버그를 찾아내는 강력한 도구를 가지고 있다!

[Address Sanitizer (ASan)](https://code.google.com/p/address-sanitizer/)라고 하며 컴파일 할 때 활성화 할 수 있다.[Address Sanitizer (ASan)](https://code.google.com/p/address-sanitizer/)를 사용하려면                       *-fsanitize = address* 매개 변수를 컴파일러 플래그에 추가해야한다. 디버깅을 쉽게하기 위해 *-ggdb*도 추가해주면 좋다.



```
gcc -fsanitize=address -ggdb -o test test.c
```



이제 작은 예제 프로그램을 실행하면 휘황찬란한 오류 메시지가 나타난다 : )

```bash
==7402==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7fff2971ab88 at pc 0x400904 bp 0x7fff2971ab40 sp 0x7fff2971ab30
READ of size 4 at 0x7fff2971ab88 thread T0
    #0 0x400903 in main /tmp/test.c:3
    #1 0x7fd7e2601f9f in __libc_start_main (/lib64/libc.so.6+0x1ff9f)
    #2 0x400778 (/tmp/a.out+0x400778)

Address 0x7fff2971ab88 is located in stack of thread T0 at offset 40 in frame
    #0 0x400855 in main /tmp/test.c:1

  This frame has 1 object(s):
    [32, 40) 'a' <== Memory access at offset 40 overflows this variable
HINT: this may be a false positive if your program uses some custom stack unwind mechanism or swapcontext
      (longjmp and C++ exceptions *are* supported)
SUMMARY: AddressSanitizer: stack-buffer-overflow /tmp/test.c:3 main
```

> 별로 의미없는 부분을 자르고, 핵심부분만 가져왔다. test.c 의 3행에서 크기가 4 인 정수 (정수의 크기)로 인해 스택 오버플로가 발생한단다! (4.9 이전의 `gcc`)는 읽기 쉬운 출력을 생성하지 못한다.



------

#### Compiling Software with Address Sanitizer

fuzzing 하기를 원하는 소프트웨어는 보통 .c 파일이 아니기 때문에 `address sanitizer` 를 컴파일러 플래그에 추가해야한다. 일반 configure 스크립트를 사용하는 소프트웨어의 경우 다음과 같이 수행 할 수 있다.



```
./configure --disable-shared CFLAGS="-fsanitize=address -ggdb" CXXFLAGS="-fsanitize=address -ggdb"
make
```

> 가능한 경우 공유 라이브러리를 비활성화하고 C 및 C ++ 컴파일러에 대한 플래그를 설정해야한다.



우리는 이제 Part 1과 같이 변형된 입력에 대해 소프트웨어를 실행할 수 있습니다. output을 로그 파일로 리다이렉션 할 때 우리는 더 이상 Segmentation faults 에 대해 grep 할 수 없으므로, 대신 Address Sanitizer 메시지를 grep 해야한다.



```
grep AddressSanitizer fuzzing.log
```



Address Sanitizer 를 사용할 때 고려해야 할 사항이 몇 가지 있다. `ASan`이 Memory Access Violation을 발견하더라도 자동으로 응용 프로그램을 중단하지는 않는다. 자동화 된 fuzzing 툴을 사용하면 일반적으로 return code 를 검사하여 segfault를 감지하려고하기 때문에 문제가된다. 그래서 우리는 다음과 같이 환경 변수 ASAN_OPTIONS에 오류가 발생했을 때 `ASan`이 소프트웨어를 중단하도록 강제 할 수 있다!



```
export ASAN_OPTIONS='abort_on_error=1'
```



하지만 또 다른 문제는 `ASan`이 엄청난 양의 가상 메모리(약 20테라바이트)를 필요로 한다는 것이다 : (  하지만 이것은 가상 메모리이므로 응용 프로그램을 계속 실행할 수 있다 : ) 그러나 `american fuzzy lop`과 같은 fuzzing 도구는 fuzz 소프트웨어의 메모리 양을 제한한다. 그러나 이것도 메모리 제한을 완전히 비활성화하여 해결할 수 있다 (`AFL`의 경우 *-m none*). 그러나 이것은 위험 할 수 있다. fuzzed sample 을 사용하면 응용 프로그램이 방대한 양의 메모리를 할당하게되어 시스템이 불안정 해지고 다른 응용 프로그램이 손상 될 수 있다 : (  따라서 동일한 시스템에서 메모리 제한없이 fuzzer를 실행하는 동안 중요한 작업을해서는 안된다.



지금 [zzuf](http://caca.zoy.org/wiki/zzuf/)는 `ASan`이 작동하지 않는다. upstream 에서는 이를 인식하고 해결 방법을 고려하고 있다. [zzuf](http://caca.zoy.org/wiki/zzuf/)를 사용하여 수동으로 fuzzing 샘플을 생성하고 `ASan` 컴파일 된 소프트웨어에 피드를 제공 할 수 있다.



`ASan`은 실행 속도를 현저히 늦추 게된다는 점도 알아 둬라. `ASan`에서 발견 된 버그는 일반적으로 예외 없이 발견 된 버그보다 덜 크리티컬 하다 (예외는 항상있는 것처럼). 그러나 더 많은 버그를 발견 할 것이다!



 [원문](https://fuzzing-project.org/tutorial2.html) 번역 -  6. 27, 2018 willwayy 이정모