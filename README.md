# Jaly.js Runtime

**단일 실행파일 `jaly.exe` 하나로 동작하는 독립 JavaScript Runtime.**
외부 DLL, Node.js, npm, Python, 별도 런타임 — 전부 필요 없음.

```
$ objdump -p jaly.exe | grep "DLL Name"
    KERNEL32.dll
    msvcrt.dll
```

참조하는 DLL은 모든 Windows에 기본 내장된 시스템 DLL 두 개뿐. JS 엔진(QuickJS-ng)까지
`-static` 정적 링크로 exe 안에 전부 포함되어 있어서, **`jaly.exe` 파일 하나만 복사해서
어떤 Windows 컴퓨터에서든 바로 실행 가능**.

## 설계 철학

Jaly는 Sandbox가 아니다.

```
JavaScript
    ↓
Jaly Runtime
    ↓
Native / OS
    ↓
사용자의 OS 권한
```

Jaly 자체는 별도의 권한 시스템이나 보안 레이어를 두지 않는다. 사용자가 Jaly를
실행하면, Jaly는 그 프로세스가 OS로부터 부여받은 권한을 그대로 사용한다 —
관리자 권한으로 실행하면 관리자 권한 범위에서 동작한다. Memory API는 raw pointer를
BigInt로 그대로 노출하고, bounds check나 handle 간접 계층을 두지 않는다. 잘못된
주소에 접근하면 C로 짠 코드와 동일하게 그냥 fault가 난다 — 이것이 의도된 trade-off.

브라우저에서 Jaly를 쓸 경우 보안 경계는 Jaly가 아니라 **브라우저**(확장 프로그램 +
Native Messaging)가 담당한다. `jaly.exe`를 직접 실행하는 사용자에게는 Jaly 자체가
아무것도 막지 않는다.

## 실행

```
jaly.exe app.js          # 스크립트 실행
jaly.exe -e "1+1"        # 인라인 코드 평가
jaly.exe --selftest      # 내장 자체 테스트
jaly.exe --version
```

### 자체 테스트 결과 (Wine으로 실제 실행 검증 완료)

```
Hello from Jaly
raw pointer = 140737464187664
read back   = 1234
Jaly.file.exists('.') = true
platform = win32 version = 0.1.0
Jaly.asm.ADD(2,3) = 5
GetTickCount() = 288331
```

`GetTickCount()` 호출은 `kernel32.dll`을 실제로 로드하고 `GetProcAddress`로 심볼을
찾아 `Jaly.native.call`로 직접 호출한 결과 — 실제 Win32 API를 JS에서 바로 호출.

## API

### Jaly.memory — raw pointer 기반, 제한 없음

```js
Jaly.memory.alloc(size) -> ptr (BigInt)
Jaly.memory.calloc(count, elemSize) -> ptr
Jaly.memory.realloc(ptr, size) -> ptr
Jaly.memory.free(ptr)

Jaly.memory.add(ptr, offset) -> ptr        // pointer arithmetic
Jaly.memory.isNull(ptr) -> bool

Jaly.memory.read8/16/32/64(ptr, offset?)
Jaly.memory.readPtr(ptr, offset?)
Jaly.memory.readF32/readF64(ptr, offset?)
Jaly.memory.readString(ptr, offset?, maxLen?) -> string (NUL-terminated)

Jaly.memory.write8/16/32/64(ptr, [offset,] value)
Jaly.memory.writePtr(ptr, [offset,] value)
Jaly.memory.writeF32/writeF64(ptr, [offset,] value)
Jaly.memory.writeString(ptr, str, offset?) -> bytesWritten

Jaly.memory.copy(destPtr, srcPtr, length)
Jaly.memory.fill(ptr, value, length)
Jaly.memory.compare(ptrA, ptrB, length) -> -1 | 0 | 1

Jaly.memory.view(ptr, length) -> ArrayBuffer   // zero-copy, wraps real memory directly
Jaly.memory.bufferAddress(arrayBuffer) -> ptr  // get raw address of a JS buffer
```

포인터는 전부 **진짜 native address**이며 JS `BigInt`로 주고받는다 (Number는 53비트
정밀도라 64비트 주소를 안전하게 담을 수 없음). handle table이나 bounds check 없음 —
`Jaly.memory.alloc`이 반환한 주소든, 다른 native 함수가 돌려준 주소든 동일하게 다룰 수 있다.

### Jaly.native — 실제 native 함수 직접 호출 (FFI)

```js
let lib = Jaly.native.loadLibrary("user32.dll")      // -> module handle (BigInt)
let fn  = Jaly.native.getProcAddress(lib, "MessageBoxA")  // -> function ptr (BigInt)

Jaly.native.call(fn, [0, "Hello", "Jaly", 0], "i32")  // 실제 MessageBoxA 호출

Jaly.native.freeLibrary(lib)
Jaly.native.lastError() -> number   // GetLastError() / errno
```

`call(fnPtr, args, returnType)`:
- `args`: 배열. 각 원소는 number / string / bigint / boolean을 자동 타입 추론하거나,
  `{ type: 'string'|'i64'|'ptr'|..., value }` 형태로 명시 가능. 최대 12개.
- `returnType`: `'void' | 'i32' | 'u32' | 'i64' | 'u64' | 'ptr' | 'double' | 'string' | 'wstring'`
  (기본값 `'i64'`)

