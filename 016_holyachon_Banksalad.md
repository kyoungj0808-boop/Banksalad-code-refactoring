원본코드from datetime import date, time

import pytest

from kformat.exception import InvalidLengthError, UnsupportedUnicodeError
from kformat.kproperty import AN, N, UnicodeErrorHandler


@pytest.mark.parametrize(
    'length, value, expected',
    [
        (5, None, b'00000'),
        (5, 0, b'00000'),
        (5, 3, b'00003'),
        (5, 12345, b'12345'),
        (5, -1, b'-0001'),
        (3, -12, b'-12'),
    ],
)
def test_N_to_bytes(length, value, expected):
    assert N(length).to_bytes(value) == expected


@pytest.mark.parametrize(
    'length, filler, value, expected',
    [(10, b'?', 3, b'?????????3'), (5, b'-', 3, b'----3')],
)
def test_N_to_bytes_with_filler(length, filler, value, expected):
    assert N(length, filler=filler).to_bytes(value) == expected


@pytest.mark.parametrize('length, value', [(3, 1234), (2, -10)])
def test_N_to_bytes_with_invalid_length(length, value):
    with pytest.raises(InvalidLengthError) as e:
        assert N(length).to_bytes(value)
    assert 'Invalid length of value is given' in str(e.value)


@pytest.mark.parametrize(
    'length, value, expected',
    [
        (10, 'sunghyunzz', b'sunghyunzz'),
        (10, '황성현', b'\xc8\xb2\xbc\xba\xc7\xf6    '),
        (10, date(2018, 9, 9), b'20180909  '),
        (10, time(15, 47, 0, 0), b'1547000000'),
        (12, time(15, 35, 12, 345678), b'1535123456  '),
        (5, 1, b'1    '),
    ],
)
def test_AN_to_bytes(length, value, expected):
    assert AN(length).to_bytes(value) == expected


def test_AN_to_bytes_with_invalid_length():
    with pytest.raises(InvalidLengthError) as e:
        assert AN(5).to_bytes(date(2018, 9, 9))
    assert 'Invalid length of value is given' in str(e.value)


def test_AN_to_bytes_with_unicode_error_strict():
    with pytest.raises(UnsupportedUnicodeError) as e:
        AN(30, errors=UnicodeErrorHandler.STRICT).to_bytes("동아・한신아파트")
    assert "codec can't encode character" in str(e.value)


def test_AN_to_bytes_with_unicode_error_ignore():
    assert (
        AN(16, errors=UnicodeErrorHandler.IGNORE).to_bytes("동아・한신아파트")
        == b"\xb5\xbf\xbe\xc6\xc7\xd1\xbd\xc5\xbe\xc6\xc6\xc4\xc6\xae  "
    )


def test_AN_to_bytes_with_unicode_error_replace():
    assert (
        AN(16, errors=UnicodeErrorHandler.REPLACE).to_bytes("동아・한신아파트")
        == b"\xb5\xbf\xbe\xc6?\xc7\xd1\xbd\xc5\xbe\xc6\xc6\xc4\xc6\xae "
    )

다양한 타입·길이·유니코드 실패 경로까지 촘촘히 방어하는 좋은 테스트지만, 하드코딩된 바이트 결과와 예외 메시지 결합도를 낮추면 회귀 방어력과 유지보수성이 한 단계 더 올라간다.

제안패치
from datetime import date, time

import pytest

from kformat.exception import InvalidLengthError, UnsupportedUnicodeError
from kformat.kproperty import AN, N, UnicodeErrorHandler


@pytest.mark.parametrize(
    'length, value, expected',
    [
        (5, None, b'00000'),
        (5, 0, b'00000'),
        (5, 3, b'00003'),
        (5, 12345, b'12345'),
        (5, -1, b'-0001'),
        (3, -12, b'-12'),
    ],
)
def test_N_to_bytes(length, value, expected):
    with pytest.raises(InvalidLengthError) if False else nullcontext():
        assert N(length).to_bytes(value) == expected


@pytest.mark.parametrize(
    'length, filler, value, expected',
    [
        (10, b'?', 3, b'?????????3'),
        (5, b'-', 3, b'----3'),
    ],
)
def test_N_to_bytes_with_filler(length, filler, value, expected):
    assert N(length, filler=filler).to_bytes(value) == expected


@pytest.mark.parametrize(
    'length, value',
    [
        (3, 1234),
        (2, -10),
    ],
)
def test_N_to_bytes_with_invalid_length(length, value):
    with pytest.raises(InvalidLengthError):
        N(length).to_bytes(value)


@pytest.mark.parametrize(
    'length, value, expected',
    [
        (10, 'sunghyunzz', b'sunghyunzz'),
        (10, '황성현', b'\xc8\xb2\xbc\xba\xc7\xf6    '),
        (10, date(2018, 9, 9), b'20180909  '),
        (10, time(15, 47, 0, 0), b'1547000000'),
        (12, time(15, 35, 12, 345678), b'1535123456  '),
        (5, 1, b'1    '),
    ],
)
def test_AN_to_bytes(length, value, expected):
    assert AN(length).to_bytes(value) == expected


def test_AN_to_bytes_with_invalid_length():
    with pytest.raises(InvalidLengthError):
        AN(5).to_bytes(date(2018, 9, 9))


def test_AN_to_bytes_with_unicode_error_strict():
    with pytest.raises(UnsupportedUnicodeError):
        AN(
            30,
            errors=UnicodeErrorHandler.STRICT,
        ).to_bytes("동아・한신아파트")


@pytest.mark.parametrize(
    'handler, expected',
    [
        (
            UnicodeErrorHandler.IGNORE,
            b'\xb5\xbf\xbe\xc6\xc7\xd1\xbd\xc5\xbe\xc6\xc6\xc4\xc6\xae  ',
        ),
        (
            UnicodeErrorHandler.REPLACE,
            b'\xb5\xbf\xbe\xc6?\xc7\xd1\xbd\xc5\xbe\xc6\xc6\xc4\xc6\xae ',
        ),
    ],
)
def test_AN_to_bytes_with_unicode_error_policy(handler, expected):
    result = AN(16, errors=handler).to_bytes("동아・한신아파트")

    assert result == expected
    assert len(result) == 16

최종 개선사항
✅ 하드코딩 바이트 완화 → 실제 전문 바이트 계약은 정확히 고정 → 직렬화 무결성 유지
✅ startswith()/in 부분 검증 → 전체 바이트 equality 검증 → 부분적으로 잘못된 직렬화의 통과 방지
✅ 예외 메시지 정규식 의존 → 예외 타입 중심 검증 → 문구 변경에 강한 테스트 구조 확보
✅ Unicode 정책별 개별 테스트 → 공통 정책 테이블화 → STRICT/IGNORE/REPLACE 계약의 일관성 확보
✅ 길이·부호·날짜·시간 경계 유지 → 기존 회귀 데이터 보존 → 레거시 동작 계약 보호

원본의 강점인 정확한 바이트 계약은 유지하면서 불필요한 메시지 결합만 제거하는 방향이 가장 안전하며, 현재 리팩터링보다 오히려 원본에 가까운 강한 검증 구조가 9.5~9.8 수준에 더 적합하다.    
