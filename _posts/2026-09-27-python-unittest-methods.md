---
layout: post
title: "파이썬 unittest에서 실제로 자주 쓰는 메서드 정리 — assertEqual부터 subTest까지"
date: 2026-09-27
categories: [테크]
tags: [Python, unittest, 단위테스트, 테스트코드, assertEqual, subTest]
---

이번 포스팅에서는 파이썬 표준 라이브러리 `unittest`에서 **실제로 자주 쓰는 메서드**를 정리합니다. 공식 문서에는 assert 메서드가 30개 넘게 나열되어 있지만, 실무에서 반복적으로 손이 가는 것은 그중 10개 정도입니다. 전부 외우려 하기보다 사용 빈도가 높은 것부터 확실히 익히는 편이 효율적입니다.

단순 목록 나열이 아니라, **왜 그 메서드를 써야 하는지**를 실패 메시지 비교로 확인하는 데 초점을 맞췄습니다.

---

## 시작점: 가장 기본적인 테스트

먼저 테스트 대상이 될 코드입니다.

```python
# calc.py
def add(a, b):
    return a + b
```

이것을 테스트하는 가장 기본적인 형태입니다.

```python
# test_calc.py
import unittest
import calc


class TestCalc(unittest.TestCase):

    def test_add(self):
        result = calc.add(1, 2)
        self.assertEqual(result, 3)

    def test_add2(self):
        result = calc.add(0, -1)
        self.assertEqual(result, -1)


if __name__ == "__main__":
    unittest.main()
```

규칙은 세 가지입니다.

1. `unittest.TestCase`를 상속한 클래스를 만든다
2. 메서드 이름은 반드시 `test_`로 시작한다 (이 접두어가 없으면 실행되지 않는다)
3. `self.assertXxx()`로 검증한다

실행 방법은 여러 가지입니다.

```bash
python -m unittest test_calc -v                    # 모듈 단위
python -m unittest test_calc.TestCalc -v           # 클래스 단위
python -m unittest test_calc.TestCalc.test_add -v  # 메서드 하나만
python -m unittest discover                        # 자동 탐색
```

---

## Tier 1: 이 4개가 사용량의 대부분

### assertEqual — 압도적 1위

`==` 비교입니다. 전체 assert 사용량의 절반 이상을 차지합니다.

```python
self.assertEqual(add(1, 2), 3)
self.assertNotEqual(add(1, 2), 4)
```

여기서 자주 놓치는 점이 있습니다. **리스트와 딕셔너리도 그냥 `assertEqual`로 비교합니다.**

```python
self.assertEqual([1, 2, 3], [1, 2, 3])
self.assertEqual({"a": 1}, {"a": 1})
```

`assertListEqual`, `assertDictEqual`이 따로 있지만 거의 쓸 필요가 없습니다. `assertEqual`이 인자의 타입을 보고 알아서 적절한 비교 함수로 위임하고, 실패 시 diff까지 출력해 줍니다.

### assertTrue / assertFalse — "참/거짓 자체"를 볼 때만

```python
cart = Cart()
self.assertFalse(cart.items)      # 빈 리스트는 falsy
cart.add_item("book")
self.assertTrue(cart.items)       # 값이 있으면 truthy
```

용도가 좁습니다. 이 메서드를 잘못 쓰는 패턴이 초보 단계에서 가장 흔한 실수인데, 바로 다음 절에서 다룹니다.

### assertRaises — 예외 검증

예외가 제대로 발생하는지 검사합니다. `with` 문(컨텍스트 매니저)으로 쓰는 것이 표준입니다.

```python
def divide(a, b):
    if b == 0:
        raise ZeroDivisionError("0으로 나눌 수 없습니다")
    return a / b
```

```python
with self.assertRaises(ZeroDivisionError):
    divide(10, 0)
```

예외 메시지까지 확인하려면 `as`로 받습니다.

```python
with self.assertRaises(ZeroDivisionError) as ctx:
    divide(10, 0)
self.assertIn("0으로 나눌 수 없습니다", str(ctx.exception))

# 정규식으로 검사하는 축약형
with self.assertRaisesRegex(ZeroDivisionError, "나눌 수 없"):
    divide(1, 0)
```

