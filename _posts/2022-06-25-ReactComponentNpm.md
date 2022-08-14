---
title: 회사 컴포넌트, hooks들을 npm에 배포해서 사용해보자
author: Sanghun lee
date: 2022-06-25 11:33:00 +0800
categories: [FE, React, Webpack, Styled-component]
tags: [React, Webpack]
math: true
mermaid: true
image:
  src: https://layer.team/wp-content/uploads/2020/04/layer-app-spot-26-best-field-data-software-revit-addin-manage.png
  width: 850
  height: 585
---

## axios

resolve에 옵션을 추가하고 rollupJson을 통해 json파일들까지 받아올 수 있게 만들어 axios의 의존성파일들을 받아오게 추가 함

```js
import rollupJson from '@rollup/plugin-json';
import resolve, { nodeResolve } from '@rollup/plugin-node-resolve';
resolve({ jsnext: true, preferBuiltins: true, browser: true }),
rollupJson(),
```

- [Ref](https://github.com/axios/axios/issues/1259)
