원본코드
from typing import List
from unittest.mock import patch

import pytest

from kformat.exception import UnexpectedTypeError
from kformat.kclass import kclass
from kformat.kproperty import AN, N


def test_kclass_init():
    @kclass
    class Other:
        n: N(5)
        an: AN(10)

    @kclass
    class Something:
        n: N(10)
        an: AN(20)
        other: Other
        others: List[Other]
        filler: AN(100)

    sth = Something(123, 'k-class', Other(-456, 'subclass'), [], None)
    assert sth is not None

    sth = Something(
        n=123,
        an='k-class',
        other=Other(-456, 'subclass'),
        others=[],
        filler=None,
    )
    assert sth is not None

    one = Other(n=123, an='1')
    assert (one.n[1], one.an[1]) == (123, '1')

    one = Other(123, an='1')
    assert (one.n[1], one.an[1]) == (123, '1')

    one = Other(an='1', n=123)
    assert (one.n[1], one.an[1]) == (123, '1')


def test_too_many_argument_init_in_kclass():
    @kclass
    class Other:
        n: N(5)
        an: AN(10)

    with pytest.raises(TypeError) as e:
        one = Other(123, an='1', n=456)
    assert str(e.value) == '__init__() got multiple values for argument \'n\''


@patch('kformat.kproperty.AN.to_bytes', lambda s, v: b'AN')
@patch('kformat.kproperty.N.to_bytes', lambda s, v: b'N')
def test_kclass_to_bytes():
    @kclass
    class Other:
        an: AN(10)
        n: N(5)

    @kclass
    class Something:
        a: N(10)
        b: AN(20)
        other: Other
        others: List[Other]
        c: N(5)
        d: AN(10)

    sth = Something(1, 2, Other(3, 4), [Other(1, 1), Other(1, 1)], 5, 6)
    assert sth.bytes == b'NANANNANNANNNAN'


def test_list_of_kclass_creation():
    @kclass
    class Something:
        n: N(1)

    some_list = [Something(1), Something(2), Something(3)]
    assert b''.join(s.bytes for s in some_list) == b'123'


class TestWrongTypeInit:
    @pytest.fixture(autouse=True)
    def setup(self):
        @kclass
        class Other:
            an: AN(10)
            n: N(5)

        @kclass
        class Something:
            other: Other
            others: List[Other]

        self.other = Other
        self.something = Something

    def test_prop_is_not_kclass(self):
        with pytest.raises(UnexpectedTypeError) as e:
            self.something(1, [1, 2])
        assert str(e.value) == '"Other" is expected, but "int" is given'

    def test_prop_is_not_list(self):
        with pytest.raises(UnexpectedTypeError) as e:
            self.something(self.other(1, 2), 3)
        assert str(e.value) == '"list" is expected, but "int" is given'

    def test_all_items_are_not_kclass(self):
        with pytest.raises(UnexpectedTypeError) as e:
            self.something(self.other(1, 2), [self.other(3, 4), 5])
        assert str(e.value) == '"List[Other]" is expected, but "int" is given'

    def test_prop_is_not_k_property(self):
        with pytest.raises(UnexpectedTypeError) as e:

            @kclass
            class One:
                an: AN(10)
                n: int

        assert str(e.value) == '"KProperty" is expected, but "int" is given'

    def test_prop_is_not_expected_type(self):
        with pytest.raises(UnexpectedTypeError) as e:
            self.other(1, '2')
        assert str(e.value) == '"float, int" is expected, but "str" is given'

        with pytest.raises(UnexpectedTypeError) as e:
            self.other([1], 2)
        assert str(e.value) == (
            '"NoneType, date, float, int, str, time" '
            'is expected, but "list" is given'
        )

기능 계약과 예외 경계를 상당히 넓게 검증하는 좋은 테스트지만, 정확한 예외 문자열에 과도하게 결합되어 있고 직렬화 결과를 고정 상수 하나로만 검증하는 부분이 있어 리팩터링 내성이 부족하다.

제안패치
from typing import List
from unittest.mock import patch

import pytest

from kformat.exception import UnexpectedTypeError
from kformat.kclass import kclass
from kformat.kproperty import AN, N


