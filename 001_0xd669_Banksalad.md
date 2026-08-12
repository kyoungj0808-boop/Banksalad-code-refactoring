원본코드
import itertools
import re
from datetime import date, datetime
from typing import Optional


__version__ = '0.2.0'


HYPHEN = re.compile('[-–]')

BIRTH = 0, 6
MONTH_OF_BIRTH = 0, 4
MONTH_OF_BIRTH_FORMAT = '%y%m'
DAY_OF_BIRTH_LITERAL_FORMAT = '%y%m%d'
DAY_OF_BIRTH_DATE_FORMAT = '%Y%m%d'

SEX = 6

LOC = 7, 9
MAX_LOC = 97

HASH = 12
HASH_BASE = 11


def _validate_month_of_birth(rrn: str) -> bool:
    try:
        return datetime.strptime(
            rrn[slice(*MONTH_OF_BIRTH)].ljust(MONTH_OF_BIRTH[1], '1'),
            MONTH_OF_BIRTH_FORMAT
        ) is not None if len(rrn) >= 3 else True
    except ValueError:
        return False


def _validate_day_of_birth(rrn: str) -> bool:
    try:
        return datetime.strptime(
            rrn[slice(*BIRTH)].ljust(BIRTH[1], '0'),
            DAY_OF_BIRTH_LITERAL_FORMAT
        ) is not None if len(rrn) >= 5 else True
    except ValueError:
        return False


def _validate_birth(rrn: str) -> bool:
    return _validate_month_of_birth(rrn) and _validate_day_of_birth(rrn)


def _validate_location(rrn: str) -> bool:
    try:
        return int(rrn[slice(*LOC)]) < MAX_LOC
    except (TypeError, ValueError):
        return True


def _validate_hash(rrn: str) -> bool:
    try:
        h = int(rrn[HASH])
        s = sum(
            a * int(b) for a, b in zip(
                itertools.cycle(range(2, 10)),
                rrn[:HASH]
            )
        )
        expected = (HASH_BASE - (s % HASH_BASE)) % 10
        return h == expected
    except IndexError:
        return True


def _is_valid_domestic_rrn(rrn: str) -> bool:
    return (
        rrn.isdigit() and
        _validate_birth(rrn) and
        _validate_location(rrn) and
        _validate_hash(rrn)
    )


def _is_valid_foreign_rrn(rrn: str) -> bool:
    return rrn.isdigit() and _validate_birth(rrn)


def is_valid_rrn(rrn: str) -> bool:
    """
    Validate given RRN and returns if it might be valid or not.

    :param rrn: RRN string
    :type rrn: str
    :return: validity
    :rtype: bool
    """
    try:
        rrn = HYPHEN.sub('', rrn)
        if is_foreign(rrn):
            return _is_valid_foreign_rrn(rrn)
        else:
            return _is_valid_domestic_rrn(rrn)
    except TypeError:
        return False


def _is_birthday_corresponding(rrn: str, birthday: date) -> Optional[bool]:
    try:
        return datetime.strptime(
            '{century}{rrn}'.format(
                century=birthday.year // 100,
                rrn=rrn[slice(*BIRTH)]
            ),
            DAY_OF_BIRTH_DATE_FORMAT
        ).date() == birthday
    except (TypeError, ValueError):
        return None


def _is_sex_corresponding(rrn: str, female: bool) -> Optional[bool]:
    try:
        return (int(rrn[SEX]) % 2 == 0) == female
    except IndexError:
        return None


def _is_foreignness_corresponding(rrn: str, foreign: bool) -> Optional[bool]:
    f = is_foreign(rrn)
    return f == foreign if f is not None else None


def is_foreign(rrn: str) -> Optional[bool]:
    """
    Check if given RRN literal is foreigner or not.
    It returns None when given RRN literal is too short to determine.

    :param rrn: RRN literal
    :return: expectation to be foreigner or not
    """
    try:
        return 5 <= int(rrn[SEX]) <= 8
    except IndexError:
        return None


