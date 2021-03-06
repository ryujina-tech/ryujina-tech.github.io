# 쉘 코드

> [해커_지망생이_알아야할_bof_기초-달고나.pdf] Study 정리



# 1. 쉘 코드 만들기

- 쉘 코드란 쉘(shell)을 실행시키는 코드이다. 
- 일종의 유저 인터페이스로, 유저의 키보드 입력을 받아 실행파일을 실행시키거나 커널에 대한 어떠한 명령을 내릴 수 있는 통로이다.
- 바이너리 형태의 기계어 코드이다. (혹은 opcode)

**->쉘 코드를 만드는 이유: 실행중인 프로세스에게 어떠한 동작을 하도록 코드를 넣어 그 실행 흐름을 조작 할 것이기 때문에 실행 가능한 상태의 명령어를 만들어야한다.**



# 2. 쉘 코드 생성

*[C 이용하여 간단한 프로그램 작성 -> 컴파일러가 변환시켜준 어셈블리어 코드 획득 -> 최적화(불필요한 부분 걷어내고 라이브러리에 종속적이지 않도록 수정) -> 바이너리 형태의 데이터 생성]*



## 2.1. 쉘 실행 프로그램

- 쉘 상에서 쉘을 실행시키려면 '/bin/sh' 명령을 내리면 된다.
- 쉘을 실행시키기 위해서 execve()라는 함수를 사용했다. (바이너리 형태의 실행파일이나 스크립트 파일을 실행시키는 함수)

```
#include <stdio.h>

void main()
{
        char *shell[2];

        shell[0] = "/bin/sh";
        shell[1] = NULL;
    
        execve( shell[0], shell, NULL );

}
```

![image-20210606214923447](https://user-images.githubusercontent.com/47252423/120928083-2fcf6780-c71e-11eb-8b82-ae5e4cbc06f1.png)

```
#include <unistd.h>
int execve(const char *pathname, char *const argv[],
           char *const envp[]);

 출처: <https://man7.org/linux/man-pages/man2/execve.2.html> 
 
  Parameter 
     - pathname: 파일 이름
     - argv: 함께 넘겨줄 인자들의 포인터
     - envp: 환경 변수 포인터
```

- execve()함수 때문에 해당 프로그램은 컴파일되면서 Linux libc와 링크된다. execve()의 실제 코드는 libc에 들어있기 때문이다. 때문에 static liblary 옵션으로 컴파일해야 한다.

> **[ Dynamic Link Library & Static Link Library ]**
>
> - ***Dynamic Link Library는 동적 링크 라이브러리***이다. 응용프로그램 실행에 있어서 실제 프로그램 동작에 매우 많은 명령어들이 사용되고, 많은 프로그램들이 공통적으로 사용하는 명령어들이 있다.
>
> - 예를 들어, C언어에서 사용하는 문자열 출력 용도의 printf() 함수는,
>   문자열을 받아서 특정한 위치의 값들을 채운 다음 화면이나 표준 출력, 소리 등의 방법으로 출력할 것이다.
>
>   A프로그램에서 printf()함수를 사용하여 화면에 출력할 것이다. 또한 B프로그램도 printf() 함수를 사용할 것이다. 이때 A도 printf()기능의 기계어 코드를 포함하고있고, B도 printf()기능의 기계어 코드를 포함하고 있다면 ***같은 기능을 하는 기계어 코드가 서로 다른 실행파일에서 모두 포함되어 있게 된다. -> 저장 공간의 낭비!***
>
>   **그래서,** ***운영체제는 이렇게 많이 사용되는 함수의 기계어 코드를 자신이 가지고 있고 다른 프로그램들이 이 기능을 빌려쓰게 해준다.*** -> A, B 모두 pritnf() 기계어 코드를 직접 가지고있지 않고 printf() 코드가 필요할 때 운영체제에게 이 기능을 쓰겠다고 하면 그 코드를 빌려주는 형태이다.     
>
> - 따라서, 응용프로그램 프로그래머는 이 기능을 직접 구현할 필요가 없고 그냥 호출만 해 주면 된다, ***컴파일러도 직접 컴파일 할 필요 없이 호출하는 기계어 코드만 생성해 주면 된다.***
>
>   
>
>   -> 이러한 기능들은 라이브러리라고 하는 형태로 존재하고 있으며, ***Linux에서는 libc라는 라이브러리에 들어있고 실제 파일로는 .so 혹은 .a라는 확장자 형태로 존재한다***. (윈도우즈에는 DDL;Dynamic Link Library 파일로 존재한다.)
>
>   
>
>    - ***운영체제의 버전과 libc의 버전에 따라 호출 형태나 링크가 달라질 수 있기 때문에*** 영향을 받지 않기 위해서 printf() ***기계어 코드를 실행파일이 직접 가지고 있게*** 할 수 있는데 그 방법이 ***Static Link Library***이다. 다만, DLL 보다 실행파일의 크기가 커질것이다. 
>
>   ![image-20210606225231176](https://user-images.githubusercontent.com/47252423/120928093-3958cf80-c71e-11eb-810d-09b4c630d6e2.png)



- Static Link Library 형태로 컴파일 후 기계어 코드 확인

  -> sh.c를 static link library(-static)로 컴파일 하여 sh라는 실행파일을 만든다.

  ```
  [level11@ftz tmp]$ gcc -static -g -o sh sh.c
  ```

  

  -> objdump를 이용해 기계어 코드를 출력한다. 덤프된 코드는 세 개의 column으로 출력된다. 첫 번째 column은 address, 두 번째는 기계어 코드, 세 번째는 기계어 코드에 대응(1:1)하는 어셈블리어 코드이다.

  ```
  [level11@ftz tmp]$ objdump -d sh | grep execve
  ```

  ![image-20210606230917912](https://user-images.githubusercontent.com/47252423/120928102-41187400-c71e-11eb-9027-0a8d6062fa65.png)

  