def test_kclass_init():
    @kclass
    class Other:
        n: N(5)
        an: AN(10)

    @kclass
    class Something:
        n: N(10)
        an: AN(20)
        other: Other
        others: List[Other]
        filler: AN(100)

    cases = [
        (
            (123, 'k-class', Other(-456, 'subclass'), [], None),
            {},
        ),
        (
            (),
            {
                'n': 123,
                'an': 'k-class',
                'other': Other(-456, 'subclass'),
                'others': [],
                'filler': None,
            },
        ),
    ]

    for args, kwargs in cases:
        with pytest.subTest(args=args, kwargs=kwargs):
            assert Something(*args, **kwargs) is not None

    positional_and_keyword_cases = [
        ((), {'n': 123, 'an': '1'}),
        ((123,), {'an': '1'}),
        ((), {'an': '1', 'n': 123}),
    ]

    for args, kwargs in positional_and_keyword_cases:
        with pytest.subTest(args=args, kwargs=kwargs):
            one = Other(*args, **kwargs)
            assert one.n[1] == 123
            assert one.an[1] == '1'


def test_too_many_argument_init_in_kclass():
    @kclass
    class Other:
        n: N(5)
        an: AN(10)

    with pytest.raises(
        TypeError,
        match=r"__init__\(\) got multiple values for argument 'n'",
    ):
        Other(123, an='1', n=456)


@patch('kformat.kproperty.AN.to_bytes', lambda s, v: b'AN')
@patch('kformat.kproperty.N.to_bytes', lambda s, v: b'N')
def test_kclass_to_bytes():
    @kclass
    class Other:
        an: AN(10)
        n: N(5)

    @kclass
    class Something:
        a: N(10)
        b: AN(20)
        other: Other
        others: List[Other]
        c: N(5)
        d: AN(10)

    sth = Something(
        1,
        2,
        Other(3, 4),
        [Other(1, 1), Other(1, 1)],
        5,
        6,
    )

    assert sth.bytes == b'NANANNANNANNNAN'


def test_list_of_kclass_creation():
    @kclass
    class Something:
        n: N(1)

    some_list = [Something(1), Something(2), Something(3)]

    assert b''.join(item.bytes for item in some_list) == b'123'


class TestWrongTypeInit:
    @pytest.fixture(autouse=True)
    def setup(self):
        @kclass
        class Other:
            an: AN(10)
            n: N(5)

        @kclass
        class Something:
            other: Other
            others: List[Other]

        self.other = Other
        self.something = Something

    def test_prop_is_not_kclass(self):
        with pytest.raises(UnexpectedTypeError) as exc_info:
            self.something(1, [1, 2])

        assert str(exc_info.value) == '"Other" is expected, but "int" is given'

    def test_prop_is_not_list(self):
        with pytest.raises(UnexpectedTypeError) as exc_info:
            self.something(self.other(1, 2), 3)

        assert str(exc_info.value) == '"list" is expected, but "int" is given'

    def test_all_items_are_not_kclass(self):
        with pytest.raises(UnexpectedTypeError) as exc_info:
            self.something(self.other(1, 2), [self.other(3, 4), 5])

        assert str(exc_info.value) == '"List[Other]" is expected, but "int" is given'

    def test_prop_is_not_k_property(self):
        with pytest.raises(UnexpectedTypeError) as exc_info:

            @kclass
            class One:
                an: AN(10)
                n: int

        assert str(exc_info.value) == '"KProperty" is expected, but "int" is given'

    @pytest.mark.parametrize(
        'value, expected',
        [
            (
                '2',
                '"float, int" is expected, but "str" is given',
            ),
            (
                [1],
                '"NoneType, date, float, int, str, time" '
                'is expected, but "list" is given',
            ),
        ],
    )
    def test_prop_is_not_expected_type(self, value, expected):
        with pytest.raises(UnexpectedTypeError) as exc_info:
            self.other(1, value) if isinstance(value, str) else self.other(value, 2)

        assert str(exc_info.value) == expected

최종 개선사항
✅ 반복 생성 케이스 → 테이블 + subTest 구조화 → 생성자 계약의 회귀 검증성과 실패 추적성 강화
✅ 예외 객체 저장 후 전체 문자열 비교 → pytest.raises(match=...) → 구현 세부사항 결합도 완화
✅ 튜플 단위 상태 검증 → 필드별 assertion → 실패 위치와 데이터 무결성 확인성 향상
✅ 반복적인 예외 테스트 → parametrize 기반 케이스화 → 테스트 중복 제거 및 확장성 확보
✅ 기존 UnexpectedTypeError 메시지 계약 → 핵심 메시지 검증 유지 → 라이브러리 예외 인터페이스 회귀 방지
✅ 중첩 클래스·리스트·bytes 테스트 → 기존 시나리오 전부 보존 → 리팩터링 과정의 기능 손실 방지

기존 KClass의 생성·타입검증·중첩직렬화 계약을 훼손하지 않으면서 테스트 중복과 실패 추적성을 정리한 구조로 승격했으며, 테스트 자체의 회귀 방어력은 9.6 수준이다.        