주의할 점은 `self.assertRaises(ZeroDivisionError, divide(10, 0))`처럼 쓰면 안 된다는 것입니다. 인자로 넘기는 순간 예외가 먼저 터져 버립니다. `with` 문을 쓰거나 `assertRaises(Exc, func, arg1, arg2)` 형태로 **함수와 인자를 분리해서** 넘겨야 합니다.

### assertIn — 포함 관계

리스트, 문자열, 딕셔너리 키에 모두 동작합니다.

```python
self.assertIn("book", cart.items)
self.assertNotIn("laptop", cart.items)
self.assertIn("ok", "status: ok")     # 문자열 부분 검사에 특히 자주 쓴다
```

API 응답에서 특정 문자열이 포함됐는지 확인할 때 많이 씁니다.

---

## 핵심: assertTrue로 비교하지 말 것

가장 강조하고 싶은 부분입니다. 아래 두 코드는 **같은 것을 검사하지만 실패했을 때의 정보량이 완전히 다릅니다.**

```python
def test_bad(self):
    self.assertTrue(add(2, 3) == 999)   # 안티패턴

def test_good(self):
    self.assertEqual(add(2, 3), 999)    # 권장
```

실제 실행 결과입니다.

```
FAIL: test_bad
AssertionError: False is not true

FAIL: test_good
AssertionError: 5 != 999
```

`assertTrue`는 `False is not true`라고만 말합니다. 실제 값이 무엇이었는지 알 수 없어서 디버깅을 처음부터 다시 해야 합니다. `assertEqual`은 `5 != 999`로 실제값을 바로 보여줍니다.

이유는 단순합니다. `assertTrue`에 도달하는 시점에 `add(2, 3) == 999`는 이미 `False`라는 불리언 하나로 뭉개져 버립니다. 원본 값 정보가 사라진 뒤에 전달되는 것입니다. `assertEqual`은 두 값을 각각 받으므로 비교 결과와 원본 값을 모두 보고할 수 있습니다.

리스트는 차이점까지 짚어 줍니다.

```
AssertionError: Lists differ: [1, 2, 3, 4] != [1, 2, 9, 4]

First differing element 2:
3
9

- [1, 2, 3, 4]
?        ^
+ [1, 2, 9, 4]
?        ^
```

`assertTrue(list_a == list_b)`로 썼다면 이 정보를 전부 잃습니다.

**정리하면, 비교하려는 값이 두 개 있다면 항상 두 값을 각각 넘기는 메서드를 쓰는 것이 원칙입니다.** 크기 비교도 마찬가지로 `assertTrue(a > b)` 대신 `assertGreater(a, b)`를 씁니다.

---

## Tier 2: 해당 상황이 오면 반드시 필요한 것들

### assertIsNone / assertIsNotNone

`None` 검사는 `==`가 아니라 `is`로 해야 합니다. `dict.get()`, `re.match()`, DB 조회처럼 "없으면 None을 반환"하는 함수를 테스트할 때 계속 등장합니다.

```python
def find_user(user_id):
    users = {1: "alex", 2: "kim"}
    return users.get(user_id)  # 없으면 None
```

```python
self.assertIsNone(find_user(999))
self.assertIsNotNone(find_user(1))
```

### assertAlmostEqual — float 비교는 무조건 이것

부동소수점 연산에는 오차가 있습니다. 유명한 예시입니다.

```python
self.assertNotEqual(0.1 + 0.2, 0.3)      # 놀랍지만 통과한다
self.assertAlmostEqual(0.1 + 0.2, 0.3)   # 이게 맞는 방법
```

`0.1 + 0.2`는 실제로 `0.30000000000000004`입니다. `assertEqual`로 비교하면 실패합니다. 소수점 자릿수를 지정할 수도 있습니다.

```python
self.assertAlmostEqual(divide(10, 3), 3.333, places=3)
```

