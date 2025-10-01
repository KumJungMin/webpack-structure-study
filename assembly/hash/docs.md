> hash 파일은 빌드된 파일명에 해시를 붙이기 위한 알고리즘을 관리.

# Webpack Contenthash와 해시 파일 처리 방식 정리

## 1. Contenthash란?

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

## 2. 해시값 계산 원리

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

## 3. 파일 타입별 해시 처리 방식

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

## 4. Contenthash의 주요 목적

- **브라우저 캐시 문제 해결**: 파일 내용이 바뀌면 파일명도 바뀌어, 캐시가 자동으로 갱신됨.
- **캐시 효율성 증가**: 변경되지 않은 파일은 해시가 그대로라 캐시를 계속 사용.
- **배포 안정성**: 파일명이 항상 달라져 CDN/서버에서 옛 파일과 충돌 없음.

<br/>

## 5. 정리

- `[contenthash]`는 파일의 "내용"을 기준으로 고속 해시 알고리즘으로 계산된 값.
- 모든 파일 타입(텍스트/바이너리)에 대해 실제 데이터가 해시의 입력값이 됨.
- 결과적으로, 파일을 변경하면 해시와 파일명이 바뀌고, 캐시/배포가 안전하게 관리됨.
