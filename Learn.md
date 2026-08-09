# Jaly.js 배워보기

---

## 1. 왜 만들어졌는가

브라우저의 JavaScript는 태생적으로 **메모리를 직접 다룰 수 없다.**

`ArrayBuffer`, `TypedArray`가 있긴 하지만, 이것들은 브라우저 엔진(V8, SpiderMonkey 등)이
관리하는 가상의 메모리 영역일 뿐이다. 진짜 메모리 주소를 손에 쥐고, 원하는 위치에
직접 값을 쓰고 읽고, 포인터 연산을 하고, OS의 native 함수를 바로 호출하는 것 —
이런 건 JavaScript 스펙 자체가 애초에 허용하지 않는다. C/C++이나 Rust로 가야만
가능했던 영역이다.

**Jaly.js는 이 벽을 없애기 위해 만들어졌다.**

```
JavaScript의 편리함
        +
C/C++ 수준의 저수준 메모리 제어
        =
Jaly.js
```

Jaly는 QuickJS라는 JavaScript 엔진을 그대로 품은 채로, 그 위에 `Jaly.memory`,
`Jaly.native`, `Jaly.asm` 같은 API를 얹어서 **진짜 native pointer**를 JS 코드에서
자유롭게 주무를 수 있게 한다. `Jaly.memory.alloc()`이 돌려주는 값은 흉내낸 핸들이
아니라 실제 메모리 주소이고, `Jaly.native.call()`은 실제 Win32 API를 그대로 호출한다.

### 최종 목표: 브라우저 안에서도 메모리 조작

Jaly.js를 만든 진짜 이유는 여기서 끝나지 않는다. 지금의 `jaly.exe`는 데스크톱에서
스크립트를 실행하는 독립 런타임이지만, 이 프로젝트의 지향점은 **일반 웹사이트에서
동작하는 브라우저 JavaScript도 Jaly의 저수준 메모리 기능을 쓸 수 있게 만드는 것**이다.

```
Chrome / Edge 같은 브라우저
        ↓
   웹사이트의 JavaScript
        ↓
     Jaly Bridge   (브라우저 확장 + Native Messaging)
        ↓
      jaly.exe
        ↓
     Native OS
```

즉 목표는:

- 웹페이지가 `Jaly.memory.alloc()`, `Jaly.memory.write32()` 같은 API를 그대로 호출하고
- 그 호출이 브라우저 확장 프로그램을 거쳐 로컬에 설치된 `jaly.exe`로 전달되고
- `jaly.exe`가 실제 OS 메모리 연산을 수행한 뒤 결과를 다시 웹페이지로 돌려주는 것.

지금까지 만든 `jaly.exe`의 `Jaly.memory` / `Jaly.native` / `Jaly.asm` API는 전부 이
목표를 위한 **기반 엔진**이다. 데스크톱에서 먼저 이 API들이 진짜로 동작하는지
검증하고, 그 다음 단계로 브라우저 브릿지를 통해 웹에서도 같은 API를 노출하는 순서로
프로젝트가 진행되고 있다.

---

## 2. 코드 배우기

### 2.1 Jaly는 어떻게 시작하는가

```
jaly.exe app.js
```

이 한 줄로 `app.js`가 실행된다. Node.js처럼 특별한 설치 과정이나 `npm install`이
필요 없다 — `jaly.exe` 파일 하나가 JS 엔진과 모든 API를 이미 담고 있다.

### 2.2 메모리는 "주소"다

일반 JavaScript에서 변수는 값이 어디 있는지 신경 쓸 필요가 없다. 하지만 Jaly에서
`Jaly.memory.alloc()`을 부르면, 진짜 메모리 주소가 돌아온다.

```js
let ptr = Jaly.memory.alloc(16); // 16바이트짜리 메모리를 새로 할당
console.log(ptr); // 예: 140737464187664n  (BigInt로 표현되는 진짜 주소)
```

`BigInt`로 표현되는 이유: 64비트 메모리 주소는 JS의 일반 `Number`(53비트까지만
정확함)로는 정밀하게 담을 수 없기 때문이다. 그래서 Jaly의 모든 포인터는 `n`이
붙는 BigInt다.

이 메모리는 다 쓰면 반드시 직접 반납해야 한다 (C의 `malloc`/`free`와 완전히 동일한
개념):

```js
Jaly.memory.free(ptr);
```

### 2.3 그 주소에 값을 쓰고, 읽기

