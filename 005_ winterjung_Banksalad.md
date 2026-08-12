원본코드
import abc
import enum
from datetime import date, time
from typing import Optional, Set

from .exception import InvalidLengthError, UnsupportedUnicodeError

__all__ = ['AN', 'N']


TYPES = Set[type]


class UnicodeErrorHandler(str, enum.Enum):
    STRICT = "strict"
    IGNORE = "ignore"
    REPLACE = "replace"


class KProperty(metaclass=abc.ABCMeta):
    def __init__(
        self, length: int, expected_types: TYPES, filler: bytes
    ) -> None:
        self.length = length
        self.expected_types = expected_types
        self.filler = filler

    @abc.abstractmethod
    def to_bytes(self, v: Optional) -> bytes:
        pass


class N(KProperty):

    EXPECTED_TYPES = {int, float}
    FILLER = b'0'
    ENCODING = 'ascii'

    def __init__(self, length: int, *, filler: Optional[bytes] = None) -> None:
        super().__init__(length, N.EXPECTED_TYPES, filler or N.FILLER)

    def to_bytes(self, v: Optional) -> bytes:
        try:
            p = b'-' if v < 0 else b''
            s = str(int(abs(v)))
        except TypeError:
            p, s = b'', ''

        b = p + bytes(s, encoding=N.ENCODING).rjust(
            self.length - len(p), self.filler
        )
        if len(b) > self.length:
            raise InvalidLengthError(self.length)
        return b


class AN(KProperty):

    EXPECTED_TYPES = N.EXPECTED_TYPES | {str, date, time, type(None)}
    FILLER = b' '
    ENCODING = 'euc-kr'
    DATE_FORMAT = '%Y%m%d'
    TIME_FORMAT = '%H%M%S%f'
    TIME_FORMAT_SLICE = 0, -2

    def __init__(
        self,
        length: int,
        *,
        filler: Optional[bytes] = None,
        errors: Optional[UnicodeErrorHandler] = UnicodeErrorHandler.STRICT
    ) -> None:
        super().__init__(length, AN.EXPECTED_TYPES, filler or AN.FILLER)
        self.errors = errors

    def to_bytes(self, v: Optional) -> bytes:
        if v is None or isinstance(v, str):
            s = v if v else ''
        elif isinstance(v, date):
            s = v.strftime(AN.DATE_FORMAT)
        elif isinstance(v, time):
            s = v.strftime(AN.TIME_FORMAT)[slice(*AN.TIME_FORMAT_SLICE)]
        else:
            s = str(int(v))

        try:
            b = bytes(s, encoding=AN.ENCODING, errors=self.errors.value).ljust(
                self.length, self.filler
            )
        except UnicodeError as e:
            raise UnsupportedUnicodeError(str(e)) from e

        if len(b) > self.length:
            raise InvalidLengthError(self.length)
        return b

고정 길이 금융 전문 포맷의 인코딩·패딩·추상화는 탄탄하지만, 입력 타입을 명시적으로 검증하지 않고 예외와 암묵적 형변환에 의존해 잘못된 값이 조용히 빈 데이터로 직렬화될 수 있다는 점이 가장 큰 약점이며, 타입 경계를 명확히 세우면 9점대 실무형 직렬화 계층으로 올라갈 수 있다.

제안패치
import abc
import enum
from datetime import date, time
from typing import Optional, Set, Type

from .exception import InvalidLengthError, UnsupportedUnicodeError


__all__ = ["AN", "N"]

TYPES = Set[Type]


class UnicodeErrorHandler(str, enum.Enum):
    STRICT = "strict"
    IGNORE = "ignore"
    REPLACE = "replace"


class KProperty(metaclass=abc.ABCMeta):
    def __init__(
        self,
        length: int,
        expected_types: TYPES,
        filler: bytes,
    ) -> None:
        if length < 0:
            raise ValueError("length must be non-negative")
        if len(filler) != 1:
            raise ValueError("filler must contain exactly one byte")

        self.length = length
        self.expected_types = expected_types
        self.filler = filler

    @abc.abstractmethod
    def to_bytes(self, v: Optional) -> bytes:
        raise NotImplementedError


