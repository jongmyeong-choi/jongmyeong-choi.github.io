---
layout: post
title: "기본 Clang Parser–Sema 콜스택"
subtitle: "함수 정의에서 ReturnStmt 생성까지"
date: 2026-07-29 21:34:13 -0400
#background: '/img/posts/2025-01-24-01.jpeg'
published: true
categories: [tech]
tags: [clang, compiler, parser, ast, llvm, sema]
---

**Clang Parser–Sema 콜스택 분석: 함수 정의에서 ReturnStmt 생성까지**

---

### 입력 코드

간단한 함수 정의를 입력으로 사용합니다.

```cpp
int f() { return 0; }
```

---

### 콜스택 대응

이 코드를 파싱하는 동안 Clang 내부에서 호출되는 함수들의 콜스택입니다.
파싱은 `ParseAST`에서 시작하여 `ParseFunctionDefinition`을 통해 함수 정의를 처리하고,
결국 `ParseReturnStatement`를 거쳐 Sema 단계의 `ActOnReturnStmt`, `BuildReturnStmt`로 넘어갑니다.

```
#0  BuildReturnStmt
#1  ActOnReturnStmt
#2  ParseReturnStatement
#3  ParseStatementOrDeclarationAfterAttributes
#4  ParseStatementOrDeclaration
#5  ParseCompoundStatementBody
#6  ParseFunctionStatementBody
#7  ParseFunctionDefinition
#8  ParseDeclGroup
#9  ParseDeclOrFunctionDefInternal
#10 ParseDeclarationOrFunctionDefinition
#11 ParseExternalDeclaration
#12 ParseTopLevelDecl
#13 ParseFirstTopLevelDecl
#14 ParseAST
```

---

### AST 대응 구조

위의 입력 코드를 파싱한 결과로 생성된 AST 트리 구조는 다음과 같습니다.
각 AST 노드가 어느 parser/sema 함수에서 생성되는지 주석으로 대응시켰습니다.

```
`-FunctionDecl f 'int ()'             : ParseFunctionDefinition .... ActOnStartOfFunctionDef
  `-CompoundStmt                     : ParseCompoundStatementBody .... ActOnCompoundStmt
    `-ReturnStmt                    : ParseReturnStatement .... ActOnReturnStmt
      `-IntegerLiteral 'int' 0     : ParseExpression
```

---

### 전체 구조 요약

파싱과 AST 생성의 전체 과정을 정리하면 아래와 같습니다.
입력된 코드가 어떻게 처리되는지를 high-level에서 다시 요약한 것입니다.

```
소스코드
 ↓
Lexer → 토큰 스트림
 ↓
Parser → ParseFunctionDefinition → ... → ParseReturnStatement
 ↓
Sema → ActOnReturnStmt → BuildReturnStmt
 ↓
ASTContext 에 ReturnStmt 포함된 트리 구조 생성
```