금액 계산, 평균, 비율처럼 float를 다루는 테스트에서는 이걸 쓰지 않으면 원인 모를 실패를 만나게 됩니다.

### assertIsInstance — 타입 검사

```python
self.assertIsInstance(add(1, 2), int)
self.assertIsInstance(divide(10, 2), float)
```

`type(x) == int`보다 권장됩니다. 상속 관계를 고려하기 때문입니다.

### assertIs / assertIsNot — 값이 아니라 객체 동일성

```python
a = [1, 2]
b = a
c = [1, 2]

self.assertIs(a, b)        # 같은 객체
self.assertIsNot(a, c)     # 값은 같지만 다른 객체
self.assertEqual(a, c)     # 값 비교는 통과
```

싱글턴이나 캐시 동작을 검증할 때 씁니다.

### assertCountEqual — 순서 무시 비교

이름이 헷갈리는 메서드입니다. '개수'를 비교하는 게 아니라 **순서와 무관하게 같은 원소들로 구성됐는지**를 검사합니다.

```python
self.assertCountEqual([1, 2, 3], [3, 1, 2])   # 통과
self.assertEqual([1, 2, 3], [3, 1, 2])        # 실패
```

DB 조회 결과처럼 순서가 보장되지 않는 값을 검증할 때 의외로 자주 필요합니다.

---

## Tier 3: 테스트 구조를 만드는 메서드

assert 계열이 아니지만 사용 빈도로는 상위권인 메서드들입니다.

### setUp / tearDown

`setUp()`은 **각 `test_` 메서드 실행 직전마다 매번** 새로 실행됩니다. 이 덕분에 각 테스트가 서로 영향을 주지 않는 깨끗한 상태에서 시작합니다.

```python
class TestCart(unittest.TestCase):

    @classmethod
    def setUpClass(cls):
        # 클래스 전체에서 딱 1번. 비용이 큰 준비 작업(DB 연결 등)
        cls.shared_config = {"env": "test"}

    def setUp(self):
        # 각 테스트 직전마다 매번 실행
        self.cart = Cart()
        self.cart.add_item("book")

    def tearDown(self):
        # 각 테스트 직후마다 실행. 파일 삭제, 연결 종료 등
        self.cart = None

    def test_1(self):
        self.cart.add_item("pen")
        self.assertEqual(self.cart.total_count(), 2)

    def test_2(self):
        # test_1에서 pen을 넣었지만 setUp이 다시 실행돼서 book만 있다
        self.assertEqual(self.cart.total_count(), 1)
```

`test_1`에서 장바구니에 항목을 추가했지만 `test_2`는 영향을 받지 않습니다. 이 독립성이 `setUp`을 쓰는 이유입니다. 클래스 변수에 상태를 두면 테스트 실행 순서에 따라 결과가 달라지는 문제가 생깁니다.

`setUpClass`는 클래스당 한 번만 실행되므로 DB 커넥션처럼 매번 만들기에는 비용이 큰 작업에 씁니다. `@classmethod` 데코레이터가 필요합니다.

### subTest — 반복 케이스 검사

알아두면 매우 유용한데 상대적으로 덜 알려진 메서드입니다. 여러 입력 케이스를 `for` 문으로 검사할 때, `subTest` 없이 쓰면 **첫 실패에서 멈춰** 나머지 케이스 결과를 볼 수 없습니다.

```python
def test_subtest(self):
    cases = [
        (1, 1, 2),
        (2, 2, 99),   # 일부러 틀림
        (3, 3, 88),   # 일부러 틀림
    ]
    for a, b, expected in cases:
        with self.subTest(a=a, b=b):   # 실패 시 어떤 입력이었는지 표시
            self.assertEqual(add(a, b), expected)
```

실제 실행 결과입니다.

```
FAIL: test_subtest (a=2, b=2)
AssertionError: 4 != 99

FAIL: test_subtest (a=3, b=3)
AssertionError: 6 != 88

Ran 1 test
FAILED (failures=2)
```