**알려진 제약 (숨기지 않고 명시):** Windows x64 호출 규약에서 정수/포인터 인자는
RCX/RDX/R8/R9 + 스택으로 전달되므로 이 방식으로 정확히 동작하지만, `double` 타입
**인자**는 XMM 레지스터를 쓰기 때문에 현재 dispatcher는 지원하지 않는다 (double
**반환값**은 지원됨). 대부분의 Win32 API — HANDLE/DWORD/포인터 인자 — 는 문제없이
호출 가능. float 인자가 필요한 시그니처는 이후 전용 shim으로 추가 예정.

### Jaly.asm — 어셈블리 스타일 추상 명령어

실제 CPU 명령을 실행하는 게 아니라, Jaly의 메모리 연산을 어셈블리풍 이름으로 감싼
추상화 레이어:

```js
Jaly.asm.MOV(destPtr, srcPtr, size)
Jaly.asm.LOAD(ptr, size) -> value      // size: 1|2|4|8
Jaly.asm.STORE(ptr, size, value)
Jaly.asm.ADD(a, b) / SUB / MUL / DIV
Jaly.asm.CMP(a, b) -> -1 | 0 | 1
```

### Jaly.binary

```js
Jaly.binary.toMemory(arrayBufferOrView) -> ptr   // malloc + copy
Jaly.binary.fromMemory(ptr, length) -> ArrayBuffer (copy)
Jaly.binary.hex(arrayBuffer) -> string
```

### Jaly.file / Jaly.process / Jaly.system / Jaly.console

```js
Jaly.file.read(path) -> ArrayBuffer
Jaly.file.readText(path) -> string
Jaly.file.write(path, data)          // string | ArrayBuffer | TypedArray
Jaly.file.exists(path) -> bool
Jaly.file.delete(path) -> bool
Jaly.file.list(dir) -> string[]
Jaly.file.isDirectory(path) -> bool
Jaly.file.mkdir(path) -> bool

Jaly.process.run(cmd) -> { exitCode, stdout, stderr }
Jaly.process.list() -> [{ pid, name }]
Jaly.process.info(pid) -> { pid, name, memoryBytes } | null

Jaly.system.cpuCount() -> number
Jaly.system.memoryInfo() -> { totalBytes, freeBytes, percentUsed }
Jaly.system.osInfo() -> { platform, version }

Jaly.console.log/info/warn/error(...)   // 전역 console도 동일하게 동작
Jaly.version / Jaly.platform
```

## 예제: MessageBoxA 직접 호출

```js
let user32 = Jaly.native.loadLibrary("user32.dll");
let MessageBoxA = Jaly.native.getProcAddress(user32, "MessageBoxA");
Jaly.native.call(MessageBoxA, [0, "Jaly에서 직접 호출했습니다", "Jaly.native", 0], "i32");
```

## 예제: 원시 메모리로 구조체 흉내내기

```js
// struct { int32 x; int32 y; } 를 흉내
let point = Jaly.memory.alloc(8);
Jaly.memory.write32(point, 10);           // x
Jaly.memory.write32(Jaly.memory.add(point, 4), 20); // y

console.log(Jaly.memory.read32(point));                       // 10
console.log(Jaly.memory.read32(Jaly.memory.add(point, 4)));   // 20

Jaly.memory.free(point);
```

## 소스에서 다시 빌드하기

```bash
# 1) mingw-w64 크로스컴파일러 + cmake 설치 (Linux에서 Windows용 빌드 시)
sudo apt install g++-mingw-w64-x86-64 gcc-mingw-w64-x86-64 cmake

# 2) QuickJS-ng 소스 받기
git clone https://github.com/quickjs-ng/quickjs.git third_party/quickjs

# 3) 빌드
mkdir build-win && cd build-win
cmake -DCMAKE_TOOLCHAIN_FILE=../mingw-toolchain.cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build . -j4
# -> build-win/jaly.exe
```

Windows 네이티브 빌드 시 동일한 `CMakeLists.txt`를 MSVC나 MinGW로 그대로 사용 가능
(MSVC는 `-static` 링크 플래그 대신 `/MT` 런타임 라이브러리 옵션 사용).

## 프로젝트 구조

```
jaly/
├── CMakeLists.txt
├── mingw-toolchain.cmake
├── README.md
├── third_party/quickjs/     (별도로 clone, 이 tar에는 미포함 — 위 빌드 방법 참고)
└── src/
    ├── main.cpp             엔트리포인트
    ├── jaly_memory.{h,cpp}  raw pointer 기반 메모리 API
    ├── jaly_native.{h,cpp}  FFI (loadLibrary/getProcAddress/call)
    ├── jaly_asm.{h,cpp}     어셈블리 스타일 추상 명령어
    ├── jaly_binary.{h,cpp}  ArrayBuffer ↔ native memory 브릿지
    ├── jaly_file.{h,cpp}    파일 I/O
    ├── jaly_process.{h,cpp} 프로세스 실행/조회
    ├── jaly_system.{h,cpp}  CPU/RAM/OS 정보
    └── jaly_console.{h,cpp} console.log 등
```

## 다음 단계

- **브라우저 브릿지**: Native Messaging 기반 확장 프로그램 (Jaly.exe를 직접 실행한다고
  가정하지 않고, 브라우저가 보안 경계와 사용자 승인을 담당하는 명시적 브릿지)
- **Jaly.network**: 소켓/HTTP 클라이언트
- **float 인자를 포함하는 native 호출**: 시그니처별 전용 shim 또는 libffi 연동