```js
Jaly.memory.write32(ptr, 1234);   // ptr이 가리키는 위치에 4바이트 정수 1234를 씀
let value = Jaly.memory.read32(ptr); // 다시 읽음
console.log(value); // 1234
```

`8/16/32/64`는 몇 비트짜리 정수를 다룰지를 뜻한다. 부동소수점은 `readF32/writeF32`,
`readF64/writeF64`를 쓴다.

### 2.4 포인터 연산 (pointer arithmetic)

C에서 `ptr + 4`를 하듯이, Jaly에서도 주소에 오프셋을 더해 새로운 주소를 만들 수 있다.

```js
let base = Jaly.memory.alloc(8);
Jaly.memory.write32(base, 10);                       // base+0 에 10 저장
Jaly.memory.write32(Jaly.memory.add(base, 4), 20);   // base+4 에 20 저장

console.log(Jaly.memory.read32(base));                     // 10
console.log(Jaly.memory.read32(Jaly.memory.add(base, 4))); // 20
```

이건 사실상 `struct { int32 x; int32 y; }`를 흉내 낸 것이다 — JS에는 원래 없는
개념이지만, 메모리 주소를 직접 다룰 수 있으니 가능해진다.

### 2.5 Native 함수를 직접 부르기 (FFI)

Jaly의 가장 강력한 기능. Windows의 DLL을 열어서, 그 안의 함수를 JS에서 바로 호출한다.

```js
let user32 = Jaly.native.loadLibrary("user32.dll");
let MessageBoxA = Jaly.native.getProcAddress(user32, "MessageBoxA");

Jaly.native.call(MessageBoxA, [0, "Hello", "Jaly", 0], "i32");
```

이 세 줄이 실제로 Windows 메시지박스를 띄운다. `Jaly.native.call`의 세 인자는
`(함수 포인터, 인자 배열, 반환 타입)`이다. 인자는 숫자/문자열/BigInt를 자동으로
적절한 native 타입으로 바꿔서 넘겨준다.

### 2.6 Jaly.asm — 어셈블리 감성의 명령어

`Jaly.memory`를 어셈블리 스타일 이름으로 감싼 것뿐이지만, "메모리를 명령어처럼
다룬다"는 감각을 준다.

```js
Jaly.asm.STORE(ptr, 4, 99);     // ptr 위치에 4바이트로 99 저장 (write32와 동일)
console.log(Jaly.asm.LOAD(ptr, 4)); // 99
console.log(Jaly.asm.ADD(2, 3));    // 5
```

> 주의: 이건 진짜 CPU 명령어를 실행하는 게 아니라, Jaly의 메모리 연산에 어셈블리
> 이름을 붙인 것이다. 진짜 native 실행은 `Jaly.native.call`이 담당한다.

---

## 3. 코드예제

### 예제 1 — 메모리에 직접 문자열 쓰고 읽기

```js
let ptr = Jaly.memory.alloc(64);

Jaly.memory.writeString(ptr, "Hello Jaly");
console.log(Jaly.memory.readString(ptr)); // "Hello Jaly"

Jaly.memory.free(ptr);
```

### 예제 2 — 두 메모리 블록끼리 복사하기

```js
let src = Jaly.memory.alloc(16);
let dst = Jaly.memory.alloc(16);

Jaly.memory.writeString(src, "copy me");
Jaly.memory.copy(dst, src, 16); // src의 16바이트를 dst로 그대로 복사

console.log(Jaly.memory.readString(dst)); // "copy me"

Jaly.memory.free(src);
Jaly.memory.free(dst);
```

### 예제 3 — 실제 Win32 API 호출 (현재 시간 틱 값 가져오기)

```js
let kernel32 = Jaly.native.loadLibrary("kernel32.dll");
let GetTickCount = Jaly.native.getProcAddress(kernel32, "GetTickCount");

let ticks = Jaly.native.call(GetTickCount, [], "i64");
console.log("부팅 후 경과 시간(ms):", ticks);
```

### 예제 4 — 파일을 통째로 메모리로 읽어서 조작하기

```js
let data = Jaly.file.read("input.bin");       // ArrayBuffer로 읽기
let ptr = Jaly.binary.toMemory(data);         // native 메모리로 복사

Jaly.memory.write8(ptr, 0xFF);                // 첫 바이트를 0xFF로 변경

let result = Jaly.binary.fromMemory(ptr, data.byteLength);
Jaly.file.write("output.bin", result);

Jaly.memory.free(ptr);
```