두 가지를 확인할 수 있습니다. 첫 실패에서 멈추지 않고 **모든 실패 케이스를 보고**하며, `(a=2, b=2)`처럼 **어떤 입력에서 실패했는지**까지 표시합니다. 테스트는 1개인데 실패는 2건으로 집계됩니다.

### skip — 임시로 건너뛰기

```python
@unittest.skip("아직 구현 안 된 기능")
def test_not_ready(self):
    ...

@unittest.skipIf(sys.platform == "win32", "윈도우에서는 미지원")
def test_unix_only(self):
    ...
```

테스트를 주석 처리하는 것보다 낫습니다. 실행 결과에 `skipped`로 남아서 "건너뛴 테스트가 있다"는 사실이 드러납니다.

---

## 파일 이름 규칙 함정

정리하다가 직접 부딪힌 문제입니다. 테스트 파일을 `main.py`로 저장해 두고 `python -m unittest discover`를 실행했는데 테스트가 하나도 실행되지 않았습니다.

`discover`의 기본 탐색 패턴이 **`test*.py`**이기 때문입니다. `main.py`는 이 패턴에 걸리지 않아서 조용히 무시됩니다. 에러도 나지 않기 때문에 알아차리기 어렵습니다.

```bash
python -m unittest discover              # main.py는 탐색되지 않음
python -m unittest main                  # 파일명을 직접 지정하면 동작
python -m unittest discover -p "*_test.py"   # 패턴을 바꿀 수도 있다
```

관례대로 **`test_` 접두어를 붙인 파일명**을 쓰는 것이 안전합니다. `calc.py`를 테스트한다면 `test_calc.py`가 됩니다. CI에서 `discover`를 돌리는 경우 파일명 규칙을 어기면 테스트가 실행되지 않은 채로 초록불이 떠서 더 위험합니다.

---

## 자주 쓰는 메서드 정리

| 메서드 | 용도 | 비고 |
|--------|------|------|
| `assertEqual(a, b)` | `==` 비교 | 1순위. 리스트·딕셔너리도 이걸로 |
| `assertRaises(Exc)` | 예외 발생 검증 | `with` 문으로 사용 |
| `assertIn(a, b)` | 포함 관계 | 리스트·문자열·dict 키 |
| `assertTrue/False(x)` | 참/거짓 자체 | 비교에는 쓰지 말 것 |
| `assertIsNone(x)` | `None` 검사 | `get()`, `match()` 결과 검증 |
| `assertAlmostEqual(a, b)` | float 비교 | float에는 필수 |
| `assertIsInstance(x, cls)` | 타입 검사 | `type()` 비교보다 권장 |
| `assertGreater/Less(a, b)` | 크기 비교 | `assertTrue(a > b)` 대신 |
| `assertCountEqual(a, b)` | 순서 무시 비교 | 이름 주의 ('개수' 아님) |
| `setUp()` | 각 테스트 직전 초기화 | 테스트 독립성 확보 |
| `subTest()` | 반복 케이스 검사 | 전체 실패 케이스 확인 |

---

## 마치며

assert 메서드를 전부 외울 필요는 없습니다. `assertEqual`, `assertRaises`, `assertIn`, `setUp`만 손에 익으면 대부분의 테스트를 작성할 수 있고, 나머지는 필요한 상황이 왔을 때 찾아 쓰면 됩니다.

다만 **`assertTrue(a == b)` 패턴은 처음부터 쓰지 않는 습관을 들이는 게 좋습니다.** 테스트는 통과할 때가 아니라 실패할 때 가치를 만듭니다. 실패 메시지가 `False is not true`인지 `5 != 999`인지의 차이가, 문제를 5초에 파악할지 30분을 헤맬지를 결정합니다. 메서드를 정확히 골라 쓰는 것 자체가 디버깅 시간에 대한 투자입니다.

`unittest`에 익숙해진 다음 단계로는 `unittest.mock`의 `patch`를 볼 만합니다. 외부 API 호출이나 DB 접근을 가로채서 네트워크 없이 테스트하는 방법인데, 실제 프로젝트 테스트에서는 assert 메서드만큼 자주 쓰입니다.
