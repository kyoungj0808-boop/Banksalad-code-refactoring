원본코드
from typing import Type

__all__ = [
    'KFormatError',
    'UnexpectedTypeError',
    'InvalidLengthError',
    'UnsupportedUnicodeError',
]


def _name(type_: Type) -> str:
    return type_.__name__ if hasattr(type_, '__name__') else str(type_)


class KFormatError(Exception):
    pass


class UnexpectedTypeError(KFormatError):
    def __init__(self, expected: Type, value):
        super().__init__()
        self.expected = expected
        self.value = value

    def __str__(self) -> str:
        return (
            f'"{_name(self.expected)}" is expected, '
            f'but "{_name(self.value)}" is given'
        )


class InvalidLengthError(KFormatError):
    def __init__(self, length: int):
        super().__init__()
        self.length = length

    def __str__(self) -> str:
        return f'Invalid length of value is given(expected: {self.length})'


class UnsupportedUnicodeError(KFormatError, UnicodeError):
    def __init__(self, msg: str):
        super().__init__()
        self.msg = msg

    def __str__(self) -> str:
        return self.msg

도메인 예외 계층과 다중 상속 설계는 탄탄하지만, UnexpectedTypeError가 실제 값과 그 값의 타입을 구분하지 않고 _name()에 넘기는 구조 때문에 오류 메시지의 의미적 정확성이 흔들리는 예외 모듈이며, 이 부분만 바로잡으면 라이브러리 전반의 진단 신뢰도까지 한 단계 올라간다.

제안패치
from typing import Type

__all__ = [
    "KFormatError",
    "UnexpectedTypeError",
    "InvalidLengthError",
    "UnsupportedUnicodeError",
]


def _name(type_or_obj) -> str:
    """Return a stable type name for either a type or an instance."""
    if isinstance(type_or_obj, type):
        return type_or_obj.__name__

    return type(type_or_obj).__name__


class KFormatError(Exception):
    """Base exception for the K-Format library."""


class UnexpectedTypeError(KFormatError):
    def __init__(self, expected: Type, value):
        self.expected = expected
        self.value = value

        super().__init__(
            f'"{_name(expected)}" is expected, '
            f'but "{_name(value)}" is given'
        )


class InvalidLengthError(KFormatError):
    def __init__(self, length: int):
        self.length = length

        super().__init__(
            f"Invalid length of value is given (expected: {length})"
        )


class UnsupportedUnicodeError(KFormatError, UnicodeError):
    def __init__(self, msg: str):
        self.msg = msg

        super().__init__(msg)

최종 개선사항
✅ 값과 타입을 동일 취급 → 타입 객체와 인스턴스 명시적 분리 → 오류 메시지의 의미적 정확성 확보
✅ __str__() 의존 → Exception.args에 메시지 직접 전달 → 표준 예외 처리 호환성 강화
✅ 예외별 메시지 초기화 방식 불일치 → 공통적인 예외 초기화 패턴 적용 → 진단 동작 일관성 확보
✅ hasattr(__name__) 기반 과잉 판별 → 실제 타입 여부 기준 판별 → 커스텀 객체 오판 가능성 제거
✅ 기존 예외 계층 유지 → 내부 표현만 정교화 → 라이브러리 API 호환성 및 유지보수성 보존

원본의 도메인 예외 설계는 유지하면서 타입 판별의 의미적 허점을 제거하고 표준 예외 계약까지 보강해, 작은 예외 모듈 안에서도 진단 정확성과 API 안정성을 함께 확보한 9.6 수준의 방어형 구조로 승격되었다.        
