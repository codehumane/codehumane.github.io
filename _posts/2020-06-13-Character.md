---
layout: post
title: java.lang.Character
summary: Vector 간단 정리
date: 2020-06-13 00:00:01 +0900
---

자바의 [Character 문서](https://docs.oracle.com/javase/8/docs/api/java/lang/Character.html)에 [Unicode Character Representations](https://docs.oracle.com/javase/6/docs/api/java/lang/Character.html#unicode) 부분이 있음. 간단히 기록.

## Unicode Specification

1. `Character` 객체가 캡슐화하는 `char` 데이터 타입은 오리지널 유니코드 명세에 기반.
2. 이는 문자<sup>character</sup>를 고정 넓이의 16 비트 엔티티<sup>fixed-width 16-bit entitie</sup>로 정의.
3. `String`과 다르게 16비트가 넘어가는 문자를 표현할 수 없음을 말하는 듯.

## supplementary characters

1. `char` 데이터 타입은 오리지널 유니코드 명세에 기반했지만, 유니코드는 그 이후로도 계속 변화했고 현재는 16비트 이상으로 표현되는 문자를 허용.
2. 유효한 *코드 포인트<sup>code points</sup>* 범위는 현재 U+0000에서 U+10FFFF 까지.
3. *Unicode scalar value*라고 부름.
4. 이 중에서 U+0000에서 U+FFFF까지의 문자 집합을 *Basic Multilingual Plane*(BMP)라고 부름.
5. 그리고 U+FFFF를 넘어가는 코드 포인트의 문자들은 *supplementary characters*로 부름.
6. 자바 플랫폼은 UTF-16을 `char` 배열과 `String`, `StringBuffer` 클래스에서 사용.
7. 그래서 supplementary characters를 `char` 값의 짝으로 표현.
8. [About Supplementary Characters](https://docs.microsoft.com/en-us/windows/win32/intl/surrogates-and-supplementary-characters#about-supplementary-characters)에 따르면 이 방식을 *surrogate pair*라고 부름.
8. 첫 번째 값은 high-surrogates 범위(\uD800-\uDBFF)를 갖고,
9. 두 번째 값은 low-surrogates 범위(\uDC00-\uDFFF)를 가짐.

*참고로, `U+`나 `\u` 문자들은 뒤이은 숫자들이 16진수임을 나타냄. 

## code point

1. `int` 값은 모든 유니코드의 코드 포인트를 표현할 수 있음.
2. int의 최하위 21개 비트는 유니코드 코드 포인트를 나타내는 데 쓰임.
3. 나머지 최상위 11개 비트는 0.
4. 21개 비트가 쓰이는 까닭은 위에서 코드 포인트의 최대값이 U+10FFFF이기 때문.
5. 바이너리 숫자로는 100001111111111111111 즉, 21자리.
6. 10진수로는 111411.

## supplementary character behavior

1. `char` 값만을 인자로 받는 메서드는 supplementary characters 지원 불가.
2. 예컨대, `Character.isLetter('\uD840')`은 `false`를 반환.
3. 하지만 `\uD840`는 high-surrogates에 포함되는 대상. 문자열에서는 글자 표현에 사용되는 것.
4. 한편,`isLetter(int codePoint)`와 같이 int를 받는 메서드는 모든 유니코드 문자를 지원할 수 있음.
5. `Character.isLetter(0x2F81A)`는 `true` 반환.

## code point vs. code unit

문서 읽다 보면, code point와 code unit 차이가 궁금해짐. [여기](https://stackoverflow.com/questions/27331819/whats-the-difference-between-a-character-a-code-point-a-glyph-and-a-grapheme) 그 차이가 잘 설명되어 있음.

> For example, the snowman glyph (☃) is a single code point but 3 UTF-8 code units, and 1 UTF-16 code unit.
