# assembly/hash 폴더와 Contenthash, WebAssembly의 역할

> Webpack의 `assembly/hash` 폴더 역할을 중심으로,  
Contenthash와 WebAssembly(WASM)가 어떻게 연계되어 동작하는지 정리합니다.

<br/>

## 폴더 구조

- `assembly/hash` 폴더는 Webpack에서 **고성능 해시 알고리즘(MD4, xxHash64 등)**을  
WebAssembly(AssemblyScript 기반)로 구현한 소스가 위치하는 곳입니다.
- JS 기반 해시 연산보다 빠른 성능을 위해 WASM 모듈을 사용합니다.
- WASM으로 빌드된 해시 함수는 Webpack의 번들 작성, 캐싱, 파일명 생성 등에 활용됩니다.

폴더 구조 예시:
```
assembly/
  hash/
    md4.asm.ts         # MD4 해시 알고리즘(AssemblyScript)
    xxhash64.asm.ts    # xxHash64 해시 알고리즘(AssemblyScript)
  tsconfig.json        # AssemblyScript/WASM 빌드 설정
```

<br/><br/>

## Webpack Contenthash와 해시 파일 처리 방식 정리

### 1. Contenthash란?

Webpack에서 `[contenthash]`는 번들 파일의 **내용을 바탕으로 계산된 해시값**을 의미합니다.  
출력 파일명에 `[contenthash]`를 사용하면, 파일 내용이 바뀌면 해시도 달라지고, 파일명도 변경됩니다.

예시:
```
output: {
  filename: "bundle.[contenthash].js"
}
```
→ 파일 내용이 변경될 때마다 `bundle.abcdef12.js`, `bundle.1234abcd.js`처럼 파일명이 달라집니다.

<br/>

### 2. 해시값 계산 원리