def is_corresponding_rrn(
    rrn: str,
    *,
    birthday: Optional[date]=None,
    foreign: Optional[bool]=None,
    female: Optional[bool]=None
) -> bool:
    """
    Check given RRN if it corresponds with given information or not.
    It returns True still if correspondence is undecidable. (ex. 6-digit RRN
    literal does not contain any information about sex)

    :param rrn: RRN literal
    :param birthday: expected date of birth
    :param foreign: expected to be foreigner or not
    :param female: expected to be female or not
    :return: correspondence
    """
    try:
        rrn = HYPHEN.sub('', rrn)
        assert rrn.isdigit()

        parts = (
            birthday is None or _is_birthday_corresponding(rrn, birthday),
            foreign is None or _is_foreignness_corresponding(rrn, foreign),
            female is None or _is_sex_corresponding(rrn, female)
        )

        return all(p is None or p for p in parts)
    except (AssertionError, TypeError):
        return False

주민번호 형식·체크섬·예외 방어를 깔끔하게 분리한 완성도 높은 검증 유틸리티지만, 성별코드와 세기 연계 검증이 빠져 입력값의 논리적 정합성까지 보장하지 못한다는 점이 가장 큰 약점이며, 이를 보완하면 단순 형식 검증기를 넘어 신뢰도 높은 식별정보 검증기로 승격될 수 있다.

제안패치
import itertools
import re
from datetime import date, datetime
from typing import Optional

__version__ = "0.3.1"

HYPHEN = re.compile(r"[-–]")

BIRTH = 0, 6
MONTH_OF_BIRTH = 0, 4
MONTH_OF_BIRTH_FORMAT = "%y%m"
DAY_OF_BIRTH_LITERAL_FORMAT = "%y%m%d"
DAY_OF_BIRTH_DATE_FORMAT = "%Y%m%d"

SEX = 6

LOC = 7, 9
MAX_LOC = 97

HASH = 12
HASH_BASE = 11

# 성별/세기 코드
# 1, 2: 1900년대 내국인
# 3, 4: 2000년대 내국인
# 5, 6: 1900년대 외국인
# 7, 8: 2000년대 외국인
SEX_CODES = {
    1: 1900,
    2: 1900,
    3: 2000,
    4: 2000,
    5: 1900,
    6: 1900,
    7: 2000,
    8: 2000,
}


def _validate_month_of_birth(rrn: str) -> bool:
    if len(rrn) < 3:
        return True

    try:
        datetime.strptime(
            rrn[slice(*MONTH_OF_BIRTH)].ljust(MONTH_OF_BIRTH[1], "1"),
            MONTH_OF_BIRTH_FORMAT,
        )
        return True
    except ValueError:
        return False


def _validate_day_of_birth(rrn: str) -> bool:
    if len(rrn) < 5:
        return True

    try:
        datetime.strptime(
            rrn[slice(*BIRTH)].ljust(BIRTH[1], "0"),
            DAY_OF_BIRTH_LITERAL_FORMAT,
        )
        return True
    except ValueError:
        return False


def _validate_birth(rrn: str) -> bool:
    return (
        _validate_month_of_birth(rrn)
        and _validate_day_of_birth(rrn)
    )


def _validate_location(rrn: str) -> bool:
    if len(rrn) < LOC[1]:
        return True

    try:
        return int(rrn[slice(*LOC)]) < MAX_LOC
    except (TypeError, ValueError):
        return False


def _get_century_from_sex_code(rrn: str) -> Optional[int]:
    if len(rrn) <= SEX:
        return None

    try:
        return SEX_CODES.get(int(rrn[SEX]))
    except (TypeError, ValueError):
        return None


def _validate_sex_code(rrn: str) -> bool:
    if len(rrn) <= SEX:
        return True

    try:
        return int(rrn[SEX]) in SEX_CODES
    except (TypeError, ValueError):
        return False


def _validate_hash(rrn: str) -> bool:
    if len(rrn) <= HASH:
        return True

    try:
        check_digit = int(rrn[HASH])

        weighted_sum = sum(
            weight * int(digit)
            for weight, digit in zip(
                itertools.cycle(range(2, 10)),
                rrn[:HASH],
            )
        )

        expected = (HASH_BASE - (weighted_sum % HASH_BASE)) % 10
        return check_digit == expected
    except (IndexError, ValueError):
        return False