### 예제 5 — struct 흉내내기 + ADD로 값 계산

```js
let point = Jaly.memory.alloc(8); // int32 x, int32 y

Jaly.memory.write32(point, 3);
Jaly.memory.write32(Jaly.memory.add(point, 4), 4);

let x = Jaly.memory.read32(point);
let y = Jaly.memory.read32(Jaly.memory.add(point, 4));

console.log("x + y =", Jaly.asm.ADD(x, y)); // 7

Jaly.memory.free(point);
```

---

## 4. 퀴즈

아래 질문에 스스로 답해본 뒤, 맨 아래 정답을 확인해보자.

**Q1.** `Jaly.memory.alloc(16)`이 돌려주는 값의 타입은 무엇이며, 왜 일반 `Number`가
아니라 그 타입을 쓰는가?

**Q2.** 다음 코드를 실행하면 무엇이 출력될까?

```js
let p = Jaly.memory.alloc(8);
Jaly.memory.write32(p, 100);
Jaly.memory.write32(Jaly.memory.add(p, 4), 200);
console.log(Jaly.memory.read32(Jaly.memory.add(p, 4)));
```

**Q3.** `Jaly.memory.alloc()`으로 할당한 메모리를 다 쓴 뒤 `Jaly.memory.free()`를
호출하지 않으면 어떤 일이 벌어지는가?

**Q4.** `Jaly.native.call(fn, args, returnType)`에서 `returnType`을 `"string"`으로
지정하면 Jaly는 반환된 값을 어떻게 해석하는가?

**Q5.** Jaly.js의 최종 목표 중 하나는 "브라우저에서도 메모리 조작이 가능하게
하는 것"이다. 이걸 실현하기 위해 구상된 구조에서, **보안 경계(security boundary)를
책임지는 주체는 무엇인가?**

**Q6.** `Jaly.asm.STORE(ptr, 4, 99)`와 `Jaly.memory.write32(ptr, 99)`는 결과가
같다. 그렇다면 `Jaly.asm` 모듈이 존재하는 이유는 무엇일까?

---

### 정답

**A1.** `BigInt` (예: `140737464187664n`). 메모리 주소는 64비트 값인데, JS의
`Number`는 정수를 최대 53비트까지만 오차 없이 표현할 수 있어서, 그보다 큰 주소값을
쓰면 값이 미묘하게 틀어질 수 있다. 그래서 Jaly는 포인터를 항상 BigInt로 다룬다.

**A2.** `200`. `p`에 4바이트짜리 `100`을 쓰고, `p+4` 위치에 `200`을 쓴 다음,
`p+4`를 다시 읽었기 때문에 `200`이 나온다.

**A3.** 메모리 누수(leak)가 발생한다. Jaly는 handle 테이블이나 가비지 컬렉터가
따로 관리해주지 않는 raw pointer 모델이기 때문에, `free()`를 안 하면 그 메모리는
프로그램이 끝날 때까지(또는 OS가 회수할 때까지) 계속 점유된 채로 남는다 — C의
`malloc`/`free`와 완전히 같은 책임 구조다.

**A4.** 반환된 값을 **native 포인터로 취급하고, 그 주소부터 NUL(`\0`)이 나올 때까지
바이트를 읽어서 문자열로 변환**한다. 즉 호출한 native 함수가 실제로 유효한
문자열 포인터를 반환했다는 전제 하에 동작한다 — 아니라면 잘못된 주소를 읽게 된다.

**A5.** **브라우저.** Jaly.exe 자체는 권한 시스템을 두지 않지만, 웹사이트가
Jaly의 기능을 쓰려면 반드시 브라우저 확장 프로그램 + Native Messaging을 거쳐야
하고, 그 경계에서 사용자 승인과 보안 판단을 브라우저가 담당한다. `jaly.exe`를
사용자가 직접 실행할 때는 이런 경계가 없다 — 그건 이미 사용자가 자기 컴퓨터에서
실행하기로 결정했다는 뜻이기 때문이다.

**A6.** 기능적으로는 다르지 않다. `Jaly.asm`은 순전히 **표현 방식**의 차이다 —
"메모리를 어셈블리 명령어처럼 다룬다"는 사고방식에 익숙한 사람에게 더 직관적인
이름(`MOV`, `LOAD`, `STORE`)을 붙여준 별칭 계층일 뿐, 내부적으로는 `Jaly.memory`와
동일한 연산을 수행한다.