- Webpack은 파일의 **실제 데이터(텍스트 또는 바이너리)**를 해시 함수에 입력합니다.
- 주로 xxHash, MD4 등 빠른 해시 알고리즘을 사용하며, WASM(WebAssembly)로 구현되어 고성능 처리에 적합합니다.
- 해시 계산과정은 Webpack 내부의 [RealContentHashPlugin](https://github.com/KumJungMin/webpack-structure-study/blob/33c59646147c623f65d6366c9070f58b8e6390ce/lib/optimize/RealContentHashPlugin.js#L118) 등에서 처리합니다.

코드 예시:
```js
const content = getFileContent(); // 파일의 텍스트 또는 바이너리
const hash = createHash("xxhash64").update(content).digest("hex");
// hash 값이 [contenthash]로 파일명에 치환됨


// createHash:  https://github.com/KumJungMin/webpack-structure-study/blob/33c59646147c623f65d6366c9070f58b8e6390ce/lib/util/createHash.js
```

<br/>

### 3. 파일 타입별 해시 처리 방식

- **텍스트 파일(JS, CSS, HTML 등)**: 파일의 전체 문자열을 해시 함수에 입력.
- **바이너리 파일(이미지, 폰트 등)**: 파일의 전체 바이트(바이너리 데이터)를 해시 함수에 입력.
- **asset/resource 타입**: 이미지, 폰트 등도 내용 기반 해시로 `[contenthash]`가 계산됨.

예시(Webpack 5 asset/resource):
```js
module: {
  rules: [
    { test: /\.jpg$/, type: "asset/resource" }
  ]
},
output: {
  assetModuleFilename: "img/[name].[contenthash][ext]"
}
```

<br/>

### 4. Contenthash의 주요 목적

- **브라우저 캐시 문제 해결**: 파일 내용이 바뀌면 파일명도 바뀌어, 캐시가 자동으로 갱신됨.
- **캐시 효율성 증가**: 변경되지 않은 파일은 해시가 그대로라 캐시를 계속 사용.
- **배포 안정성**: 파일명이 항상 달라져 CDN/서버에서 옛 파일과 충돌 없음.

<br/>


### 5. 정리

- `[contenthash]`는 파일의 "내용"을 기준으로 고속 해시 알고리즘으로 계산된 값.
- 모든 파일 타입(텍스트/바이너리)에 대해 실제 데이터가 해시의 입력값이 됨.
- 결과적으로, 파일을 변경하면 해시와 파일명이 바뀌고, 캐시/배포가 안전하게 관리됨.


<br/><br />

---

<br/><br />


## WebAssembly로 구현된 해시 관련 코드와 WebAssembly 개요

### 1. WebAssembly란?

WebAssembly(웹어셈블리, WASM)는 웹 브라우저와 서버 등 다양한 환경에서 빠르고 효율적으로 실행 가능한 **저수준 이진 포맷**입니다.  
JS, C, C++, Rust, AssemblyScript 등 다양한 언어로 코드를 작성해 빌드하면 웹에서 네이티브처럼 동작할 수 있습니다.

### WebAssembly의 주요 특징
- **빠른 실행 속도**: 네이티브와 유사한 성능(자바스크립트보다 훨씬 빠름)
- **이식성**: 다양한 플랫폼(웹, 서버, 데스크탑)에서 실행 가능
- **언어 독립성**: C, C++, Rust, AssemblyScript 등 여러 언어 지원
- **보안**: 샌드박스 환경에서 안전하게 실행됨

<br />

### 2. WebAssembly의 장점

- **성능**: 반복 연산, 복잡한 계산, 이미지/비디오 처리, 해시 연산 등 CPU 집약적 작업에서 JS보다 월등히 빠름
- **모듈화**: JS에서 WASM 모듈을 쉽게 import하여 사용할 수 있음
- **생태계 확장**: 다양한 언어의 라이브러리를 웹에서 재사용 가능

<br />

### 3. JS와 WebAssembly 문법 차이점

| 구분        | JavaScript 예시                | AssemblyScript(WebAssembly) 예시      |
|-------------|-------------------------------|----------------------------------------|
| 변수 선언   | `let x = 1;`                  | `let x: i32 = 1;`                      |
| 함수 선언   | `function add(a, b) { ... }`  | `export function add(a: i32, b: i32): i32 { ... }` |
| 타입 명시   | 동적(예: `let x = "hi"`)      | 정적(예: `let x: string = "hi"`)       |
| 배열        | `const arr = [1,2,3];`        | `let arr: i32[] = [1,2,3];`            |
| 모듈 내보내기| `module.exports = ...`        | `export ...`                            |

- **타입 시스템**: AssemblyScript 등 WASM 언어는 명확한 타입(정수/실수 등)을 반드시 명시
- **내보내기/불러오기**: WASM 함수는 `export` 키워드로 외부에서 사용할 수 있게 함
- **런타임 제한**: 브라우저나 JS 런타임에서 직접 실행 불가, WASM 모듈로 로드해야 함

<br />

> AssemblyScript 예시 (해시 함수)
```typescript
export function add(a: i32, b: i32): i32 {
  return a + b;
}
```
위 코드는 AssemblyScript(타입스크립트와 유사)로 작성되어 WebAssembly 모듈로 빌드될 수 있습니다.

<br />

### 4. WebAssembly로 구현된 해시 관련 코드의 활용

Webpack 등 빌드 도구에서는 xxHash, MD4 등 해시 알고리즘을 WASM으로 구현하여  
- JS보다 빠르게 해시 연산 처리  
- 대용량 파일/번들에서 빌드 속도 향상  
- 파일 이름에 `[contenthash]` 등으로 해시값 적용 시 성능 개선

<br/>

### 5. 참고

- WASM은 JS와 다르게 타입을 명확히 선언하고, 브라우저/Node.js에서 모듈로 불러와 사용해야 함
- WASM 모듈은 JS에서 다음과 같이 불러올 수 있음:
  ```js
  const wasm = await WebAssembly.instantiateStreaming(fetch('hash.wasm'));
  const hashValue = wasm.instance.exports.xxhash64(dataPtr, length);
  ```

<br/>
