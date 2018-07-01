# Tutorial - Beginner's Guide to Fuzzing

### Part 3 : Instrumented fuzzing with american fuzzy lop

 [원문](https://fuzzing-project.org/tutorial3.html) 번역 -  6. 27, 2018 willwayy 이정모



[zzuf](http://caca.zoy.org/wiki/zzuf/) 와 같은 단순한 fuzzer로 fuzzing을 하면 버그를 쉽게 찾을 수 있지만 훨씬 더 좋은 방법의 fuzzing 전략이 있다. 하나는 사용 된 파일 형식을 인식하는 fuzzer를 작성하는 것입니다. 예를 들어 파일 헤더의 모든 필드를 최대 값 또는 최소값으로 설정하려고 할 수 있습니다. 그러나 형식을 인식하는 fuzzing은 성가신 일입니다. 왜냐하면 당신이 퍼지하고있는 모든 입력 형식에 대해 fuzzer가 필요하기 때문이다. 예를 들어 파일 헤더의 모든 필드를 최대 값 또는 최소값으로 설정하려고 할 수 있습니다. 그러나 모든 입력 형식에 대해 Fuzzer가 필요하기 때문에 포맷 인식은 번거로운 것 같다.



Compile-time instrumented fuzzing 은 또 다른 경로로 진행되는데, fuzzer가 응용 프로그램의 코드 경로를 감지 할 수 있도록 응용 프로그램의 코드에 경로를 추가한다. 그런 다음 더 많은 fuzzing을 위해 코드의 많은 부분을 노출시키는 좋은 fuzzing 샘플을 사용할 수 있다.



[american fuzzy lop (afl)](http://lcamtuf.coredump.cx/afl/)은 계측 된 fuzzing을 수행하며 현재로서는 사용할 수있는 최상의 fuzzing tool 일 것이다. 물론 좋은  다른 퍼저들도 많다. 어쨋든 `AFL`을 사용하면 상당히 [놀라운결과](https://lcamtuf.blogspot.com/2014/11/pulling-jpegs-out-of-thin-air.html)가 발생할 수 있습니다.



autoconf 기반 구성 스크립트를 사용하는 도구가 있다고 가정해 봅시다. `AFL`로 코드 컴파일 :

```
./configure CC="afl-gcc" CXX="afl-g++" --disable-shared; make
```

> 다시 --disable-shared를 사용하여 가능한 경우 정적으로 컴파일하고 LD_PRELOAD 호출을 피한다.



이제 하나 또는 여러 개의 입력 샘플이 필요하다. 샘플의 크기는 작아야하며, 우리는 우리가 호출 할 디렉토리에 그것들을 배치해야한다. 이제 우리는 afl-fuzz를 시작해보자 :

```
afl-fuzz -i in -o out [path_to_tool] @@
```

> @@ 은 fuzzed input 파일로 대체된다. 이것을 건너 뛰면 표준 입력에 퍼지 파일이 전달된다.



이 작업을 수행할 때 Linux커널에 의한 자동 frequency scaling 이 제대로 작동하지 않기 때문에 일반적으로 CPU2REQ설정을 성능으로 변경해야 해서 불편하다.



root 로 이 명령을 실행하면된다 :

```
echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

또는 CPUFREQ 설정을 무시하도록 afl에 지시 할 수도 있다.

```
AFL_SKIP_CPUFREQ=1 afl-fuzz -i in -o out [path_to_tool] @@
```



터미널 크기가 너무 작다는 경고를받을 수도 있다.이 경우 터미널 창을 조금 더 크게 만들면된다. 이 과정들을 마치면  다음과 같은 인터페이스가 표시된다.

<img src="https://fuzzing-project.org/afl-screenshot.png" style="zoom:90%" />

> 모든 것이 정확하면 언제나 새로운 경로를 보아야한다. 총 경로가 1에 머물러 있다면 아마도 잘못된 설정했을 것이다. 가장 흥미로운 점은 uniq crashes 이다. 거기에서 segfault가 발견되면 대부분은 메모리 액세스 오류일 것이다.



`american fuzzy lop` 은 많은 파일 쓰기 (초당 수백 또는 수천 개)를 수행한다. 이 블로그 게시물은 RAM 디스크를 사용하여 SSD에 너무 많은 스트레스를주지 않도록 제안했다. 아마 여러분들도 이렇게 하면 더 좋을 것이다.



------

#### american fuzzy lop and Address Sanitizer

우리는 Part 2에서 `Address Sanitizer`가 버그를 찾는 능력을 크게 향상시킬 수 있다는 것을 배웠고 이 장에서는 `AFL`이 뛰어난 fuzzers 중 하나임을 알았다. 하지만 우리가하고 싶은 것은 이 둘의 힘을 합치는 것이다.



하지만 불행하게도 여러 가지 어려움이 있다. `Address Sanitizer`는 많은 가상 메모리가 필요하며 기본적으로 `AFL`은 Fuzz 된 소프트웨어가 가져 오는 메모리 양을 제한 한다. 이 문제를 해결하는 가장 쉬운 방법은 메모리 제한 (*-m none*)을 비활성화하는 것이지만, 전장이도 말했듯이 Fuzz sample 로 인해 많은 메모리를 할당하고 사용함으로 인해 시스템에서 임의의 충돌이 발생할 수 있다. 이러한 작업 도중 중요한 일을해서는 안된다. 언제 주옥될지 모르기 때문이다.



`Address Sanitizer`를 사용하려면 컴파일하는 동안 환경 변수 AFL_USE_ASAN을 1로 설정해야한다.

```
AFL_USE_ASAN=1 ./configure CC=afl-gcc CXX=afl-g++ LD=afl-gcc--disable-shared
AFL_USE_ASAN=1 make
```



`Address Sanitizer`와 함께 `AFL`을 사용하면 fuzzing 작업 속도가 상당히 느려지므로 주의해야한다.

하지만 여전히 버그가 더 많이 발견 될 가능성이 높다...



자 튜토리얼은 대략 이게 전부다! 좀 더 `AFL` 에 자세히 알고싶으면 [american fuzzy lop documentation](http://lcamtuf.coredump.cx/afl/README.txt)를 참고하고, 이제 스스로 퍼징을 해봐!! 굿럭!



 [원문](https://fuzzing-project.org/tutorial3.html) 번역 -  6. 27, 2018 willwayy 이정모