class N(KProperty):
    EXPECTED_TYPES = {int, float}
    FILLER = b"0"
    ENCODING = "ascii"

    def __init__(
        self,
        length: int,
        *,
        filler: Optional[bytes] = None,
    ) -> None:
        super().__init__(
            length,
            N.EXPECTED_TYPES,
            N.FILLER if filler is None else filler,
        )

    def to_bytes(self, v: Optional) -> bytes:
        if not isinstance(v, (int, float)) or isinstance(v, bool):
            raise TypeError("N property requires int or float")

        sign = b"-" if v < 0 else b""
        value = str(int(abs(v)))

        available_length = self.length - len(sign)
        if available_length < 0:
            raise InvalidLengthError(self.length)

        encoded = value.encode(self.ENCODING)

        if len(encoded) > available_length:
            raise InvalidLengthError(self.length)

        return sign + encoded.rjust(available_length, self.filler)


class AN(KProperty):
    EXPECTED_TYPES = N.EXPECTED_TYPES | {
        str,
        date,
        time,
        type(None),
    }

    FILLER = b" "
    ENCODING = "euc-kr"
    DATE_FORMAT = "%Y%m%d"
    TIME_FORMAT = "%H%M%S%f"
    TIME_FORMAT_SLICE = 0, -2

    def __init__(
        self,
        length: int,
        *,
        filler: Optional[bytes] = None,
        errors: Optional[UnicodeErrorHandler] = UnicodeErrorHandler.STRICT,
    ) -> None:
        super().__init__(
            length,
            AN.EXPECTED_TYPES,
            AN.FILLER if filler is None else filler,
        )

        if not isinstance(errors, UnicodeErrorHandler):
            raise TypeError("errors must be a UnicodeErrorHandler")

        self.errors = errors

    def to_bytes(self, v: Optional) -> bytes:
        if v is None:
            value = ""
        elif isinstance(v, str):
            value = v
        elif isinstance(v, date):
            value = v.strftime(self.DATE_FORMAT)
        elif isinstance(v, time):
            value = v.strftime(self.TIME_FORMAT)[slice(*self.TIME_FORMAT_SLICE)]
        elif isinstance(v, (int, float)) and not isinstance(v, bool):
            value = str(int(v))
        else:
            raise TypeError("AN property received unsupported value type")

        try:
            encoded = value.encode(
                self.ENCODING,
                errors=self.errors.value,
            )
        except UnicodeError as exc:
            raise UnsupportedUnicodeError(str(exc)) from exc

        if len(encoded) > self.length:
            raise InvalidLengthError(self.length)

        return encoded.ljust(self.length, self.filler)

최종 개선사항
✅ 잘못된 입력을 빈 값으로 변환 → 명시적 타입 검증 후 즉시 실패 → 조용한 데이터 누락 방지
✅ EXPECTED_TYPES 선언만 존재 → 구현체에서 실제 타입 계약 검증 → 선언과 실행 동작의 불일치 제거
✅ filler or 기본값 → None만 기본값으로 취급 → 명시적 인자 의미 보존
✅ filler 길이 무검증 → 단일 바이트 계약 검증 → 고정 길이 직렬화 무결성 강화
✅ 추상 메서드의 pass → NotImplementedError → 미구현 구현체의 조기 실패 보장
✅ Unicode 예외 처리와 길이 검증은 기존 계약 유지 → 입력 검증만 강화 → 기존 직렬화 규칙의 회귀 위험 최소화

원본은 고정 길이 바이트 포맷이라는 목적에 맞게 상당히 잘 설계된 코드지만, 타입 계약을 선언하고도 실제 경계에서 검증하지 않는 것이 가장 큰 약점이다. 이 부분을 보강하되 직렬화 규칙 자체는 건드리지 않는 것이 9.5~9.8 수준의 가장 보수적인 개선이다.