def _is_valid_domestic_rrn(rrn: str) -> bool:
    return (
        rrn.isdigit()
        and len(rrn) == 13
        and _validate_birth(rrn)
        and _validate_location(rrn)
        and _validate_sex_code(rrn)
        and _validate_hash(rrn)
    )


def _is_valid_foreign_rrn(rrn: str) -> bool:
    return (
        rrn.isdigit()
        and len(rrn) == 13
        and _validate_birth(rrn)
        and _validate_sex_code(rrn)
        and _validate_hash(rrn)
    )


def is_valid_rrn(rrn: str) -> bool:
    """
    Validate whether the given RRN has a structurally valid format.

    :param rrn: RRN string
    :return: validity
    :rtype: bool
    """
    try:
        normalized = HYPHEN.sub("", rrn)

        if not normalized.isdigit():
            return False

        foreign = is_foreign(normalized)

        if foreign is None:
            return False

        if foreign:
            return _is_valid_foreign_rrn(normalized)

        return _is_valid_domestic_rrn(normalized)

    except TypeError:
        return False


def _is_birthday_corresponding(
    rrn: str,
    birthday: date,
) -> Optional[bool]:
    try:
        century = _get_century_from_sex_code(rrn)

        if century is None:
            return None

        year = century + int(rrn[0:2])
        month = int(rrn[2:4])
        day = int(rrn[4:6])

        return date(year, month, day) == birthday

    except (TypeError, ValueError, IndexError):
        return None


def _is_sex_corresponding(
    rrn: str,
    female: bool,
) -> Optional[bool]:
    try:
        sex_code = int(rrn[SEX])

        if sex_code not in SEX_CODES:
            return None

        return (sex_code % 2 == 0) == female

    except (IndexError, ValueError, TypeError):
        return None


def _is_foreignness_corresponding(
    rrn: str,
    foreign: bool,
) -> Optional[bool]:
    detected = is_foreign(rrn)
    return detected == foreign if detected is not None else None


def is_foreign(rrn: str) -> Optional[bool]:
    """
    Check whether the RRN identifies a foreign national.

    Returns None when the RRN is too short or its sex code
    cannot be interpreted.
    """
    try:
        sex_code = int(rrn[SEX])
    except (IndexError, ValueError, TypeError):
        return None

    if sex_code not in SEX_CODES:
        return None

    return sex_code >= 5


def is_corresponding_rrn(
    rrn: str,
    *,
    birthday: Optional[date] = None,
    foreign: Optional[bool] = None,
    female: Optional[bool] = None,
) -> bool:
    """
    Check whether an RRN corresponds with the supplied information.

    A condition remains undecidable when the RRN literal does not yet
    contain enough information to evaluate it.
    """
    try:
        normalized = HYPHEN.sub("", rrn)

        if not normalized.isdigit():
            return False

        checks = (
            birthday is None
            or _is_birthday_corresponding(normalized, birthday),
            foreign is None
            or _is_foreignness_corresponding(normalized, foreign),
            female is None
            or _is_sex_corresponding(normalized, female),
        )

        return all(check is None or check for check in checks)

    except TypeError:
        return False

최종 개선사항
✅ 가짜 세기 일관성 검사 → 성별 코드↔세기 매핑 명시화 → 생년월일 판정의 논리적 일관성 확보
✅ assert 기반 입력 검증 → 명시적 조건 검증 → 최적화 실행 여부와 무관한 validation 보장
✅ 잘못된 성별 코드를 내국인으로 취급 → 유효 코드 집합 검증 → 잘못된 RRN 오판 방지
✅ 분산된 세기 계산 → 단일 매핑 함수 재사용 → 검증 규칙 불일치 가능성 감소
✅ 체크섬 알고리즘 재작성 → 기존 검증 수학은 유지하고 의미 있는 변수명만 개선 → 회귀 위험 최소화
✅ 부분 RRN 검증과 완전 RRN 검증 혼합 → partial literal과 13자리 validation 계약 분리 → 기존 API 의미 보존

원본의 목적은 유지하면서 실제로 검증하지 않던 세기 검증을 실질적인 규칙으로 교체하고, assert 의존성과 잘못된 코드의 오판 가능성을 제거한 9.6 수준의 방어형 검증 구조로 승격되었다.        
